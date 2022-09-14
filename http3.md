# 博客支持http3(QUIC)

## 编译安装nginx
目前nginx有一个专门支持http3的分支，但是没有合并到主干，所以得自己编译安装。

官方网页在[这里](https://quic.nginx.org/)

具体安装方式可以参照[readme](https://quic.nginx.org/readme.html)来进行，这里简单描述下。

首先需要编译[boringssl](https://boringssl.googlesource.com/boringssl/)，这个是google用来替代openssl的库，nginx的QUIC模块依赖它提供的加解密功能。
```bash
git clone https://boringssl.googlesource.com/boringssl/
mkdir boringssl/build
cd boringssl/build
cmake ..
make
```
boringssl使用cmake作为依赖管理，因此只需要按照cmake标准构建方式，生成Makefile,然后通过`make`命令进行编译即可。

然后是编译nginx,需要使用hg命令
```bash
hg clone -b quic https://hg.nginx.org/nginx-quic
cd nginx-quic
./auto/configure --prefix=/opt/nginx --with-http_v2_module \
  --with-http_v3_module --with-stream_quic_module \
  --with-cc-opt="-I../boringssl/include" \
  --with-ld-opt="-L../boringssl/build/ssl -L../boringssl/build/crypto"
make
make install
```
nginx是标准的automake风格，需要注意的是，虽然支持quic的nginx用了一个独立的仓库，
还要记得clone之后checkout到quic分支。

执行`configure`的时候有几个开关需要注意：

* prefix：安装的路径，仅仅指定prefix,会按照nginx的目录结构进行安装
* with-http_v3_module：开启http3支持
* with-cc-opt/with-ld-opt：需要指定刚刚编译的boringssl路径，包括头文件路径和lib路径，这里通过给编译器增加-I和-L参数来增加查找目录。

然后就是`make`和`make install`了，注意因为要安装到/opt/nginx目录，需要root权限。

## 配置
```
server {
        server_name coolex.info;
        root /var/www/localhost/htdocs;
        listen [::]:443 http3 reuseport;
        listen [::]:443 ssl http2;

        ssl_certificate     /etc/letsencrypt/live/xxx/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/xxx/privkey.pem;
        ssl_protocols       TLSv1.3;
        ssl_early_data on;


        add_header Alt-Svc 'h3=":443"; ma=86400, h3-29=":443";ma=86400,h3-27=":443";ma=86400';
        index index.php;
}
```
这里摘录一段博客的配置。

* listen：这里写了两条，因为http3基于quic协议，使用的是udp协议，所以需要同时指定443端口支持http3和http2。这样nginx会同时监听tcp的443端口和udp的443端口。同时还要注意，因为博客支持ipv6解析，所以还需要增加ipv6的监听。
* ssl_certificate/ssl_certificate_key：和http2一样，http3也必须使用SSL,
不过之前博客就用了let‘s encrypt的证书，这里直接使用即可。
* ssl_protocols：http3需要TLS1.3
* ssl_early_data：开启TLS 1.3协议支持的0-RTT，以加快重联速度。
* Alt-Svc头:[Alt-Svc](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Alt-Svc)头用于告诉浏览器支持http3协议。目前浏览器默认都会发起http2请求，当获取到这个头时，后续请求可以切换到http3。后面的ma参数（max age）表示这个头的缓存时间。具体可以参考[rfc7838](https://httpwg.org/specs/rfc7838.html#caching-alt-svc-header-field-values)。

由于nginx不是系统安装的，需要单独创建一个systemd的unit文件，方便管理和开机自动启动。

```
[Unit]
Description=The NGINX HTTP and reverse proxy server
After=syslog.target network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/opt/nginx/sbin/nginx -t
ExecStart=/opt/nginx/sbin/nginx
ExecReload=/opt/nginx/sbin/nginx -s reload
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

保存到`/etc/systemd/system/nginx.service`，然后reload下就可以使用了。

## 验证
配置完成并启动后，可以通过[http3check](https://http3check.net/)来进行验证。