# Hystrix——Netflix熔断器

Hystrix是Netflix开源的熔断器模式实现。在微服务体系中，服务之间依赖众多，容易引起一个服务异常，导致级联异常（同步调用级联超时、异常扩散、线程池耗尽等等）。

![Hystrix fallback](../img/HystrixFallback.png)

如上图所示，Hystrix提供了一个fallback机制，当服务不可用（超时、发生异常）时，Hystrix会通过fallback中指定的本地方法来作为后备。

## 和Feign客户端结合
[前文](eureka_1.md#使用feign客户端)介绍了如果通过Feign，使用注解的方式创建REST客户端。其中Feign默认就会对其客户端使用Hystrix。例如：

```java
@FeignClient("cron")
public interface CronClient {
    @RequestMapping(method = RequestMethod.GET,
            value = "/cron/not-exist/{name}", produces = "application/json")
    Cron getCronFail(@PathVariable("name") String name);
}
```
还是和前文类似，这里故意写一个不存在的url，Feign请求的时候会抛出异常，同时触发Hystrix的熔断机制。默认情况下，Feign没有配置fallback，这会导致运行时Hystrix继续向上抛出```HystrixRuntimeException```，异常栈类似：

```
com.netflix.hystrix.exception.HystrixRuntimeException: CronClient#getCronFail(String) failed and no fallback available.
...
	at com.netflix.hystrix.AbstractCommand$22.call(AbstractCommand.java:805)
	at com.netflix.hystrix.AbstractCommand$22.call(AbstractCommand.java:790)
...
Caused by: feign.FeignException: status 404 reading CronClient#getCronFail(String)
	at feign.FeignException.errorStatus(FeignException.java:62)
	at feign.codec.ErrorDecoder$Default.decode(ErrorDecoder.java:91)
```

相关测试代码：
```java
@Test(expected = HystrixRuntimeException.class)
public void testHystrix() {
    String name = "1811133_aenv_deploy";
    Cron cron = cronClient.getCronFail(name);

    System.out.println(cron);
}
```

下面来尝试增加fallback机制。首先重新定义一个支持fallback的CronClient：
```java
@FeignClient(name = "cron", fallback = CronClientFallBack.class)
public interface FallBackCronClient {

    @RequestMapping(method = RequestMethod.GET,
            value = "/cron/not-exist/{name}", produces = "application/json")
    Cron getCronFail(@PathVariable("name") String name);
}
```

这里和之前的FeignClient定义类似，唯一的不同是定义了fallback。这里fallback随意定义一个默认实现：

```java
@Component
public class CronClientFallBack implements FallBackCronClient {
    @Override
    public Cron getCronFail(String name) {
        return Cron.notExist;
    }
}
```

这里只是为了测试，返回一个默认的不存在的替代类。通过单元测试进行测试：

```java
@Test
public void testHystrixFallback() {
    String name = "1811133_aenv_deploy";
    Cron cron = fallBackCronClient.getCronFail(name);

    System.out.println(cron);

    assertThat(cron, is(equalTo(Cron.notExist)));
}
```
通过断言可以发现，支持fallback的客户端，在访问出现404的时候，调用了fallback类的对应方法。通过这个机制，可以在业务上做适当的定义，将依赖服务的异常隔离在很小的范围之内。

上面示例代码，如果将Hystrix日志级别降到DEBUG（通过配置项```logging.level.com.netflix.hystrix.AbstractCommand=DEBUG```），在控制台可以看见这样的输出：
```
[ hystrix-cron-1] com.netflix.hystrix.AbstractCommand      : Error executing HystrixCommand.run(). Proceeding to fallback logic ...
feign.FeignException: status 404 reading FallBackCronClient#getCronFail(String)
	at feign.FeignException.errorStatus(FeignException.java:62) ~[feign-core-9.3.1.jar:na]
	at feign.codec.ErrorDecoder$Default.decode(ErrorDecoder.java:91) ~[feign-core-9.3.1.jar:na]
...
```
这里可以发现，Hystrix已经在调用fallback实现，避免404的异常向上传播。

参照[spring cloud文档](http://cloud.spring.io/spring-cloud-static/Camden.SR3/#spring-cloud-feign-hystrix-fallback)，除了直接指定请求的fallback类之外，还可以在```@FeignClient```注解上通过fallbackFactory属性指定FallbackFactory。这样的好处是可以针对不同的异常类型定制不同的fallback实现。这块还没有进行尝试。

## Hystrix相关配置
前面介绍通过Feign如何配置Hystrix fallback。之前测试中还发现如果Hystrix配置不当，可能会遇到执行超时等问题。Hystrix可以通过配置项配置全局的设置（主要是隔离方式和阈值），也可以针对单独的一个命令配置。

例如之前遇到的超时问题，Hystrix针对每个命令执行会默认会有1000ms的超时时间限制。这个配置可以通过：
```
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds
```
来修改执行超时时间。如果需要禁用超时，可以通过
```
hystrix.command.default.execution.timeout.enabled
```
来全局禁用。

针对Feign自动创建的Hystrix command，Feign会生成一个CommandKey，对应规则为：{ClassName}#{mathodName}({ParameterType})。注意这里的ClassName为类的简称，不包含其包名；多个参数类型用逗号隔开。如```CronClient.getCron```方法在调用时，CommandKey为```CronClient#getCron(String)```，以此类推，如果要设置这个key的超时时间，配置项为：

```
hystrix.command.CronClient#getCron(String).execution.isolation.thread.timeoutInMilliseconds=100
```

具体生成代码，可以参考```feign.Feign```类中的configKey方法：

```java
public static String configKey(Class targetType, Method method) {
  StringBuilder builder = new StringBuilder();
  builder.append(targetType.getSimpleName());
  builder.append('#').append(method.getName()).append('(');
  for (Type param : method.getGenericParameterTypes()) {
    param = Types.resolve(targetType, targetType, param);
    builder.append(Types.getRawType(param).getSimpleName()).append(',');
  }
  if (method.getParameterTypes().length > 0) {
    builder.deleteCharAt(builder.length() - 1);
  }
  return builder.append(')').toString();
}
```

最后，如果在使用Feign时希望禁用Hystrix，可以在配置项中增加：
```
feign.hystrix.enabled
```
