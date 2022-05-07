# 从0开始学kubernetes——第一个CRD

## 准备工作
CRD是k8s的重要扩展方式，通过CRD可以在k8s定义的对象之上进行各种自定义扩展，第一个CRD就是对deployment和service进行了组装，简化提交内容。

基于go语言开发CRD，框架层面感觉已经比较成熟了，直接安装`kubebuilder`即可。通过`kubebuilder`，可以很快的生成代码脚手架，对于开发者来说，只需简单的定义CRD扩展资源，以及针对CRD操作的controller代码片段。

`kubebuilder`执行时需要依赖`kustomize`，这两个命令都可以简单的通过命令安装（macOS）：
```
brew install kustomize
brew install kubebuilder
```

## 初始化工程
初始化之前，由于众所周知的原因，可以参照(https://goproxy.io/zh/)[goproxy网站]，设置下代理：

```
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.io,direct
```

设置之后，在工作目录中创建一个空的文件夹，然后用`kubebuilder`初始化工程：
```
kubebuilder init --domain helloworld
```

该命令会初始化一堆东西，具体生成的代码和配置文件还没去详细了解。。。

## 编码
首先，需要新增一个接口：
```
kubebuilder create api --group web --version v1 --kind Blog
```

该命令输入之后，会提示是否需要创建resource以及controller，都选择是即可。

创建完成之后，会生成对应的类型（位于 api/$VERSION 目录下）以及controller（位于 controllers 目录下），主要就修改这两个文件。

### 修改CRD支持的Spec
实际CRD提交时的yaml配置，需要和类型对象关联起来。因此yaml中扩展字段都需要在类型中声明:
```go
type BlogSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	Url string `json:"url,omitempty"`
}
```

对于Spec字段，需要修改对应Kind的Spec结构体。同时还有一个Status结构体，用于保存CRD状态，不过在简单的示例中暂时不需要。
这里需要注意，每个字段都需要加上json标记。同时注释里面已经写明了，修改该文件之后，需要重新执行`make`。`kubebuiler`会自动生成类型对应的deepcopy和其他依赖的东西。

### controller实现
CR对象提交之后不会创建任务资源，它更像是一个有状态的任务，需要一个controller来监听任务的事件，进行实际的资源操作。

`kubebuilder`生成的框架，这些逻辑主要写在`Reconcile`函数中。对于这个简单的Blog kind，controller实现的功能是针对每个Blog实例，创建一个nginx镜像的pod，同时创建一个lb的service对外提供访问支持。

（下面代码忽略各种异常处理，完整代码见[仓库](https://github.com/babydragon/crd-demo)）

首先是获取当前Blog实例：

```go
ctx := context.Background()
log := r.Log.WithValues("blog", req.NamespacedName)

blog := &webv1.Blog{}
err := r.Get(ctx, req.NamespacedName, blog)

...

url := blog.Spec.Url
log.Info("get blog success", "url", url)

label := map[string]string{
    "server": blog.Name,
}
```

然后检查和创建对应的deployment：
```go
deployment := &appsv1.Deployment{}
err = r.Get(ctx, types.NamespacedName{Name: deployment.Name, Namespace: deployment.Namespace}, deployment)
if err != nil && errors.IsNotFound(err) {
    log.Info("deployment not found, start to create")

    newDeployment, err := r.newDeployment(blog, label)

    ...

    err = r.Create(ctx, newDeployment)
    
    ...

    log.Info("create deployment success")
}
```

最后是创建对应的service：
```go
service := &v1.Service{}
err = r.Get(ctx, types.NamespacedName{
    Namespace: service.Namespace,
    Name:      service.Name,
}, service)

if err != nil && errors.IsNotFound(err) {
    log.Info("service not found, start to create")
    newService, err := r.newService(blog, label)
    
    ...

    err = r.Create(ctx, newService)
    
    ...

    log.Info("create service success")
}
```

初始化deployment和service的详细代码见[仓库](https://github.com/babydragon/crd-demo)

## 运行
运行之前需要编译和安装（将CRD apply到k8s上）：

```
make && make install && make run
```

当然，执行之前需要确认本地minikube环境已经正常启动。

执行之后，控制台会打印出controller初始化的消息。同时，可以通过k8s的命令行工具确认CRD已经正确安装：

```
kubectl get crd
```

上述命令会输出目前k8s上的CRD：
```
NAME                   CREATED AT
blogs.web.helloworld   2020-07-06T07:32:02Z
```

最后修改下默认生成的sample，然后创建一个CR试下：
```
apiVersion: web.helloworld/v1
kind: Blog
metadata:
  name: blog-sample
spec:
  # Add fields here
  url: http://a.com
```

```
kubectl apply -f config/samples/web_v1_blog.yaml
```

执行之后，可以在控制台看见controller打印的日志，正常情况下，应该会提示创建deployment和service成功。
同时也可以通过`kubectl`查询对应的deployment、service和pod。在minikube环境下，可通过
```
minikube service list
```
来查看server实际对外暴露的路径，在本地请求可以看见nginx的默认index页面了。

