# WADL REST API的描述语言

## 简介
WADL（Web Application Description Language）是一种用于描述web应用对外提供接口语言，它使用XML来表示。其中主要包括每个资源的定位（URI）、请求类型（GET、POST等）、请求参数类型（路径参数、query参数等）、响应值和响应类型等一系列内容。根据这个XML，可以直接读取到系统提供的所有REST API，并能够进行调用。

由于WADL使用XML描述，对于开发者来说，可以通过标准的xslt进行转换，将机器读取的XML转换成便于人读取的HTML，进而行程API文档。关于WADL具体格式，参见w3c的[Web Application Description Language](https://www.w3.org/Submission/wadl/)

## 实现
对于使用springboot的微服务应用来说，会使用两种框架实现REST服务，一种是jersey，一种是spring MVC。这里不进行二者实现的比较和使用方式介绍，仅对于WADL相关内容做出描述。

### jersey
jersey是一个完整实现JAX-RS API，并对其进行了扩展的REST API库，springboot对其有对应的starter，引入和使用都非常方便。jersey本身已经自带了WADL生成，因此如果使用jersey作为REST API框架，不需要任何修改，系统已经自带了WADL文件的生成。其默认地址是/application.wadl，访问系统对应的URL即可。

### spring MVC
spring MVC默认没有提供生成WADL相关的功能，但是可以通过注入```RequestMappingHandlerMapping```对象来获取spring MVC中的映射关系，进而获取到生成WADL所需要的所有内容。

```java
public class WadlEndpoint extends AbstractMvcEndpoint {

    private static String xs_namespace="http://www.w3.org/2001/XMLSchema" ;

    @Autowired
    private RequestMappingHandlerMapping handlerMapping;

    @Autowired
    private WebApplicationContext webApplicationContext;

    public WadlEndpoint() {
        super("/application.wadl", false);
    }

    @RequestMapping(method = RequestMethod.GET, produces = MediaType.APPLICATION_XML_VALUE)
    public @ResponseBody Application generateWadl(HttpServletRequest request) {
        Application application = new Application();

        Doc doc = new Doc();
        doc.setTitle(webApplicationContext.getId() + " WADL");
        application.getDoc().add(doc);

        Resources wadlResources = new Resources();
        wadlResources.setBase(getBaseUrl(request));

        Map<RequestMappingInfo, HandlerMethod> handlerMethods = handlerMapping.getHandlerMethods();
        for (Map.Entry<RequestMappingInfo, HandlerMethod> entry : handlerMethods.entrySet()) {
            HandlerMethod handlerMethod = entry.getValue();
            Class<?> beanType = handlerMethod.getBeanType();
            if (!beanType.isAnnotationPresent(RestController.class)) {
                // not a rest controller,skip
                continue;
            }

            RequestMappingInfo mappingInfo = entry.getKey();
            Set<String> patterns = mappingInfo.getPatternsCondition().getPatterns();
            for (String uri : patterns) {
                Resource wadlResource = genResource(uri, mappingInfo, handlerMethod);
                wadlResources.getResource().add(wadlResource);
            }
        }

        application.getResources().add(wadlResources);


        return application;
    }
    ...
}
```

首先按照springboot endpoint的方式，定义一个生成WADL的endpoint，并将当前endpoint处理的URL绑定为```/application.wadl```，以便和jersey生成的路径保持一致，方便后面的xslt转换操作。

然后这个类需要注入的是```RequestMappingHandlerMapping```和```WebApplicationContext```这两个对象，前者可以获取到spring MVC中的绑定情况（通过```@RequestMapping```注解标注的方法），后者获取到对应的类，用以解析方法的入参、出参等数据。

接着就可以从```RequestMappingHandlerMapping```对象中，通过```getHandlerMethods```方法获取到当前spring容器中所有url的绑定情况。返回值是一个Map类型，key是```RequestMappingInfo```对象，其中包括了```@RequestMapping```注解中的所有字段；```HandlerMethod```是处理方法的描述，可以获取到对应的类和方法。

最后就是根据这些数据，拼装生成最终的WADL XML了。这里首先有个简单的过滤，如果处理类不是```@RestController```注解修饰的，直接跳过。因此这个WADL文件，仅针对```@RestController```注解修饰的类，不包括```@Controller```注解修饰的类。

最后，除了标准的注解之外，为了增加可读性，还新建了一个```@ApiDescribe```注解，该注解可以用于绑定方法上，更加详细的描述方法的作用。通常情况下，通过绑定方法的名称，基本上可以知道这个方法的具体业务场景和大致业务逻辑，这个注解只是作为补充，在生成Method的时候，加上对应的文档。

## 生成HTML
前面介绍了jersey和spring MVC框架下如何生成WADL文件。该文件是XML类型，如果知道每个标签的大致含义，基本上是可以看懂的。为了能够更加直观和美观的查看API文档，找到了https://github.com/ehearty/Swadl 工程，它通过xslt，将WADL的XML文档转换成类似Swagger的界面，可读性大大增强。

生成方式比较简单，该工程直接将转换代码放在了js中，也就是将xslt过程放在了浏览器中进行。因此，只需要修改下初始载入的链接，改成/application.wadl，js会完成加载对应的XML，并通过工程中的stylesheet来进行转换和美化。

相关静态资源也已经放置到工程中，因此只要工程使用了spring MVC，即可通过/wadl.html来展示当前工程的API文档。
