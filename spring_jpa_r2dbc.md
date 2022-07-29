# 同时使用spring data jpa和spring data r2dbc

## 动机
计划把原先基于spring mvc的应用，web层改成基于spring webflux,因此针对底层数据源也希望能够直接使用reactive模式，既使用spring data r2dbc。

但是，之前用了很多jpa的特性，无法一次性整体移除jpa依赖，最终考虑jpa和r2dbc并存的方案，
同时优先迁移查询频率搞的业务，以体现出r2dbc带来的IO优势。

## 实施
**注：本文基于spring-boot 3.0 milestore版本**
### 依赖引入
首先增加spring data r2bdc依赖：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
```

然后是针对数据库的驱动，这里由于不同profile会使用不同的驱动，因此需要添加两种，
第一个是针对本地h2的驱动：
```xml
<dependency>
  <groupId>io.r2dbc</groupId>
  <artifactId>r2dbc-h2</artifactId>
</dependency>
```
然后是针对测试环境mysql的驱动：
```xml
<dependency>
  <groupId>org.mariadb</groupId>
  <artifactId>r2dbc-mariadb</artifactId>
  <exclusions>
    <exclusion>
      <groupId>io.zipkin.brave</groupId>
      <artifactId>brave-instrumentation-http</artifactId>
    </exclusion>
  </exclusions>
</dependency>
<dependency>
  <groupId>io.r2dbc</groupId>
  <artifactId>r2dbc-pool</artifactId>
</dependency>
```
这里连接mysql用了r2dbc-mariadb,对比了下这个驱动相比r2dbc-mysql要活跃不少。
不过需要注意的是，`r2dbc-mariadb`这个依赖会间接引入brave,然后触发springboot初始化
zipkin相关组件。但是它的依赖又不完全，会导致springboot初始化失败。因此这里直接exclude。

### 代码拆分和配置
完成依赖引入之后，就可以开始对代码进行修改了。为了迁移方便，将整个dao层整体(包括实体类和repository)
拆分成jpa包和r2dbc包，随后通过配置类来进行初始化。

首先是jpa的初始化。由于jpa + hibernate的自动装配，依赖数据源的自动初始化;而引入了r2dbc之后，会破坏初始化，
最终导致缺少本可以自动装配的`entityManagerFactory`bean而初始化失败。解决方式就是自己配置这些bean。

```java
@EnableJpaRepositories(basePackages = "com.hyperpaas.storage.jpa")
@Configuration
@EnableConfigurationProperties({JpaProperties.class, HibernateProperties.class})
public class JPAConfiguration {

    @Resource
    private JpaProperties jpaProperties;

    @Resource
    private HibernateProperties hibernateProperties;

    @Resource
    DataSourceProperties dataSourceProperties;

    @Bean(name = "jpaDataSource")
    public DataSource jpaDataSource() {
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName(dataSourceProperties.getDriverClassName());
        dataSource.setUrl(dataSourceProperties.getUrl());
        dataSource.setUsername(dataSourceProperties.getUsername());
        dataSource.setPassword(dataSourceProperties.getPassword());

        return dataSource;
    }

    @Bean(name = "entityManagerFactory")
    public LocalContainerEntityManagerFactoryBean entityManager() {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(jpaDataSource());
        em.setPackagesToScan("com.hyperpaas.platform.account.storage.source.jpa");
        em.setPersistenceProvider(new HibernatePersistenceProvider());
        em.setJpaVendorAdapter(jpaVendorAdapter());
        em.setJpaPropertyMap(extraProperties());

        return em;
    }

    private JpaVendorAdapter jpaVendorAdapter() {
        HibernateJpaVendorAdapter adapter = new HibernateJpaVendorAdapter();
        adapter.setShowSql(jpaProperties.isShowSql());
        if (jpaProperties.getDatabase() != null) {
            adapter.setDatabase(jpaProperties.getDatabase());
        }
        if (jpaProperties.getDatabasePlatform() != null) {
            adapter.setDatabasePlatform(jpaProperties.getDatabasePlatform());
        }
        adapter.setGenerateDdl(jpaProperties.isGenerateDdl());

        return adapter;
    }

