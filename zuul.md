# Zuul——应用网关

Zuul是Netflix开源的应用网关，其主要作用是一个反向代理，将请求请求分发到后端实际提供REST服务的应用上去。相比于普通的方向代理应用（如nginx、haproxy等），Zuul的最大优势是和Netflix的Hystrix、Eureka、Ribbon等组件能够完美融合，实现通过Eureka注册中心，自动将注册上来的服务暴露出去。当然，Zuul也支持自定义filter，以在代理请求前后修改request和response。

## 应用搭建
使用spring cloud来构建一个基础功能的Zuul非常简单。和一般的spring cloud应用一样，在dependencyManagement中引入spring cloud的依赖管理：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-dependencies</artifactId>
    <version>${spring-cloud.version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```
同样的，spring cloud的当前版本，参照http://projects.spring.io/spring-cloud/ 页面选择对应的版本。
然后在dependencies中增加zuul的依赖：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zuul</artifactId>
</dependency>
```

最后，只需要在应用执行入口类上增加注解：```@EnableZuulProxy```即可开启。

另外，如果需要开启从eureka获取注册的服务，在依赖中再增加：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```
spring cloud starter zuul发现包含了eureka之后，会自动初始化DiscoveryClient，以获取注册到eureka的服务。

## 一些配置
搭建zuul服务代码很简单，一个注解即可搞定。由于zuul使用了eureka服务发现，还需要一些eureka相关的配置。第一个是eureka的地址，和其他服务发现客户端一样，通过```eureka.client.serviceUrl.defaultZone```配置。另外，由于网关无法预知后端服务性能，可能需要设置下超时时间：
```
# timeout
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=60000
ribbon.ConnectTimeout=3000
ribbon.ReadTimeout=60000
```
这里分别设置了hystrix默认超时时间和ribbon的请求超时时间，避免一些耗时应用因为网关原因无法完成数据传输。当然，超时时间设置的越长，对网关压力越大。

针对zuul本身，spring cloud同时提供了丰富的配置，可以通过这些配置修改zuul的路由策略，避免自己编写filter。zuul相关的配置都在```org.springframework.cloud.netflix.zuul.filters.ZuulProperties```类中，根据其中的字段配置即可。这里介绍几个常用的：

**注意：代码中驼峰格式的，在配置文件中可以用-分割，然后字母小写（如ignoredServices，在properties文件中可以通过zuul.ignored-services来配置）**

* addHostHeader：是否透传http request中的Host头，默认为false，如果后端应用需要获取Host头，可以设置为true来开启。
* ignoredServices：需要忽略的服务，默认情况下，服务发现客户端发现的所有服务都会被代理，如果需要忽略，可以在这里配置，多个值用逗号分割。
* sensitiveHeaders：设置为敏感的http头字段，默认值为Cookie、Set-Cookie、Authorization。默认情况下，设置为敏感的头字段会被过滤，后端服务无法获取到这些头。因此，默认情况下，后端服务是**无法**获取到请求中的cookie的。

除了这些全局配置，zuul可以通过在配置文件中静态配置的方式来固定路由属性。路由相关的配置对象为：```org.springframework.cloud.netflix.zuul.filters.ZuulProperties.ZuulRoute```，其中主要包括这几个核心字段：

* id：路由标识，对于动态从服务发现客户端中生成的路由规则，id为服务名。
* path：路由对应的url路径模式。对于从服务发现客户端生成的路由规则，默认为/${id}/\*\*。既url的第一段作为服务名的匹配。
* serviceId：后端服务id，默认为eureka的服务名。类似于FeignClient中定义的名称，一般无需单独配置，除非针对该路由后端有独立的请求方式。
* url：后端的url，对于非服务发现类的后端，可以通过配置该参数将请求代理到实际的http服务提供者。**注意：serviceId和url互斥，只能设置一个**。
* stripPrefix：是否去除url前缀，默认值为true。该值也可以全局配置，这里配置只针对单条路由有效，表示将path中匹配到的数据去除后拼接实际的后端请求url。
* sensitiveHeaders：这里可以独立设置，以覆盖全局的http请求头中敏感字段设置。


通常配置可以是这样的（这里是yaml格式，properties格式类似）：

```
zuul:
  routes:
    users:
      path: /myusers/**
      url: http://example.com/users_service
```

上述配置表示：如果url格式为/myusers/\*\*，则将请求代理到```http://example.com/users_service```。此时请求网关```http://gateway.example.com/myusers/u1```，网关实际会去请求```http://example.com/users_service/u1```。
