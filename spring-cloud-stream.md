# spring stream使用

## spring stream基本用法

### 绑定管道
spring cloud的stream模块，需要使用```@EnableBinding```注解触发，该注解参数是绑定的输入输出管道的定义。通常情况下，可以使用```Source```、```Sink```等类来初始化对应的管道。这两个接口，实际上是通过```@Input```和```@Output```两个注解，来定义输入和输出管道。

```Source```类定义如下：

```java
public interface Source {

  String OUTPUT = "output";

  @Output(Source.OUTPUT)
  MessageChannel output();

}
```
因此，如果通过```@EnableBinding(Source.class)```来绑定输出管道，实际上是定义了一个名为“output”的管道。以此类推，绑定Sink会初始化名为“input”的输入管道。如果需要自定义管道和管道名，可以自定义一个接口，通过```@Output```和```@Input```注解来自定义输入输出管道，两个注解的参数是管道的名称。

### 使用管道
通过```@EnableBinding```注解初始化的输入管道，可以通过```@ServiceActivator```或者```@StreamListener```注解来使用。这两个注解都可以直接使用在方法上，被注解方法的参数是消息体（payload）或者直接使用Message接口。```@ServiceActivator```注解可以同时指定输入管道和输出管道，此时注解方法的入参为输入管道的消息，返回值会被发送到输出管道。二者最主要的区别是```@StreamListener```注解对消息体的类型转换支持比较全面，但是没有尝试过，可以参照[文档](http://docs.spring.io/spring-cloud-stream/docs/current/reference/htmlsingle/#_using_streamlistener_for_automatic_content_type_handling)来尝试。使用```@ServiceActivator```的大致方式为：

```java
@ServiceActivator(inputChannel= Sink.INPUT)
public void receive(Message msg) {
  // 处理消息
}
```

输出管道可以直接注入```MessageChannel```类型，bean名称为注解中初始化的输出管道名称，例如：

```java
@Autowired
@Qualifier(Source.OUTPUT)
private MessageChannel sopsTaskChannel;
```
这样就可以直接在代码中使用```MessageChannel```的send方法直接发送消息了。

### 管道配置
除了使用```@EnableBinding```绑定初始化管道之后，还需要佐以配置。管道配置很多，均以```spring.cloud.stream.bindings```开头，加上管道的名字，最常用的配置有两个：

* spring.cloud.stream.bindings.${channelName}.destination：管道的目的地，对于消息队列来说，就是消息队列的名称；
* spring.cloud.stream.bindings.${channelName}.group：管道分组，这个概念主要来自于kafka，对于消费者来说，一个分组的消费者共享消息队列；

**注意：**${channelName}必须和```@EnableBinding```注解初始化的管道名称相同，如果不同，配置将会失效。如果destination没有配置，默认的destination将会和管道名称相同。例如```@EnableBinding(Source.class)```绑定的输出管道，其名称为“output”，那么要设置该管道的目的地，需要增加配置：```spring.cloud.stream.bindings.output.destination```。

## 集成metaq
有了spring cloud stream的抽象之后，引入metaq就比较方便了。首先在pom中增加依赖：

```xml
<!-- metaq -->
 <dependency>
     <groupId>com.alibaba.cloud</groupId>
     <artifactId>spring-cloud-stream-starter-metaq</artifactId>
     <version>1.0.0-SNAPSHOT</version>
 </dependency>
```

然后按照前面的方式，通过```@EnableBinding```注解初始化需要使用的队列，如果需要发送消息（生产者），可以绑定```Output```，如果需要接受消息（消费者），可以绑定```Sink```。当然也可以模仿这两个接口自定义管道名称。注意管道名字和配置的一致性。

管道初始化之后，配置加上对应的destination即可。destination就是metaq的topic。这样就可以注入管道或者初始化消费者。具体的示例可以参照[项目示例](http://gitlab.alibaba-inc.com/spring-boot-incubator/spring-cloud-stream-binder-metaq/tree/master)。

## 坑

### 管道命名和实际队列
管道名称和实际队列名称其实没什么关系，管道名称实际由```@Input```和```@Output```两个注解指定，实际队列通过配置文件中的destination指定。因此需要特别注意管道配置的key（spring.cloud.stream.bindings.${channelName}.destination），这里的channelName**必须**和管道名称相同。

关于管道命名还有一个需要注意的地方。spring cloud stream的抽象中，管道不一定和后端消息队列关联。应用内部也可以定义输入和输出管道，来实现应用内部的事件系统。因此，如果在一个应用中定义了名称相同的输入和输出管道，实际运行中这两个管道会直接对接，即发送管道send方法调用之后，会直接调用消费者方法。此时，即使通过配置文件定义了管道的destination，实际也不会发送任何消息。

### 关于group
前面配置还提到了spring.cloud.stream.bindings.${channelName}.group配置，group是消费者分组，group相同的消费者会同时消费一个队列。对于metaq来说，group就是console中设置的```consumerId```。特别注意对于日常和预发环境，consumerId不是必须的，所以可以不添加订阅者，直接消费队列中的消息，但是正式环境必须申请订阅。因此为了防止到线上无法消费消息，建议从日常环境开始就申请订阅。