    private Map<String, Object> extraProperties() {
        Supplier<String> defaultDdlMode = () -> getDefaultDdlAuto(jpaDataSource());
        return this.hibernateProperties.determineHibernateProperties(jpaProperties.getProperties(),
                new HibernateSettings().ddlAuto(defaultDdlMode));
    }

    @Bean(name = "transactionManager")
    public PlatformTransactionManager transactionManager() {
        JpaTransactionManager transactionManager = new JpaTransactionManager();
        transactionManager.setEntityManagerFactory(entityManager().getObject());
        return transactionManager;
    }

    private String getDefaultDdlAuto(DataSource dataSource) {
        if (!EmbeddedDatabaseConnection.isEmbedded(dataSource)) {
            return "none";
        }
        return "create-drop";
    }
}
```

这里定义了3个bean,分别是：
* jpaDataSource：datasource,主要用于装配上层的管理器
* entityManagerFactory：实例管理工厂类
* transactionManager：事务管理

为了初始化的时候不会混淆jpa和r2dbc，jpa只扫描jpa包下面的对象。
另外，这里尽可能的参照`org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaConfiguration`
类中初始化对应bean的方式，以确保在配置层面完全兼容（包括使用原配置定义是否自动执行DDL等）。

因此，虽然是自定义的配置类，但是仍然使用了jpa定义的`JpaProperties`和`HibernateProperties`两个属性类。

相对来说r2dbc的配置就方便许多了（但是也有坑）。
```java
@EnableR2dbcRepositories(basePackages = "com.hyperpaas.storage.r2dbc")
@Configuration
public class R2dbcConfiguration {
}
```
实际上只需要通过`EnableR2dbcRepositories`注解开启即可，同样为了区分，指定了只扫描r2dbc包下的文件。

### r2dbc使用
对于通常查询，spring-data-r2dbc和jpa一样，都能够通过在Repository里面定义函数之后，自动生成SQL。
二者唯一的区别，只是repository接口继承的接口不同而已。针对r2dbc,需要继承`ReactiveCrudRepository`
这个接口。另外还有一点需要注意的是，r2dbc是基于reactive的，因此接口的返回值也需要是`Mono`或者`Flux`。

一个简单的实例：
```java
public interface AccountReactiveRepository extends ReactiveCrudRepository<Account, Long> {
    /**
     * 通过手机号和租户查询账户
     * @param phone
     * @param tenantId
     * @return
     */
    Mono<Account> findAccountByPhoneNumAndTenantId(String phone, UUID tenantId);
}
```

## 采坑和总结
整个接入过程中还是踩了不少的坑，最开始遇到的，就是仅使用`EnableR2dbcRepositories`和
`EnableJpaRepositories`两个注解时，springboot初始化失败的问题。具体原因前文已经提到了，
者也导致了jpa的配置类变得异常的复杂。

后面的坑都和`r2dbc-maridb`有关。

### springboot启动报：zipkin2.reporter.urlconnection.URLConnectionSender类找不到
由于之前一直没有接入过zipkin,也没有搭建过zipkin server（当然，后续肯定是要搞的，
还得接入到grafana中），出现这个错误比较意外。

首先，确认报错原因来自：`org.springframework.boot.actuate.autoconfigure.tracing.zipkin.ZipkinConfigurations.SenderConfiguration`这个内部类。
而这个Sender开始自动装配，是因为`ZipkinAutoConfiguration`这个自动装配类满足了条件：
`@ConditionalOnClass(Sender.class)`，既classpath中包含了`zipkin2.reporter.Sender`这个类。

通过maven dependency确认依赖来源：
```
[INFO] +- org.mariadb:r2dbc-mariadb:jar:1.1.1-rc:compile
[INFO] |  \- io.projectreactor.netty:reactor-netty:jar:1.1.0-M2:compile
[INFO] |     \- io.projectreactor.netty:reactor-netty-http-brave:jar:1.1.0-M2:runtime
[INFO] |        \- io.zipkin.brave:brave-instrumentation-http:jar:5.13.9:runtime
[INFO] |           \- io.zipkin.brave:brave:jar:5.13.9:runtime
[INFO] |              \- io.zipkin.reporter2:zipkin-reporter-brave:jar:2.16.3:runtime
[INFO] |                 \- io.zipkin.reporter2:zipkin-reporter:jar:2.16.3:runtime
[INFO] |                    \- io.zipkin.zipkin2:zipkin:jar:2.23.2:runtime
```

解决方案参照前文，直接在引入r2dbc-mariadb的地方exclude掉brave相关依赖即可。

### r2dbc-mariadb解析uuid字段报错：IllegalArgumentException: No encoder for class java.util.UUID
由于代码中的ID类字段都使用的是UUID类型，而spring-data-r2dbc和maridb都无法处理UUID。这个错误来自于
`org.mariadb.r2dbc.codec.Codecs`这个类，其中预定义了一系列内置解码器。
需要注意的是，这个列表被声明成了`public static final`，因此即使可以读取，也没法往里面添加。

不过spring data在上层支持自定义spring通用converter来转换数据类型，可以用来解决这个问题。
当时，为了方便直接查询，这些UUID在JPA中声明的类型是字符类型，因此JPA插入mysql使用的是VARCHAR类型。
如果是BINARY(16)格式，转换的时候还需要特别注意字节序！

首先需要定义两个converter,用来做UUID <=> String的转换。

```java
@WritingConverter
public class UUID2StringConvertor implements Converter<UUID, String> {
    @Override
    public String convert(UUID uuid) {
        return uuid.toString();
    }
}
```

```java
@ReadingConverter
public class String2UUIDConvertor implements Converter<String, UUID> {
    @Override
    public UUID convert(String str) {
        return UUID.fromString(str);
    }
}
```

最后进行装配：
```java
@Configuration
public class R2dbcMysqlConfiguration {

