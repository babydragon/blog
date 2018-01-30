# 深入参合logback

项目中通过slf4j桥接到logback进行日志记录。出于可靠性考虑，需要监控日志文件大小，防止因为日志文件过大导致系统不可用。

### 从slf4j到logback
平时我们在获取日志的logger对象的时候，都会通过`LoggerFactory`对象，但是现在我们要获取的是`ILoggerFactory`对象：

```java
ILoggerFactory iLoggerFactory = LoggerFactory.getILoggerFactory();
```

如果当前slf4j是桥接到logback的，这个`iLoggerFactory`的最终实现会是`ch.qos.logback.classic.LoggerContext`对象。这样我们就获取到了logback的上下文信息。

### 获取logback appender
通过logback的`LoggerContext`对象，我们可以先获取到logback所有的logger，进而获取到这些logger对应的所有appender。

```java
LoggerContext lc = (LoggerContext)iLoggerFactory;
List<Logger> loggers = lc.getLoggerList();
FileAppender<ILoggingEvent> fileAppender = null;
for (Logger logger : loggers) {
    Iterator<Appender<ILoggingEvent>> appenderIterator = logger.iteratorForAppenders();
    while (appenderIterator.hasNext()) {
        Appender<ILoggingEvent> appender = appenderIterator.next();
        if (appender instanceof FileAppender) {
            fileAppender = (FileAppender<ILoggingEvent>) appender;
            break;
        }
    }

    if (fileAppender != null) {
        break;
    }
}
```

由于我们的目标是获取日志文件，因此我们只关注`FileAppender`对象。

### 日志文件
已经获取到了`FileAppender`之后，就可以获取这个appender写入的文件路径。

```java
String fileName = fileAppender.getFile();
```

事实上，大部分时候我们不会直接使用`FileAppender`，而是会使用`RollingFileAppender`。不论是哪个对象，我们都只能获取到当前正在使用的日志文件，无法再获取到之前回卷的日志文件。

### 关于`RollingFileAppender`
获取日志文件之外，还有一个需求是能够删除日志文件。最初的设想是能够手动触发`RollingFileAppender`的回卷操作，这样可以直接把所有带有特定格式的日志文件直接删除。`RollingFileAppender`类中实际也包含了`rollover()`方法。但是直接调用的时候，可能抛出异常。

查看异常栈之后发现，回卷之前，`RollingPolicy`需要计算当前文件回卷之后的文件名，如果回卷文件名格式中包含了index（%i）的时候，会因为第一次没有初始化而导致失败。因此最好不要直接调用该方法强制回卷文件。

### logback的单元测试（强制同步写入）
作为一个日志框架，IO性能非常重要，异步写入成为了标配。但是在单元测试中，我们没有办法等待缓冲区写入硬盘，极有可能导致对日志文件的断言失败。

为了能够正常进行单元测试，需要实现一个同步写入的appender，它会创建一个每次写日志都执行sync操作的output stream。

```java
public class ImmediateFileAppender<E> extends RollingFileAppender<E> {

    @Override
    public void openFile(String file_name) throws IOException {
        synchronized (lock) {
            File file = new File(file_name);
            boolean result = FileUtil.createMissingParentDirectories(file);
            if (!result) {
                addError("Failed to create parent directories for [" + file.getAbsolutePath() + "]");
            }

            ImmediateResilientFileOutputStream resilientFos = new ImmediateResilientFileOutputStream(file, append);
            setOutputStream(resilientFos);
        }
    }

    @Override
    protected void writeOut(E event) throws IOException {
        super.writeOut(event);

    }
}
```

这个类主要作用是重写打开文件，将文件输出流设置成自定义的`ImmediateResilientFileOutputStream`。下面是这个类的实现：

```java
public class ImmediateResilientFileOutputStream extends OutputStream {
    protected FileOutputStream os;

    public ImmediateResilientFileOutputStream(File file, boolean append) throws FileNotFoundException {
        os = new FileOutputStream(file, append);
    }
    @Override
    public void write(int b) throws IOException {
        os.write(b);
    }

    @Override
    public void flush() throws IOException {
        if (os != null) {
            try {
                os.flush();
                os.getFD().sync(); // 这里强制进行sync操作
            } catch (IOException e) {
                // ignore
            }
        }
    }
}
```

和默认实现的主要差别，就是在`FileOutputStream`刷新之后，再执行一次`sync`操作，确保缓冲区同步到磁盘中。

最后，测试用的logback-test.xml中，需要将以前的RollingFileAppender改成这个类。并且确保immediateFlush设置为true。

注意：因为每次写入都强制同步，千万不要用到生产环境中～
