# 从0开始学kubernetes——临时容器

## 介绍
serverless一般要求pod中运行的容器无状态并足够轻量，这样才能更容易的进行调度。目前我们也正在推进将原有的富容器逐步推进修改成轻量容器，既尽可能确保业务容器中只运行业务进程，同时对镜像进行瘦身。

但是，容器瘦身之后遇到的第一个问题，就是运维工具的缺失，导致难以对业务进程进行调试。这里用一个最简单的例子来展示，当我们将java的容器精简到使用
`openjdk:8-jre-alpine`之后，就会无法使用jdk提供的一些调试工具，例如`jstack`、`jmap`等。

此时，就可以使用k8s的临时容器，在需要调试的时候，将依赖的jdk容器注入到业务容器所在的pod中。

## 验证步骤

### 预先准备
首先需要准备一个最简单的Java应用镜像，为了模拟这个必要性，镜像使用`openjdk:8-jre-alpine`作为基础镜像，这样就不会带有任何jdk的工具。
```Dockerfile
ROM openjdk:8-jre-alpine

RUN apk add --no-cache tini

COPY Simple.class /app/Simple.class
WORKDIR /app

CMD ["/sbin/tini", "--", "java", "Simple"]
```

注意，由于alpine容器中，如果java进程的PID为1时，`jstack`、`jmap`什么的会报错，提示：
```
1: Unable to get pid of LinuxThreads manager thread
```

`Simple`这个类很简单：

```java
class Simple {
	public static void main(String[] args) throws Exception {
		while(true) {
			System.out.println("I'm still here!");
			Thread.sleep(10000L);
		}
	}
}
```
因此没有使用docker镜像来构建，直接用javac命令编译成了class文件，在前面的Dockerfile里面，直接复制进去了。

### 创建pod
首先，先创建一个简单的deployment：
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-java
  labels:
    app: simple-java
spec:
  replicas: 1
  selector:
    matchLabels:
      app: simple-java
  template:
    metadata:
      labels:
        app: simple-java
    spec:
      containers:
      - name: simple-java
        image: simple_java:latest
        volumeMounts:
        - mountPath: /tmp
          name: tmp-dir
      volumes:
      - name: tmp-dir
        emptyDir: {}
```

这里的镜像使用刚才制作的镜像。使用了挂载点，是为了简化jdk命令的执行，因为java attach依赖/tmp目录中的unix socket。

启动本地minikube之后，apply下这个yaml文件，即可创建对应的deployment和pod了。

### 注入临时容器
注入前，按照k8s官方文档，需要提前准备一个临时文件的json：
```json
{
    "apiVersion": "v1",
    "kind": "EphemeralContainers",
    "metadata": {
            "name": "simple-java-58b59d9456-wqg7p"
    },
    "ephemeralContainers": [{
        "command": [
            "sh"
        ],
        "image": "openjdk:8-alpine",
        "imagePullPolicy": "IfNotPresent",
        "name": "debugger",
        "stdin": true,
        "tty": true,
        "targetContainerName": "simple-java",
        "terminationMessagePolicy": "File",
        "volumeMounts": [{
            "name": "tmp-dir",
            "mountPath": "/tmp"
        }]
    }]
}
```
这里由于我们使用了deployment，因此pod的名字需要修改成实际pod的名称。使用的镜像是jdk，这样就会包含我们需要的jdk命令了。这里需要特别注意两个地方：

1. 必须指定targetContainerName，只有具体指定了目标容器名称，k8s在创建临时容器的时候，才会和它共享pid ns，否则会看不到我们需要操作进程的PID。
2. 为了方便执行，这里和主容器一起共享了/tmp，目的仅仅是为了能够方便java attach

现在来应用这个临时容器配置：
```bash
k replace --raw /api/v1/namespaces/default/pods/simple-java-58b59d9456-wqg7p/ephemeralcontainers -f simple-java-debugger.json
```
api路径中的pod名称，同样需要修改成实际pod的名称。最后的json文件就是上面的那个。

执行成功之后，如果我们再次获取pod的yaml配置，就可以看见临时容器的配置：
```yaml
ephemeralContainers:
  - command:
    - sh
    image: openjdk:8-alpine
    imagePullPolicy: IfNotPresent
    name: debugger
    resources: {}
    stdin: true
    targetContainerName: simple-java
    terminationMessagePolicy: File
    tty: true
    volumeMounts:
    - mountPath: /tmp
      name: tmp-dir
