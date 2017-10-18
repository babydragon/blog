# logback导致CPU利用率彪高排查

## 现象
最近遇到一例进程CPU使用率很高的情况，经过初步排查，CPU都消耗在用户态，系统和IO占用非常低；使用```jstack```命令打印线程栈信息，发现有许多业务线程都在执行（或阻塞）和logback相关的代码。有部分线程在执行```java.lang.Throwable.getStackTraceElement```这个native方法。初步排查和logback日志记录有关，暂时关闭日志记录之后，进程CPU占用率立马下降。

## 排查
通过初步排查，比较怀疑下面这个栈调用导致了CPU占用率的上升：

```
java.lang.Thread.State: RUNNABLE
        at java.lang.Throwable.getStackTraceElement(Native Method)
        at java.lang.Throwable.getOurStackTrace(Throwable.java:827)
        - locked <0x000000076f6dbe88> (a java.lang.Throwable)
        at java.lang.Throwable.getStackTrace(Throwable.java:816)
        at ch.qos.logback.classic.spi.CallerData.extract(CallerData.java:60)
        at ch.qos.logback.classic.spi.LoggingEvent.getCallerData(LoggingEvent.java:258)
        at ch.qos.logback.classic.pattern.FileOfCallerConverter.convert(FileOfCallerConverter.java:22)
        at ch.qos.logback.classic.pattern.FileOfCallerConverter.convert(FileOfCallerConverter.java:19)
        at ch.qos.logback.core.pattern.FormattingConverter.write(FormattingConverter.java:36)
        at ch.qos.logback.core.pattern.PatternLayoutBase.writeLoopOnConverters(PatternLayoutBase.java:115)
        at ch.qos.logback.classic.PatternLayout.doLayout(PatternLayout.java:141)
        at ch.qos.logback.classic.PatternLayout.doLayout(PatternLayout.java:39)
        at ch.qos.logback.core.encoder.LayoutWrappingEncoder.encode(LayoutWrappingEncoder.java:115)
        at ch.qos.logback.core.OutputStreamAppender.subAppend(OutputStreamAppender.java:230)
        at ch.qos.logback.core.rolling.RollingFileAppender.subAppend(RollingFileAppender.java:235)
        at ch.qos.logback.core.OutputStreamAppender.append(OutputStreamAppender.java:102)
        at ch.qos.logback.core.UnsynchronizedAppenderBase.doAppend(UnsynchronizedAppenderBase.java:84)
        at ch.qos.logback.core.spi.AppenderAttachableImpl.appendLoopOnAppenders(AppenderAttachableImpl.java:51)
        at ch.qos.logback.classic.Logger.appendLoopOnAppenders(Logger.java:270)
        at ch.qos.logback.classic.Logger.callAppenders(Logger.java:257)
        at ch.qos.logback.classic.Logger.buildLoggingEventAndAppend(Logger.java:421)
        at ch.qos.logback.classic.Logger.filterAndLog_0_Or3Plus(Logger.java:383)
        at ch.qos.logback.classic.Logger.warn(Logger.java:704)
```

因此试图通过一个简单的springboot应用来模拟。这里栈在执行获取栈元素的操作，所以首先考虑栈比较深的场景。创建下面这个类，通过循环调用来模拟比较深的调用栈：

```java
public class DeepStackTask implements Runnable {

    private Log log = LogFactory.getLog(DeepStackTask.class);

    private int count;
    private Random r;

    public DeepStackTask(int count) {
        if (count <= 0) {
            count = 1;
        }
        this.count = count;
        r = new Random();
    }

    @Override
    public void run() {
        try {
            noop();
        } catch (Exception e) {
            // ignore
        }
    }

    private void noop() throws InterruptedException {
        log.warn("no op");
        if (count > 0) {
            count--;
            int count = 1000;
            double sum = 0;
            for (int i = 0; i < 1000; i++) {
                sum += r.nextDouble();
            }
            System.out.println(sum/count);
            noop();
        }
    }
}
```

该类实现了```Runnable```接口，因此可以通过一个简单的线程池不停创建该对象即可。

```java
@Service
public class LogTestService {

    private ExecutorService threadPool;

    @PostConstruct
    public void init() {
        threadPool = Executors.newFixedThreadPool(5);
    }

    @Scheduled(fixedRate = 10)
    public void autoSubmitEvent() {
        threadPool.submit(new DeepStackTask(20));
    }

}
```

这里可以通过线程池大小、任务提交间隔和栈深度来模拟高压力环境。

最终调整到30个线程、栈深度100,使用青龙插装之后，CPU占用率有显著上升，但是在```DeepStackTask```类中直接使用logback记录日志，CPU占用率上升不明显。

然后尝试debug应用，最终发现如果断点打在logback的```CallerData```类上，直接使用logback的logger一直没有运行到过。查看了前面贴的栈对应的logback源代码，发现此时logback试图从当前线程创建的```Throwable```对象获取当前线程的整个栈信息，这个调用来自于```FileOfCallerConverter```这个对象。从类名猜测（这个类源码中没有注释），这是在转换日志格式中的文件名。

通过这个线索，在demo应用中修改logback日志格式，加上打印文件名，果然瞬间CPU利用率飚升。

## 解决
知道了问题根源：

1. logback记录文件名的时候，每次都要获取当前栈信息
2. 获取栈信息性能不怎么样

解决方法很简单，在高频率日志记录的地方，不要打印文件名和行号！

## 插曲
青龙日志记录模块，为了能够动态修改日志级别（？）自己包装了日志记录相关的类。因此在实际调用过程中，会先调用青龙内部的日志记录类，然后再代理到实际的日志记录组件。这样在logback中会遇到问题，记录的文件名和行号，永远是青龙自己包装过的类。

排查上述问题（```FileOfCallerConverter```类）的过程中，发现logback在获取文件名（和行号）的时候，大致流程是（具体代码逻辑见```ch.qos.logback.classic.spi.CallerData#isInFrameworkSpace```方法）：

1. 获取当前线程栈
2. 从栈顶开始向下遍历调用者
3. 当调用者不是logback边界类的时候，返回剩余栈
4. 从剩余栈中获取顶端调用者，作为日志记录调用者
5. 获取日志记录调用者文件名（行号）

这个流程里面，由于自己包装的日志记录类不再logback的边界类中，因此会被logback认定为日志记录发起的类，导致最终的记录信息中调用文件名和行号都来自自己封装的日志记录类，而不是实际业务代码。

logback在判断的时候留了一个```frameworkPackageList```的列表作为扩展，并且默认在其中设置了groovy。因此我们也可以在自己的LogFactory中，增加自己的日志记录框架包名，让logback从栈调用链中排除。当然，不要在高并发场景下记录文件名！！
