# Java Fork/Join框架

Fork/Join框架是Java 7引入的并行执行框架，它的基本原理就是将一个大任务，切分成能够单独执行的小任务，再将每个小任务的结果汇总起来的流程。通常情况下，能够使用该框架的任务，都是能够被切分，并且切分后的子任务互不依赖，可以单独执行。例如一个进行数列加法的任务：1+2+3+...+n，我们可以将其切分成最小2个数字的加法，然后父任务将子任务的结果再累加起来，最终得出结果。通过将这些子任务放到独立线程中运行，最终能够降低整个大任务的执行时间。

## Fork/Join框架使用
整个Fork/Join框架主要有两部分组成：```ForkJoinTask```和```ForkJoinPool```。前者约束了Fork/Join框架本身，提供了fork/join等方法，需要使用Fork/Join框架的任务，必须继承```ForkJoinTask```或其子类；后者提供了Fork/Join框架的执行环境，负责调度运行Fork/Join任务及其子任务。

按照```ForkJoinTask```类的[注释](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinTask.html)，通常情况下无需直接继承```ForkJoinTask```类，建议按照任务类型划分，继承对应的三个子类：

* [RecursiveAction](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/RecursiveAction.html): 用于无返回值的任务，事实上该类型任务是```ForkJoinTask<Void>```的特化，获取result的时候（getRawResult方法）永远返回null。
* [RecursiveTask](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/RecursiveTask.html)：用于有返回值的任务，该任务将compute方法的返回值作为任务的返回值。
* [CountedCompleter](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CountedCompleter.html)：确保所有子任务都必须完成。

例如一个累加任务，由于需要将每个子任务结果累加，我们可以使用```RecursiveTask```，将大的累加任务细分，最终将每个子任务的结果重新求和。

```java
public class ForkJoinCountTask extends RecursiveTask<Integer> {

    private static final int THRESHOLD = 10;

    private int start;
    private int end;

    public ForkJoinCountTask(int start, int end) {
        this.start = start;
        this.end = end;
    }

    @Override
    protected Integer compute() {
        int sum = 0;
        boolean canCompute = (end - start) <= THRESHOLD;
        if (canCompute) {
            System.out.println(String.format("start to compute from %d to %d", start, end));
            for (int i = start; i <= end; ++i) {
                sum += i;
            }
        } else {
            int middle = (start + end) / 2;
            ForkJoinCountTask left = new ForkJoinCountTask(start, middle);
            ForkJoinCountTask right = new ForkJoinCountTask(middle + 1, end);

            left.fork();
            right.fork();

            Integer leftResult = left.join();
            Integer rightResult = right.join();

            sum = leftResult + rightResult;
        }
        return sum;
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ForkJoinPool pool = new ForkJoinPool();
        ForkJoinCountTask task = new ForkJoinCountTask(1, 100);
        ForkJoinTask<Integer> result = pool.submit(task);
        System.out.println(result.get());
    }
}
```

类似于递归函数，上述任务先检查当前输入是否能够达到最小任务阈值，如果已经达到，则直接计算；如果达不到，新建两个任务分别计算两部分，等待子任务完成后，将子任务结果求和之后返回。和普通递归不同，子任务会在ForkJoinPool中并行执行。执行结果如下：

```
start to compute from 14 to 19
start to compute from 51 to 57
start to compute from 58 to 63
start to compute from 64 to 69
start to compute from 70 to 75
start to compute from 76 to 82
start to compute from 1 to 7
start to compute from 8 to 13
start to compute from 83 to 88
start to compute from 89 to 94
start to compute from 95 to 100
start to compute from 39 to 44
start to compute from 45 to 50
start to compute from 33 to 38
start to compute from 20 to 25
start to compute from 26 to 32
5050
```

## 关于common pool
前文关于Java 8 Stream API中提到了，parallelStream执行并行操作是，默认会使用common pool来执行。这个逻辑是在```ForkJoinTask```中实现的。
```java
public final ForkJoinTask<V> fork() {
    Thread t;
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
        ((ForkJoinWorkerThread)t).workQueue.push(this);
    else
        ForkJoinPool.common.externalPush(this);
    return this;
}
```
在ForkJoinTask中调用fork方法fork子任务时，```ForkJoinTask```会自己判断执行线程是否在```ForkJoinPool```中运行，如果是则使用这个pool；如果不是，则使用common pool来执行。之前示例中已经看出结果了。

由于common pool是全局静态的执行池，common pool的初始化在```ForkJoinTask```的静态块中完成，具体执行时会调用```ForkJoinPool```的makeCommonPool方法：
```java
private static ForkJoinPool makeCommonPool() {
    int parallelism = -1;
    ForkJoinWorkerThreadFactory factory = null;
    UncaughtExceptionHandler handler = null;
    try {  // ignore exceptions in accessing/parsing properties
        String pp = System.getProperty
            ("java.util.concurrent.ForkJoinPool.common.parallelism");
        String fp = System.getProperty
            ("java.util.concurrent.ForkJoinPool.common.threadFactory");
        String hp = System.getProperty
            ("java.util.concurrent.ForkJoinPool.common.exceptionHandler");
        if (pp != null)
            parallelism = Integer.parseInt(pp);
        if (fp != null)
            factory = ((ForkJoinWorkerThreadFactory)ClassLoader.
                       getSystemClassLoader().loadClass(fp).newInstance());
        if (hp != null)
            handler = ((UncaughtExceptionHandler)ClassLoader.
                       getSystemClassLoader().loadClass(hp).newInstance());
    } catch (Exception ignore) {
    }
    if (factory == null) {
        if (System.getSecurityManager() == null)
            factory = defaultForkJoinWorkerThreadFactory;
        else // use security-managed default
            factory = new InnocuousForkJoinWorkerThreadFactory();
    }
    if (parallelism < 0 && // default 1 less than #cores
        (parallelism = Runtime.getRuntime().availableProcessors() - 1) <= 0)
        parallelism = 1;
    if (parallelism > MAX_CAP)
        parallelism = MAX_CAP;
    return new ForkJoinPool(parallelism, factory, handler, LIFO_QUEUE,
                            "ForkJoinPool.commonPool-worker-");
}
```

从上述代码可以看见，如果没有配置，默认的common pool线程数是机器CPU数-1（如果小于1，则初始化为1）。这个值可以通过```java.util.concurrent.ForkJoinPool.common.parallelism```变量调整。所以之前提到了，如果ParallelStream有并行操作执行，强烈建议专门定义一个```ForkJoinPool```。
