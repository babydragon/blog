# docker maven plugin

## 目的
docker maven plugin主要用于在maven工程中，直接进行docker镜像构建，以简化后续的镜像构建流程。本文针对springboot工程（单jar），给出建议的配置。

## 配置
首先引入依赖，通用配置如下：
```xml
<plugin>
    <groupId>com.spotify</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <configuration>
        <imageName>${docker.image.name}</imageName>
        <dockerDirectory>${project.basedir}/docker</dockerDirectory>
        <resources>
            <resource>
                <targetPath>/</targetPath>
                <directory>${project.build.directory}</directory>
                <include>${project.build.finalName}.jar</include>
            </resource>
        </resources>
    </configuration>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>build</goal>
            </goals>
        </execution>
        <execution>
            <id>tag-image</id>
            <phase>package</phase>
            <goals>
                <goal>tag</goal>
            </goals>
            <configuration>
                <image>${docker.image.name}</image>
                <newName>${docker.repo}/${docker.image.name}</newName>
                <forceTags>true</forceTags>
            </configuration>
        </execution>
        <execution>
            <id>push-image</id>
            <phase>install</phase>
            <goals>
                <goal>push</goal>
            </goals>
            <configuration>
                <imageName>${docker.repo}/${docker.image.name}</imageName>
            </configuration>
        </execution>
    </executions>
</plugin>
```
以上配置有两个约定，一个是Dockerfile文件存储在源码根目录的/docker目录下，最终只使用构建出来的${project.build.finalName}.jar。另外提取了变量docker.image.name和docker.repo，其中docker.repo主要为了能够推送到私有仓库，给下面的tag操作使用；docker.image.name可以写死，个人建议结合git-commit-id-plugin插件获取当前git提交信息，避免因为镜像名称重复，导致之前的镜像被覆盖。

```xml
<docker.version>${git.commit.id.describe}-${git.commit.id.abbrev}</docker.version>
<docker.image.name>xxx:${docker.version}</docker.image.name>
```
每个应用，可以只修改docker.image.name的前缀，后缀版本使用两个git版本信息。注意git.commit.id.describe值默认是提交版本号，建议在打包前给当前git仓库创建tag，之后git.commit.id.describe值会变成最近一个tag名称，例如：
```bash
git tag -a 1.0 -m '1.0 pub'
```
之后镜像构建的版本就变成了```xxx:1.0-abcabc```既应用名、tag名、commit id。

最后，上述配置会将docker镜像构建、tag放到package goal中，docker镜像push放到install goal中，如果要在这几个goal执行时，不进行docker镜像构建，可以加上```-DskipDocker```来跳过。docker maven plugin需要依赖本地docker daemon，因此本地需要启动docker服务，如果在jenkins上构建，需要确保slave上启动docker服务，如果在jenkins的docker容器中构建，需要确保docker容器运行时，将主机的docker unix socket挂载到容器中（既dind，docker in docker模式）。

## Dockerfile
上一节的配置，约定了Dockerfile存放在代码根目录的docker目录中，需要在其中创建Dockerfile，以真正进行镜像构建。对于springboot应用来说，只需要依赖jre即可，为了尽可能缩小镜像体积，基础镜像采用了openjdk:8-jre-alpine：

```Dockerfile
FROM openjdk:8-jre-alpine

MAINTAINER lingjie.jinlj <lingjie.jinlj@alibaba-inc.com>

RUN mkdir /work
WORKDIR /work
ADD gateway-1.0-SNAPSHOT.jar app.jar

CMD ["java", "-Dfile.encoding=UTF-8", "-server", "-Xmx512m", "-Xms512m","-jar", "app.jar"]
```
对于一般的应用，只需要修改maven构建出来的jar包名称即可。由于docker maven plugin在配置resource的时候，不支持将资源文件重命名，只能直接复制，如果需要固定每一个Dockerfile，可以通过其他maven插件，固定构建出来的jar包名称。由于使用了alpine版本的jre，整个镜像非常小，但是jre没有提供jdk中提供的一些调试工具，如果需要其中的一些工具（如jstack、jmap等）可以使用openjdk:8-jdk-alpine基础镜像，但是这样会增加整个镜像的大小。

按照上述pom中配置的镜像版本号，如果需要在shell中获取，可以通过以下命令获取：
```bash
git describe --abbrev=7 | awk -F'-' '{print $1"-"substr($3,2)}'
```
