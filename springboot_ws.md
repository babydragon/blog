# springboot websocket

## 后端配置

springboot中增加websocket支持，相关文档在http://docs.spring.io/spring/docs/current/spring-framework-reference/html/websocket.html

首先需要在pom中引入依赖：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

引入依赖之后，开启websocket功能，和其他springboot扩展一样，通过```@EnableWebSocket```注解即可。

我们只使用标准的websocket，而不是用其上层包装的[STOMP](https://en.wikipedia.org/wiki/Streaming_Text_Oriented_Messaging_Protocol)协议。因此，首先需要实现一个对应的handler。

```java
@Component("taskHandler")
public class TaskHandler extends TextWebSocketHandler {
}
```
通常handler可以继承```BinaryWebSocketHandler```或者```TextWebSocketHandler```两个类，支持两种数据格式。通常情况下用文本格式的应该会比较多。

handler定义了一系列websocket通信流程中的接口，包括连接建立（```afterConnectionEstablished```）、消息接收（```handleTextMessage```）、连接关闭（```afterConnectionClosed```）等。可以通过试下这些方法来实现业务逻辑。本文实现的逻辑比较简单，是一个类似消息订阅的功能，客户端通过websocket协议创建连接之后，服务端持有连接session，当有消息需要发送时，向客户端进行广播。

为了实现该功能，首先需要重写```afterConnectionEstablished```方法，当连接建立的时候，将seesion保存起来。这里需要注意一点，为了减少客户端和服务端交互带来的复杂度，服务端直接获取了客户端建立连接时附带的参数。但是由于这个参数来自于http请求，除了hander之外，还需要为这个handler增加一个```HandshakeInterceptor```。

```HandshakeInterceptor```可以在客户端和服务端进行websocket握手时，获取到中间信息，并将其保存到websocket session的attribute中。
```java
public class HandshakeParameterInterceptor implements HandshakeInterceptor {
    @Override
    public boolean beforeHandshake(ServerHttpRequest serverHttpRequest, ServerHttpResponse serverHttpResponse,
                                   WebSocketHandler webSocketHandler, Map<String, Object> attribute) throws Exception {
        if (serverHttpRequest instanceof ServletServerHttpRequest) {
            ServletServerHttpRequest request = (ServletServerHttpRequest) serverHttpRequest;
            Map<String, String[]> parameterMap = request.getServletRequest().getParameterMap();
            Map<String, String> httpParams = parameterMap.entrySet().stream().filter(entry -> entry.getValue().length > 0)
                    .collect(Collectors.toMap(Map.Entry::getKey, entry -> entry.getValue()[0]));
            attribute.putAll(httpParams);
            return true;
        }

        return false;
    }

    @Override
    public void afterHandshake(ServerHttpRequest serverHttpRequest, ServerHttpResponse serverHttpResponse,
                               WebSocketHandler webSocketHandler, Exception e) {
        // to nothing
    }
}
```

```HandshakeInterceptor```接口有方法，分别表示握手前和握手后，由于我们需要获取http请求的参数，所以选择拦截握手前的参数。这里看上去和servlet中处理类似，通过request获取到请求参数，并将其全部放置到attribute参数中，以保证后续websocket seesion能够读取到。

这里插播下websocket的握手流程，以便更好的理解为什么能够握手拦截器来获取到请求参数。websocket握手开始于客户端发起的http请求，客户端会发送一个标准的HTTP 1.1请求头，唯一不同的是会带有Upgrade头，其值是websocket，Connection头的值设置为Upgrade。表示该请求要求服务端对连接协议进行升级，升级成websocket协议。请求头类似于：
```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
```
服务端接受到请求之后，判断可以升级成websocket协议，会响应HTTP code 101,表示协议切换，例如：
```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```
此时，就完成了协议升级，后续直接发送websocket帧进行websocket交互。完整websocket握手协议，参见[RFC 6455](https://tools.ietf.org/html/rfc6455)前文的拦截器就在这时起了作用，获取到第一个HTTP请求头的参数，然后和websocket session绑定起来。

websocket连接建立之后，服务端就可以通过session对象向客户端发送消息了。

完成了handler和inteceptor之后，就需要通过配置将他们组装起来了。配置的方式需要实现```WebSocketConfigurer```接口：
```java
@Configuration
public class WsConfiguration implements WebSocketConfigurer {

    // handlers
    @Autowired
    private TaskHandler taskHandler;

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(taskHandler, "ws/topic/task")
                .addInterceptors(parameterIntercepor());
    }

    @Bean
    public HandshakeInterceptor parameterIntercepor() {
        return new HandshakeParameterInterceptor();
    }
}
```
需要实现```registerWebSocketHandlers```方法，将handler和对应的path关联起来，然后对handler设置interceptor，如果需要设置跨域请求，也可以通过```setAllowedOrigins```方法设置，确保websocket请求可以从指定域请求。这样整个springboot应用就可以实现websocket协议的响应了。

对于线上应用，还需要注意tengine（nginx）的配置。默认情况下，HTTP的Upgrade头不会代理到后端，需要在nginx.conf文件中增加：
```
http {
    ...
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }
    server {
        location / {
                proxy_pass   http://backends;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection $connection_upgrade;
                proxy_set_header Host $host;
        }
    }
    ...
}
```

## 前端配置
后端完成之后，前端也需要实现websocket请求。前端的websocket请求，可以直接使用对应的[接口](https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket)。对于reactjs实现的前端，可以使用[react-websocket](https://github.com/mehmetkose/react-websocket)组件来实现。

该组件可以直接在render函数中添加```<Websocket />```标签来使用。该标签中最重要的属性就是url，表示websocket的连接地址，```onMessage```属性构建数据的回调。这样可以很简单的通过websocket来向后端订阅数据。不过该组件只适用于订阅websocket消息，不适用于双工交互。
