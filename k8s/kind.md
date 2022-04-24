# 通过Kind来了解k8s结构

## 简介和安装
kind是将k8s部署在本地docker环境中的工具，本身仅包含一个命令行工具，同时有多个CPU架构下的node镜像，可以直接在本机docker环境中拉起一个k8s集群。
目前kind依赖docker环境，按照它的[文档](https://kind.sigs.k8s.io/docs/design/principles/#target-cri-functionality)，后续会和docker解耦，只需要一个CRI就可以了。因此还是一个比较轻量化的工具，可以通过自动化搭建简单的k8s集群，并在此之上运行集成测试。

kind安装比较方便，可以借助于包管理工具，也有预编译的二进制文件：https://kind.sigs.k8s.io/docs/user/quick-start/#installing-with-a-package-manager 。对于mac来说，直接用homebrew安装即可：
```
brew install kind
```

## 创建集群
kind为所有版本的k8s都做了节点镜像，因此可以通过指定节点镜像版本的方式，来指定实际创建的k8s版本。在创建之前，可以在[docker hub](https://hub.docker.com/r/kindest/node/tags)上搜索下对应的tag，
以确认k8s集群的版本。

然后，就可以使用`kind`命令来创建集群了：

```
kind create cluster --name CLUST_NAME --image IMAGE_NAME
```

注意：
* 节点镜像都比较大（普遍在400多M），直接从docker hub拉取速度比较慢
* 对于M1（ARM）架构电脑，在选择镜像版本的时候需要注意只有比较新的版本支持ARM64

完成集群创建之后，本地就可以通过`kubectl`命令来直接和集群进行交互了。

## 节点镜像内容
既然kind能够直接拉起一个k8s集群，我们可以通过进入节点镜像来看下里面到底包含了什么。

```
systemd-+-containerd---15*[{containerd}]
        |-containerd-shim-+-kube-scheduler---8*[{kube-scheduler}]
        |                 |-pause
        |                 `-11*[{containerd-shim}]
        |-containerd-shim-+-kube-controller---6*[{kube-controller}]
        |                 |-pause
        |                 `-11*[{containerd-shim}]
        |-containerd-shim-+-kube-apiserver---9*[{kube-apiserver}]
        |                 |-pause
        |                 `-11*[{containerd-shim}]
        |-containerd-shim-+-etcd---14*[{etcd}]
        |                 |-pause
        |                 `-11*[{containerd-shim}]
        |-containerd-shim-+-kube-proxy---7*[{kube-proxy}]
        |                 |-pause
        |                 `-11*[{containerd-shim}]
        |-containerd-shim-+-coredns---8*[{coredns}]
        |                 |-pause
        |                 `-12*[{containerd-shim}]
        |-containerd-shim-+-coredns---9*[{coredns}]
        |                 |-pause
        |                 `-11*[{containerd-shim}]
        |-containerd-shim-+-local-path-prov---12*[{local-path-prov}]
        |                 |-pause
        |                 `-11*[{containerd-shim}]
        |-containerd-shim-+-kindnetd---6*[{kindnetd}]
        |                 |-pause
        |                 `-11*[{containerd-shim}]
        |-kubelet---18*[{kubelet}]
        `-systemd-journal
```

通过容器内的进程树可以看见主要的进程：

* kube-scheduler：k8s默认调度器，负责将 Pods 指派到节点上
* kube-controller：默认控制器进程
* kube-apiserver：api服务，对外提供标准API，所有其他组件都通过它来交互
* etcd：kv存储
* kube-proxy：节点上运行的网络代理
* kubelet：节点上负责Pod中容器运行的代理
* coredns:dns解析服务
* [kindnetd](https://github.com/aojea/kindnet)：CNI插件，为集群网络提供双栈网络支持
* [local-path-prov](https://github.com/rancher/local-path-provisioner)：让节点可以使用本地存储

除了最后两个算是给kind定制的插件，其他都是k8s标准组件（或插件），[官方文档](https://kubernetes.io/zh/docs/concepts/overview/components/)给了一个架构图：
![架构图](https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg)