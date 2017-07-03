# Java 8 Stream API

## 优势

Java 8中的Stream概念类似于实时数据处理框架的Stream概念，将数据流抽象成一个对象。Java的Stream通常由集合转换而来，但Stream并不等同于集合，Stream可以看作是一个数据生成器，能够生成无穷无尽的数据。另外，集合中的数据都直接保存在内存中，而Stream中的数据，是在数据使用时实时计算出来的。Stream提供了[generate](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#generate-java.util.function.Supplier-)方法，通过接受一个[Supplier](https://docs.oracle.com/javase/8/docs/api/java/util/function/Supplier.html)来每次创建一个对象。具体创建方式下面会有示例。

Stream本身是数据源的抽象，其最重要的功能是处理这些数据。此时Stream类似于迭代器，但Stream提供了转换、聚合等各种操作，配合lambda表达式，可以在不使用循环的情况下处理数据。不过迭代器由调用者负责，Stream的迭代隐含在其中的各种操作中（如map等）。

相比于直接使用循环，Stream API更加简洁，并且可以容易的转成并行处理，充分利用CPU。

## 使用

首先是创建Stream，Stream的创建有两种方式，一种是通过集合类的[stream方法](https://docs.oracle.com/javase/8/docs/api/java/util/Collection.html#stream--)直接生成，一种是通过Stream类的[generate方法](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#generate-java.util.function.Supplier-)创建。

一般情况下，最常用的应该是前者，例如，将一个List对象转换成Stream只需要：
```java
List<Integer> intList = Arrays.asList(1, 2, 3, 4, 5);
Stream<Integer> stream = intList.stream();
```
即可创建一个Stream对象，后续可以通过Stream对象提供的方法操作其中包含的数据。

如果要使用自定义的数据生成器，需要实现[Supplier接口](https://docs.oracle.com/javase/8/docs/api/java/util/function/Supplier.html)，当然，也可以直接通过lambda表达式进行创建。

```java
Random random = new Random();
Stream<Integer> stream = Stream.generate(random::nextInt);
```
这个例子通过Java提供的随机数对象，创建一个随机数Stream。通常情况下，需要通过generate来创建的Stream，都用来表示无穷尽的流。事实上，Java已经内置了多个这样的流，例如IntStream、LongStream、DoubleStream等数列流，Random对象也提供了ints、longs、doubles等方法提供随机数列流。因此这个示例也可以这样写：
```java
Random random = new Random();
IntStream intStream = random.ints();
```

创建完成之后，就可以通过Stream提供的方法对流进行操作，常用的操作有filter、map、collect等。

#### filter
filter方法主要用于过滤Stream，以筛选出有意义的数据。例如，需要筛选出前面intList转换的Stream中所有的偶数：
```java
List<Integer> intList = Arrays.asList(1, 2, 3, 4, 5);
intList.stream().filter((n) -> n % 2 == 0).forEach(System.out::println);
```
如示例所示，fileter方法接受一个Predicate对象（或者是一个返回boolean类型的lambda表达式），Stream会对其中的所有数据应用这个过滤方法，返回过滤后的新Stream对象。示例中直接将新生成Stream的每个元素输出，结果为：
```
2
4
```

**注意：** Stream有流的概念，因此一个Stream对象如果已经操作过了，就无法再次进行操作。例如：
```java
List<Integer> intList = Arrays.asList(1, 2, 3, 4, 5);
Stream<Integer> intStream = intList.stream();
Stream<Integer> evenStream = intStream.filter((n) -> n % 2 == 0);
Stream<Integer> oddStream = intStream.filter((n) -> n % 2 != 0);
```
上述代码试图从一个Stream对象中，分别过滤出奇数和偶数，生成两个不同的Stream，但是这样的代码实际运行时会抛出```java.lang.IllegalStateException```提示：stream has already been operated upon or closed。

#### map/flatMap
对于Stream的操作，最多的应该就是map操作了，map操作实际上是将Stream中的数据，从一种数据类型转换成另一种数据类型。例如，前面的整型Stream转换成字符串类型，可以：
```java
List<Integer> intList = Arrays.asList(1, 2, 3, 4, 5);
Stream<String> stringStream = intList.stream().map(String::valueOf);
```
通过对整型Stream中的每个元素执行```String::valueOf```方法之后，整型Stream转换成了字符串Stream。

和map类似的还有一个方法是flatMap。顾名思义，该方法是将源Stream中的每个元素，转换成一个Stream，然后将这些Stream重新整合成一个Stream。既map方法是一对一映射转换，flatMap方法是一对多映射转换。

```java
public static void flatMap() {
    List<Integer> intList = Arrays.asList(1, 2, 3, 4, 5);
    Stream<Integer> flatStream = intList.stream().flatMap(i -> {
        List<Integer> result = new ArrayList<>();
        if (i > 2 && i % 2 == 0) {
            result.add(2);
            result.add(i / 2);
        } else {
            result.add(i);
        }
        return result.stream();
    });

    flatStream.forEach(System.out::println);
}
```

上述示例将大于2的偶数都除以2,然后放到新的Stream中，输出结果如下：

```
1
2
3
2
2
5
```
由此可以看出，相比于map，flatMap中传入的lambda表达式返回值是一个Stream，但是映射之后多个Stream会被合成为一个Stream。map用于1对1的转换，flatMap用于1对多的转换。

#### collect
对于Stream的操作，通常还是需要在处理完成之后转换成普通的类型。这里常见的有两种，一种是重新转换成Java内置的集合类型，既将一个集合经过一系列的过滤、转换之后，生成一个新的集合的过程；另一种是类似于SQL中的聚合函数，将集合中的数据聚合成一个结果。

第一种情况：
```java
List<Integer> intList = Arrays.asList(1, 2, 3, 4, 5);
Stream<Integer> intStream = intList.stream();
Stream<Integer> evenStream = intStream.filter((n) -> n % 2 == 0);
List<Integer> evenList = evenStream.collect(Collectors.toList());
```
其中```Collectors```中提供了大量的方法，能够支持将Stream的数据转换成普通对象，常用的有```toSet```、```toList```、```toMap```等转换成集合类的方法，另外```joining```也挺好用，可以将Stream中的字符串，整合成一个字符串。由于Stream中的数据和集合类比较类似，因此```toSet```、```toList```等方法不需要任何参数即可转换，如果要转成Map对象，需要传入两个lambda表达式，以计算出Map的key和value。

比如下面这个示例：
```java
Stream<User> userStream = IntStream.range(0, 10).mapToObj(i -> {
    User u = new User();
    u.setId(i);
    u.setName("u_" + i);

    return u;
});

Map<Integer, User> result = userStream.collect(Collectors.toMap(User::getId, Function.identity()));
```
这个示例首先创建了个```User```对象的Stream（这里通过IntStream来进行转换，个人认为比通过for循环更加清晰），然后将其转换成Map对象，其中key是User对象中的id字段，value是User对象（```Function.identity```方法类似于v->v的lambda表达式，既直接返回传入的参数）。如果不使用Stream，通常我们会创建一个Map，然后通过循环遍历```List<User>```对象，创建最终的Map对象。

除此之外，Stream还提供了基本的聚合函数，例如```min```、```max```、```count```等平时使用不多，这里不提供示例。

## parallelStream
如果从Java集合类转换成Stream，除了```stream()```方法之外，还有一个```parallelStream```方法。顾名思义，该方法转换成的Stream，其中的数据操作是可以并发进行的。因此，对于Stream的数据操作，无论是否使用parallelStream，都应该认为是可以并行计算的，**千万不要依赖Stream中数据的顺序**。

parallelStream中的大部分方法（数据可以独立计算，相互直接没有干扰）都是并行执行的，并行处理的任务，会被提交到ForkJoinPool中进行执行。**这里需要特别注意：如果不在一个自定义的ForkJoinPool中执行，任务实际会被提交到系统默认创建的公共ForkJoinPool中。**
因此，这里强烈建议，如果需要使用parallelStream，并行操作时，建议自己创建一个独立的ForkJoinPool，以避免对系统公共ForkJoinPool产生影响，同时能够控制任务执行的总线程数量。

```java
IntStream.range(1, 5).parallel().forEach(i -> {
    System.out.println(Thread.currentThread().getName());
});

System.out.println("======");

ForkJoinPool pool = new ForkJoinPool(4);
pool.submit(() -> {
    IntStream.range(1, 5).parallel().forEach(i -> {
        System.out.println(Thread.currentThread().getName());
    });
});
```
代码执行之后的输出：
```
main
ForkJoinPool.commonPool-worker-3
ForkJoinPool.commonPool-worker-1
main
======
ForkJoinPool-1-worker-1
ForkJoinPool-1-worker-1
ForkJoinPool-1-worker-2
ForkJoinPool-1-worker-2
```
从结果可以看见，如果parallelStream的操作不在限定的ForkJoinPool中运行，部分任务在主线程中执行，部分任务在公共ForkJoinPool中执行。

另外，需要注意的是，不是所有操作都支持并行执行的，如果不支持，仍然会以串行的方式执行。
