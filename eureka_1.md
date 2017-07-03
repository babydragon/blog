# 服务发现——基于eureka

## eureka架构

![eureka架构图](../img/eureka_architecture.png)

eureka是Netflix开源的服务发现组件，和常见服务发现/RPC框架类似，eureka由注册中心（eureka server），服务提供者，服务消费者共同组成。其中eureka server提供REST API，支持服务提供者将自己（应用级别）注册到注册中心。同时为了支持高可用，eureka server提供了数据中心的概念，同一个数据中心和不同数据中心之间支持高可用复制。参照上图，既服务提供者可以注册到不同的数据中心，这些eureka server会进行复制（eureka server的复制根据其配置是[有方向性的](http://didispace.com/springcloud6/#深入理解)），消费者获可以自行选择负载均衡策略，例如同数据中心优先等。

spring cloud提供了基于Netflix组件的封装，包括spring-cloud-starter-hystrix、spring-cloud-starter-feign、spring-cloud-starter-eureka-server、spring-cloud-starter-eureka、spring-cloud-starter-ribbon等，后面会一一介绍。

## eureka server部署

spring cloud提供了spring-cloud-starter-eureka-server包，大大简化了eureka server的部署，因此这里对代码只做一些简单的说明。

首先是pom中引入spring cloud的pom依赖，以管理版本。
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```
这里的spring-cloud.version参数，可以参照[http://projects.spring.io/spring-cloud/](http://projects.spring.io/spring-cloud/)网站上标记成current的版本，目前是“Camden.SR3”。

然后pom中就可以直接引入依赖：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

最后，代码中只需要在springboot应用入口类上添加```@EnableEurekaServer```注解，启动eureka server即可。

eureka server部署，特别是高可用部署的时候，最折腾的是配置。首先是和eureka server本身相关的配置：
（**注意：这里的配置只适用于单数据中心的高可用，没有涉及多数据中心复制**）

```properties
spring.application.name=eureka
server.port=${PORT:8761}
```
这里服务启动的端口，默认采用eureka的默认端口8761,spring.application.name这个配置比较重要，eureka client在注册服务的时候，都会使用这个配置来唯一标识应用。这也是后面所有接入应用需要规范的地方。

然后是eureka server注册相关的配置：
```
eureka.environment=aegis
eureka.enableSelfPreservation=false
eureka.client.registerWithEureka=true
eureka.client.fetchRegistry=false
eureka.server.waitTimeInMsWhenSyncEmpty=0
eureka.instance.preferIpAddress=true
```
第一个对应的是eureka server页面的Environment部分，registerWithEureka设置成true之后，eureka server自身也会注册到eureka上。preferIpAddress设置为true，由于服务器的主机名只有在机房DNS有解析，如果服务只在机房网络中调用，可以不设置，但是考虑到方便本地调试，建议后续所有应用都加上这个配置，让应用注册到eureka时的hostname字段都直接使用ip字段。

最后是复制相关的配置，eureka server复制的配置项和eureka client注册到多个eureka server相同，都通过eureka.client.serviceUrl.defaultZone来完成，其中的列表通过逗号分割。

### 部署和部署中遇到的坑
前面提到过，eureka server的复制是有方向性的，为了能够实现双向复制，最终部署到了三台机器上，并将一台机器的defaultZone指向另外两台机器。由于采用了springboot的jar包进行部署，参照[springboot文档](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html#boot-features-external-config-application-property-files)可以在部署服务器上创建config目录，通过外部的properties文件来覆盖打包在jar中的配置文件。

这里需要特别注意，需要复制的服务器列表，如果preferIpAddress设置为true之后，必须直接使用目标机器IP。之前尝试通过设置eureka.instance.hostname和url域名匹配、主机名这些，结果eureka server启动之后，这些服务器列表都被列入了unavailable-replicas。查找了对应的判断代码（com.netflix.eureka.util.StatusUtil）：
```java
private boolean isReplicaAvailable(String myAppName, String url) {

    try {
        String givenHostName = new URI(url).getHost();
        Application app = registry.getApplication(myAppName, false);
        if (app == null) {
            return false;
        }
        for (InstanceInfo info : app.getInstances()) {
            if (info.getHostName().equals(givenHostName)) {
                return true;
            }
        }
        givenHostName = new URI(url).getHost();
    } catch (Throwable e) {
        logger.error("Could not determine if the replica is available ", e);
    }
    return false;
}
```
从上述逻辑可以看出，判断复制是否有效的方式是先去注册中心获取对应的应用，然后将应用的主机名和url的host部分进行对比。由于设置了preferIpAddress配置，上报的应用只会将自己的IP作为主机名上报，如果url中使用主机名或者域名，将会导致这里无法匹配，别识别成无效的复制配置。

## 服务注册

服务提供者要注册到eureka server，需要在pom中增加依赖管理：
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```
和前文描述一样，这里spring-cloud.version的值，可以参照[spring cloud项目](http://projects.spring.io/spring-cloud/)中的版本，新项目可以选择current版本。如果是已有springboot工程，建议参照页面下方[Release Trains](http://projects.spring.io/spring-cloud/#release-trains)，对比spring cloud每个版本对应的springboot版本，跨版本可能会导致一些不可预知的问题。

确认好依赖之后，直接引入依赖即可：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

代码中的修改很少，只需要在springboot入口的Application类上增加注解```@EnableEurekaClient```即可。eureka客户端会根据基本配置，将自己注册到eureka server上。这些配置有：

```
eureka.client.serviceUrl.defaultZone=http://zk1.alibaba.net:8761/eureka,http://zk2.alibaba.net:8761/eureka,http://zk3.alibaba.net:8761/eureka
eureka.instance.leaseRenewalIntervalInSeconds=5
eureka.instance.leaseExpirationDurationInSeconds=10
eureka.instance.preferIpAddress=true
```
第一个配置制定了eureka server地址，多个地址通过逗号分割。preferIpAddress配置表示注册上去的服务，使用本机IP作为主机名。因为服务器的主机名只有在机房DNS才能解析，eureka客户端在远程调用的时候，会使用主机名加端口来调用，如果不设置，会由于主机名无法解析导致调用失败。另外两个设置了心跳间隔，默认leaseRenewalIntervalInSeconds为30s，leaseExpirationDurationInSeconds默认为90s，前者值越小，注册发现间隔越短，后者值越小，服务不可用发现间隔越短。当然，心跳间隔太小，对服务器压力越大，如果对实时性要求不高，可以使用默认值。

如果需要让eureka来进行健康检查（默认情况下eureka不会进行健康检查，仅通过心跳来判断服务是否可用），可以通过配置eureka.client.healthcheck.enabled来强制开启。

注册完成之后，可以在eureka server上看见注册信息。

### 依赖冲突
直接依赖spring-cloud-starter-eureka，需要特别注意jersey的依赖。之前有部分服务型应用通过jersey来提供服务，并且通过spring-boot-starter-jersey来引入jersey。其中eureka依赖jersey 1.x，后者引入了jersey2，此时直接启动会报jersey初始化失败。目前spring cloud团队还未能解决该[问题](https://github.com/spring-cloud/spring-cloud-netflix/issues/846)，使用jersey2的应用需要特别注意，目前测试发现，通过：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
    <exclusions>
        <exclusion>
            <groupId>javax.ws.rs</groupId>
            <artifactId>jsr311-api</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```
方式排除jsr包，暂时测试eureka的服务注册和jersey的REST API都可用。完美解决需要依赖spring-cloud-starter-eureka升级依赖，目前eureka 1.6版本已经支持jersey2。

## 服务消费

服务消费者功能开启和提供者一样，首先引入依赖管理：
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

和前文描述一样，这里spring-cloud.version的值，可以参照[spring cloud项目](http://projects.spring.io/spring-cloud/)中的版本，新项目可以选择current版本。如果是已有springboot工程，建议参照页面下方[Release Trains](http://projects.spring.io/spring-cloud/#release-trains)，对比spring cloud每个版本对应的springboot版本，跨版本可能会导致一些不可预知的问题。

确认好依赖之后，直接引入依赖即可：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```
最后，和服务提供者一样，在入口类上增加注解```@EnableEurekaClient```即可激活服务发现客户端。

服务配置和服务提供者类似**必须**设置eureka.client.serviceUrl.defaultZone和spring.application.name。

### 通过RestTemplate实现负载均衡

spring web提供了RestTemplate类提供便捷的REST访问和类型转换。如果对习惯于使用RestTempate，对于eureka客户端来说，只需要通过```@LoadBalanced```注解创建一个RestTemplate对象即可注入使用：

```java
@Configuration
public class EurekaClientConfiguration {

    @LoadBalanced
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

使用的时候，直接在bean中注入RestTemplate即可，通过RestTemplate请求基于eureka服务发现的服务提供者，只需要在域名处使用注册的服务名即可。

```java
...
@Autowired
private RestTemplate restTemplate;
...
public void testLoadBalance() {
    String result = restTemplate.getForObject("http://cron/cron/ok", String.class);
    System.out.println(result);
}
```

如上述代码片段，RestTemplate对象调用服务CRON时，只需要在url的域名处直接使用服务名。此时，eureka客户端会默认使用ribbon来处理负载均衡问题。ribbon的相关配置，可以参照```org.springframework.cloud.netflix.ribbon.eureka.RibbonEurekaAutoConfiguration```类上的注解来了解，通过上面的注解我们也可以发现，ribbon可以通过```ribbon.eureka.enabled=false```来进行关闭。

如果需要自定义ribbon的负载均衡策略，可以参照[spring cloud的文档](http://cloud.spring.io/spring-cloud-static/Camden.SR3/#_customizing_the_ribbon_client)。默认的负载均衡规则为```ZoneAvoidanceRule```。

如果代码中需要混用支持负载均衡的RestTemplate和不支持负载均衡的RestTemplate，参照[文档](http://cloud.spring.io/spring-cloud-static/Camden.SR3/#_multiple_resttemplate_objects)声明高优先级非负载均衡RestTemplate，然后通过```@LoadBalanced```注解限定注入负载均衡版本。

### 使用Feign客户端

[Feign](https://github.com/Netflix/feign)客户端是一个声明式REST客户端，它可以通过一系列注解，无需编码即可创建一个REST客户端。spring对其进行了再封装，支持直接通过spring mvc的注解来标注请求内容。

要使用Feign，首先需要引入依赖：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-feign</artifactId>
</dependency>
```
然后，和所有spring starter一样，在入口处通过注解```@EnableFeignClients```声明启动Feign。创建一个Feign客户端非常简单，新建一个接口，通过```@FeignClient```注解标注即可：

```java
@FeignClient("cron")
public interface CronClient {

    @RequestMapping(method = RequestMethod.GET,
            value = "/cron/{name}", produces = "application/json")
    Cron getCron(@PathVariable("name") String name);
}
```

这里定义一个获取定时器的客户端，```@FeignClient```注解参数为调用服务名称，需要和注册到eureka上的服务名称相同。其中的方法和spring mvc中定义一个Controller方法类似，通过```@RequestMapping```注解指定如何请求对应的服务，spring将会默认使用```HttpMessageConverters```将响应内容转换成java bean。

然后，就可以直接通过注入这个client来请求对应的服务：
```java
...
@Autowired
private CronClient cronClient;
...
@Test
public void testFeignClient() {
    String name = "1811133_aenv_deploy";
    Cron cron = cronClient.getCron(name);

    System.out.println(cron);

    assertThat(cron, is(not(nullValue())));
    assertThat(cron.getName(), is(equalTo(name)));
}
```

默认情况下，Feign会使用ribbon作为负载均衡客户端实现，Hystrix作为服务熔断器。Hystrix是Netflix开源的服务熔断器，它能够在依赖服务出现问题式提供熔断机制，避免服务异常引起雪崩。通过Feign相关注解创建的客户端默认会启动Hystrix功能，如果需要提供fallback机制，可以在```@FeignClient```注解中通过fallback属性定制fallback行为。具体请参考[文档](http://cloud.spring.io/spring-cloud-static/Camden.SR3/#spring-cloud-feign-hystrix-fallback)，Hystrix的具体使用后续会提供单独文档。