    @Resource
    private DatabaseClient databaseClient;

    @Bean
    public R2dbcCustomConversions r2dbcCustomConversions() {
        R2dbcDialect dialect = DialectResolver.getDialect(this.databaseClient.getConnectionFactory());
        List<Object> converters = new ArrayList<>(dialect.getConverters());
        converters.addAll(R2dbcCustomConversions.STORE_CONVERTERS);

        List<Object> customConverters = new ArrayList<>();
        customConverters.add(new String2UUIDConvertor());
        customConverters.add(new UUID2StringConvertor());
        return new R2dbcCustomConversions(
                CustomConversions.StoreConversions.of(dialect.getSimpleTypeHolder(), converters),
                customConverters);
    }
}
```

这样就可以通过自定义`R2dbcCustomConversions`来覆盖默认的bean,往里面增加自定义的converter。

### 总结一下
目前通过这种方式，将同一个数据库的读写分成了两类repository,这样做的好处是可以优先在上层接入
spring webflux。在spring webflux的controller中，针对r2dbc的数据库操作可以直接按照reactive的
方式进行，而仍然是JPA方式的数据库操作，可以先通过`subsribeOn`的方式，将其放到一个独立的线程池中运行。

当然，这样的方式也大大增加了维护成本，不过由于目前DB结构都比较简单，后续的改动，希望的是慢慢从JPA迁移到r2dbc上来。另外，针对大量的查询操作，也是迁移的重点对象。剩下的插入、更新、删除等低频操作，仍然保留使用JPA影响也不会很大，毕竟这类操作业务上频率不会太高，给一个合适大小的线程池能够隔离运行即可。
同时，保留了JPA，还能利用JPA提供的自动执行DDL等功能，降低开发时的维护成本。