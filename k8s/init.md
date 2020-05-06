# 从0开始学kubernetes——基础环境安装

由于工作原因，很多内部工具没有Linux版本，目前已经将工作系统从Linux换成了Mac，因此后面介绍的都是在Mac系统上的。

## 使用homebrew安装kubectl
kubectl是和k8s集群交互的入口，感觉从systemd被大多数Linux发行版接受之后，好多管理类命令的命名，都变成了xxxctl。

使用homebrew安装包很方便，直接通过命令`brew install kubectl`即可。

唯一需要注意的是，在安装homebrew的时候，都应该修改了镜像，否则homebrew更新和下载速度会非常慢。同样，我也换了镜像。参照之前使用Gentoo的经验，通常我会优先选择的镜像站有：阿里云、中科大和清华。

由于速度优势，大部分情况下，我会优先尝试阿里云的镜像。但是对于homebrew来说，可能会遇到问题。阿里云的镜像本身更新未发现什么问题，但是在使用`brew install`命令安装的时候，会发现好几次都无法在bottle仓库中找到预编译好的二进制版。大部分情况下，二进制包找不到，回退到使用源码编译很正常（特别是对于我这个习惯使用Gentoo的人），但是Golang还真是个特例，下载速度太慢了。。。

所以，这里需要提醒的是，还是建议把bottle镜像改成清华，这样下面的minikube、kubectl和golang都能够正确下载到二进制包。

## 安装minikube
为了在本地能够启动k8s，最方便的应该就是minikube了。参照minikube的[安装文档](https://kubernetes.io/docs/tasks/tools/install-minikube/#before-you-begin)，除了安装minikube本身之外，还需要安装一个虚拟机。当然，最方便和最熟悉的当然是Virtualbox。

homebrew有一个叫[cask](https://github.com/Homebrew/homebrew-cask)的组件，可以用来安装二进制应用程序，避免单调的拖拽（不过拖拽相比Windows软件传统的下一步，要好很多了）。

安装完cask并切换镜像之后，可以直接安装virtualbox：
```
brew cask install virtualbox
```

**注意**：由于virtualbox安装需要系统权限，所以第一次安装会失败。失败之后，需要到设置->安全与隐私中允许一下，再重新执行即可。

然后就是安装minikube：
```
brew install minikube
```

安装完成之后，可以直接使用。

```
minikube start
```

第一次运行比较慢，minikube检测到virtualbox之后，需要去下载对应的iso镜像文件和docker文件系统什么的，最后一次性将虚拟机都设置好。后续启动就比较快了，直接复用这个虚拟机就好了。

## 验证
首先开启minikube:
```
minikube start
```

启动之后，可以通过
```
minikube dashboard
```
命令，通过浏览器访问dashboard页面。

同时，kubectl命令也同样可以直接操作minikube了：
```
% kubectl version
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.1", GitCommit:"7879fc12a63337efff607952a323df90cdc7a335", GitTreeState:"clean", BuildDate:"2020-04-10T21:53:51Z", GoVersion:"go1.14.2", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.0", GitCommit:"9e991415386e4cf155a24b1da15becaa390438d8", GitTreeState:"clean", BuildDate:"2020-03-25T14:50:46Z", GoVersion:"go1.13.8", Compiler:"gc", Platform:"linux/amd64"}
```
可以看见服务端和客户端的版本信息。

最后别忘了关闭minikube

```
minikube stop
```