```

此时，可以直接attach到临时容器上去：
```bash
k attach -it -c debugger simple-java-58b59d9456-wqg7p
```

由于共享了pid ns，在临时容器中，我们可以看见目标java进程：
```bash
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 /sbin/tini -- java Simple
    6 root      0:00 java Simple
   16 root      0:00 sh
   22 root      0:00 ps aux
```

通过`jstack 6`命令，可以获取目标java进程的线程栈信息。

调试完成退出临时容器之后，这个容器会被销毁，无法再次attach:
```
kubectl attach simple-java-58b59d9456-wqg7p -c debugger -i -t
If you don't see a command prompt, try pressing enter.
error: unable to upgrade connection: container debugger not found in pod simple-java-58b59d9456-wqg7p_default
```

此时describe下这个pod，可以看见：
```
Ephemeral Containers:
  debugger:
    Container ID:  docker://bf31d7c59f41a8522f3a557b241d54a1db08aad4cf12cec5124466c4c742afcd
    Image:         openjdk:8-alpine
    Image ID:      docker-pullable://openjdk@sha256:94792824df2df33402f201713f932b58cb9de94a0cd524164a0f2283343547b3
    Port:          <none>
    Host Port:     <none>
    Command:
      sh
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Tue, 04 Aug 2020 15:26:10 +0800
      Finished:     Tue, 04 Aug 2020 15:34:14 +0800
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /tmp from tmp-dir (rw)
```
临时容器状态已经是Terminated。

## 总结
临时容器特别适合包含主容器剥离出来的一些调试工具，在需要的时候临时注入到目标pod中。不过临时容器目前还处于alpha状态，有个比较尴尬的问题，就是在pod中添加临时容器之后，目前还无法删除，同时如果这时候临时容器已经退出，会导致无法再次attach，也不会被拉起（临时容器不支持probe什么的），相关的issue：https://github.com/kubernetes/kubernetes/issues/84764

## 补充
目前临时容器最大的坑是无法删除，如果attach了临时容器，然后退出了容器主进程（和前面示例中展示的那样），会导致这个容器无法再attach，也无法重新启动。此时，如果要重复上述步骤再次进行调试，需要新创建一个临时容器，同时还需要保留老的配置，否则k8s会拒绝新的配置：

```json
{
    "apiVersion": "v1",
    "kind": "EphemeralContainers",
    "metadata": {
            "name": "simple-java-58b59d9456-wqg7p"
    },
    "ephemeralContainers": [{
        "command": [
            "sh"
        ],
        "image": "openjdk:8-alpine",
        "imagePullPolicy": "IfNotPresent",
        "name": "debugger",
        "stdin": true,
        "tty": true,
        "targetContainerName": "simple-java",
        "terminationMessagePolicy": "File",
        "volumeMounts": [{
            "name": "tmp-dir",
            "mountPath": "/tmp"
        }]
    },{
        "command": [
            "sh"
        ],
        "image": "openjdk:8-alpine",
        "imagePullPolicy": "IfNotPresent",
        "name": "debugger1",
        "stdin": true,
        "tty": true,
        "targetContainerName": "simple-java",
        "terminationMessagePolicy": "File",
        "volumeMounts": [{
            "name": "tmp-dir",
            "mountPath": "/tmp"
        }]
    }]
}
```
配置文件需要做这样的修改，再新增一个临时容器。重新修改后pod的状态（describe结果）：
```
Ephemeral Containers:
  debugger:
    Container ID:  docker://bf31d7c59f41a8522f3a557b241d54a1db08aad4cf12cec5124466c4c742afcd
    Image:         openjdk:8-alpine
    Image ID:      docker-pullable://openjdk@sha256:94792824df2df33402f201713f932b58cb9de94a0cd524164a0f2283343547b3
    Port:          <none>
    Host Port:     <none>
    Command:
      sh
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Tue, 04 Aug 2020 15:26:10 +0800
      Finished:     Tue, 04 Aug 2020 15:34:14 +0800
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /tmp from tmp-dir (rw)
  debugger1:
    Container ID:  docker://2312f07e335a8317cf1790f9ffb8b892fdfe0e0f080430f67c6349de4424d160
    Image:         openjdk:8-alpine
    Image ID:      docker-pullable://openjdk@sha256:94792824df2df33402f201713f932b58cb9de94a0cd524164a0f2283343547b3
    Port:          <none>
    Host Port:     <none>
    Command:
      sh
    State:          Running
      Started:      Thu, 06 Aug 2020 13:53:41 +0800
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /tmp from tmp-dir (rw)
```
此时，除了前面创建的名为debugger容器之外，又新增了一个debugger1容器。