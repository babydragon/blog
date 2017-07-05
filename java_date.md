# java 8的时间日期相关对象

java 8引入了```java.time```包，其中包含新的时间日期相关的类。相比于```java.util.Date```等原始的时间日期类，新的类从时间相关领域模型出发，操作起来更加语义化。

## 本地时间类
新API中最常用的应该就是本地时间，即Local开头的一些类，比如```LocalDate```、```LocalDateTime```等。这些类都有对应的```of```静态方法来初始化。当然，如果要获取当前时间，可以使用```now```方法。

初始化对象之后，可以通过相关getter方法来获取时间的某个属性，例如年、月、日等，也可以通过```plus```相关方法来计算时间。

```java
LocalDateTime dateTime = LocalDateTime.now();
System.out.println(dateTime.toString());
dateTime = dateTime.plusHours(10);
System.out.println(dateTime.toString());
```

上面是一个简单的示例，```LocalDateTime```有对应的toString方法，可以直接将时间日期格式化输出，默认会按照ISO-8601格式格式化。如果需要按照自己的需求格式化输出，可以使用```format```方法，定制自己的格式化格式。

```java
System.out.println(dateTime.format(DateTimeFormatter.ofPattern("yyyy-MM-dd hh:mm:ss")));
```
例如按照最常见的格式，可以使用```DateTimeFormatter.ofPattern```方法直接初始化一个格式化类。

修改后的输出为：

> 2017-07-04T17:14:18.764

> 2017-07-05 03:14:18

## 时间日期的测试
前文介绍了可以通过```LocalDateTime.now```方法获取当前时间。但是如果需要进行单元测试，依赖机器当前时间的代码会非常困难。事实上，```now```方法还支持传入一个```Clock```对象，以定制当前时钟。

```java
Clock clock = Clock.systemDefaultZone();
LocalDateTime dateTime = LocalDateTime.now(clock);
System.out.println(dateTime.toString());
```

如这个代码片段，看上去和之前的输出没有什么区别，但是如果封装类中需要获取当前时间来进行计算（例如根据时间的签名等），建议能够在构造函数，或者提供setter类暴露clock的写入接口。这样，在正常使用的时候，可以将Clock初始化成系统时钟，当单元测试的时候，初始化成固定时钟，以方便断言：

```java
LocalDateTime fixTime = LocalDateTime.of(2017, 1, 1, 0, 0, 0);
clock = Clock.fixed(fixTime.toInstant(ZoneOffset.UTC), ZoneId.of("UTC"));
dateTime = LocalDateTime.now(clock);
System.out.println(dateTime.toString());
```

这里首先获取了2017年1月1日对应的时刻，然后用这个时刻初始化一个固定时钟（使用```Clock.fixed```方法）。这样，虽然还是通过```LocalDateTime.now```来获取当前时间，实际只会返回之前设置好的固定时刻。通过完整的代码来实验下：

```java
Clock clock = Clock.systemDefaultZone();
LocalDateTime dateTime = LocalDateTime.now(clock);
System.out.println(dateTime.toString());

Thread.sleep(1000);
dateTime = LocalDateTime.now(clock);
System.out.println(dateTime.toString());

System.out.println("======");

LocalDateTime fixTime = LocalDateTime.of(2017, 1, 1, 0, 0, 0);
clock = Clock.fixed(fixTime.toInstant(ZoneOffset.UTC), ZoneId.of("UTC"));
dateTime = LocalDateTime.now(clock);
System.out.println(dateTime.toString());

Thread.sleep(1000);
dateTime = LocalDateTime.now(clock);
System.out.println(dateTime.toString());
```

代码输入如下：
> 2017-07-04T17:45:29.135
>
> 2017-07-04T17:45:30.135
>
> ======
>
> 2017-01-01T00:00
>
> 2017-01-01T00:00

从结果可以看见使用了固定时钟之后，每次输出时间都是相同的，这样就能够进行单元测试断言了。

## 时区
前面示例在从```LocalDateTime```转换成```Instant```对象的时候，已经涉及到时区。创建一个```ZoneId```对象，可以通过```of```方法，传入时区名称。例如北京时间可以：

```java
ZoneId.of("Asia/Shanghai");
```

这样逻辑比较简单，老的时间API中，```Date```对象本身是不包含时区的，而是格式化的时候，通过```SimpleDateFormat```来设置时区并格式化。另外，新的时间日期API中，还引入了```ZonedDateTime```对象，可以在创建时设置时区，这样可以不依赖系统时区来进行计算。

## Duration和Period

这两个类语义上比较明确，和之前提到的时间点相关类型（LocalDateTime、Instant等）不同，这两个类表示的是时间间隔。其中Period精度以日为单位，Duration的精度比较高，精确到纳秒。因此，后面如果需要表示时间间隔，不再需要使用long类型来表示时间间隔。
