# 把github action跑在自己的k8s集群中

## 背景
由于众所周知的网络问题，通过github action自带的runner来构建java应用，
特别是推送镜像什么的，很容易遇到网络问题。

另外，由于免费用户的私有仓库每个月运行时间有限制，虽然目前我们还没能到到这个配额，
但目前还只是做了基础的构建，并没有铺开进行持续集成、静态代码检测等任务。

因此，决定自己在k8s集群中部署self host runner,让action执行到内部集群中。
这样有几个好处：

1. 不用再关注执行配额
2. runner镜像可以自定义，可以安装一些默认软件，简化action配置。当然，这样也会减少action的通用性。
3. 运行在集群内，可以方便的使用内部网络访问nexus等基础服务，也可以在集群中直接进行部署操作。

当然，也会有一些缺陷，也是整个配置过程中遇到的问题：
1. 由于网络问题，checkout代码最好使用ssh方式，不然容易不稳定。
2. 也是由于网络问题，不建议直接使用github action marketplace里面提供的actions。
3. 不再建议使用action自带的cache，由于cache是需要上传到github的，本地集群运行之后，速度会很慢。

## 安装actions-runner-controller(arc)
和其他在k8s集群中安装应用相同，直接使用`helm`来进行安装。

```bash
helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller
```

创建一个values.yaml文件，用于相关配置：
```yaml
authSecret:
  create: true
  github_token: xxx
image:
  repository: registry-vpc.cn-shanghai.aliyuncs.com/hyperpaas-tools/actions-runner-controller
  actionsRunnerRepositoryAndTag: registry-vpc.cn-shanghai.aliyuncs.com/hyperpaas-tools/actions-runner:1.0.0
  dindSidecarRepositoryAndTag: registry-vpc.cn-shanghai.aliyuncs.com/hyperpaas-tools/docker:dind
  actionsRunnerImagePullSecrets:
    - aliyun
  tag: v0.27.4
imagePullSecrets:
  - name: aliyun
```

说明：
1. github_token需要在github上生成PAT,具体需要的PAT的权限可以参考：https://github.com/actions/actions-runner-controller/blob/master/docs/authenticating-to-the-github-api.md#deploying-using-pat-authentication
2. image.repository和image.tag：用于指定controller的镜像，这里对镜像本身没有修改，只是tag到了阿里云的容器服务，方便拉取。
3. image.actionsRunnerRepositoryAndTag：用于指定controller创建出来的runner使用的镜像，这里我使用的是自定义定向，基于默认提供的镜像，安装了一些基础软件和进行了基础配置。
4. image.dindSidecarRepositoryAndTag：runner pod中，除了运行runner的容器，还有一个运行dockerd的容器，用于进行docker操作。包括拉取基于镜像的action，我用来构建我们自己的业务镜像。同样没有对镜像进行修改，只是tag了下，方便拉取。
5. imagePullSecrets：指定controller拉取的secrets.
6. image.actionsRunnerImagePullSecrets：指定runner pod镜像拉取的secrets。

安装进行安装：
```bash
helm upgrade --install --namespace actions-runner-system \
  --create-namespace --wait actions-runner-controller \
  actions-runner-controller/actions-runner-controller -f values.yaml
```

安装完成之后，需要提交一个CR来创建runner：
```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: aliyun-runner
  namespace: actions-runner-system
spec:
  replicas: 1
  template:
    spec:
      serviceAccountName: actions-runner
      organization: hyperpaas
      labels:
        - aliyun
```

说明：
1. 由于目前没有太大压力，所以用了RunnerDeployment，同时replicas设置成了1。arc支持RunnerDeployment自动扩所容，后续有需要可以配置。
2. metadata.name：注意这里只是CR的名字，不是runner的名字，实际注册到github上的runner名字，是最终创建的pod的名字。
3. spec.template.spec.serviceAccountName：这里需要先创建这个service account,用于后续在runner中直接操作集群。默认如果没有配置，用的是default。如果没有在runner中操作集群资源的需求，这里可以不设置。
4. spec.template.spec.organization：这里将runner注册到github对应的组织中。arc支持注册到仓库、组织等多个层级。
5. spec.template.spec.label：注册到github上的label。github action在挑选runner的时候是按照标签而不是名字，因为名字是pod的名称，每次都会变动。同时，github也会自动加上self-hosted标签，以及诸如操作系统、CPU架构等标签。如果只有一个自定义runner可以不设置，直接让action运行在self-hosted标签上即可。

创建CR之后，controller会创建对应的deployment（如果是RunnerSet就是sts），然后运行对应的pod。
pod使用的镜像，就是前面在安装controller的时候指定的镜像。

运行成功之后，可以在仓库的设置页面（settings -> actions -> runners）中看见注册上来的runner。

## 自定义runner镜像
由于runner不在github,会有很多网络问题，可以通过自定义runner镜像，进行预先安装，来尽可能规避这些问题。

```Dockerfile
FROM summerwind/actions-runner:latest

USER root
#remove old apt sources
RUN rm -rf /etc/apt/sources.list.d/* && rm -rf /etc/apt/sources.list

# add new apt sources
RUN echo "deb https://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse" >> /etc/apt/sources.list && \
    echo "deb https://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse" >> /etc/apt/sources.list && \
    echo "deb https://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse" >> /etc/apt/sources.list && \
    echo "deb https://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse" >> /etc/apt/sources.list && \
    echo "deb https://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse" >> /etc/apt/sources.list

# install pkg
RUN apt update && apt install -y openjdk-17-jdk-headless maven

# clean
RUN apt clean && rm -rf /var/lib/apt/lists/*

USER runner
ADD ssh /home/runner/.ssh
ADD settings.xml /home/runner/settings.xml
RUN sudo chown -R runner:runner /home/runner/.ssh && sudo chmod 600 /home/runner/.ssh/id_rsa
```

说明：
1. 使用基础镜像为`summerwind/actions-runner`，也就是默认的runner镜像。这个镜像是基于ubuntu 22.04。
2. 为了软件安装速度，将软件源改成aliyun
3. 内置安装jdk和maven,这样不用在action里面设置setup java什么的。因为这个action也会在github上下载。导致运行速度很慢。缺点就是无法支持多版本。（目前业务不需要，如果需要，可以考虑新建runner通过label隔离不同版本运行环境，或者选择本地安装jdk）
4. 内置maven settings和ssh key。因为都是自己管理，就不再action上通过security来设置了。

## 通过k8s的service account来控制runner可以操作的资源
基于CD的需求，action需要完成构建镜像+更新pod镜像的操作。默认情况下，runner使用了default帐号，
该帐号没有操作其他k8s资源的权限。

这里我们需求是：通过action来更新部署在test ns中的deployment镜像。

因此，先创建一个新的sa（service account）
```bash
kubectl create serviceaccount actions-runner --namespace actions-runner-system
```

然后，创建一个`rolebinding`把刚创建的sa关联到cluster role的edit角色上，并限定在test ns中。
这里没有详细列举可以操作的对象和赋予的权限，而是直接用了自带的edit角色。但是为了安全，通过ns来限制操作范围。

```bash
k create rolebinding actions-runner-admin \
  --clusterrole edit --serviceaccount actions-runner-system:actions-runner \
  --namespace test
```
说明：
1. --serviceaccount指定的sa,需要用冒号来限定sa所在的ns。
2. --namespace指定的是可以操作的ns

设置完成之后，前一个步骤中提交的CR里面，通过设置serviceAccountName来限制runner中进程可以操作的k8s集群中的资源。

## 一个示例action
```yaml
name: Aliyun build and push

# 支持手工触发、push到main触发和push tag触发
on:
  workflow_dispatch:
  push:
    tags:
      - '*'
    branches:
      - main
env:
  APP: app1

jobs:
  build-and-push:
    runs-on: [self-hosted, aliyun] # 使用自定义runner
    outputs:
      image_name: ${{steps.build-image.outputs.IMAGE_NAME}} # 输出镜像名称
    steps:
    - name: Setup oss
      env:
        ak: ${{secrets.accessKeyId}}
        sk: ${{secrets.accessKeySecret}}
      run: |
        wget -q https://gosspublic.alicdn.com/ossutil/1.7.13/ossutil64
        chmod +x ossutil64
        mkdir $HOME/bin && mv ossutil64 $HOME/bin/
        echo "$HOME/bin" >> $GITHUB_PATH
        $HOME/bin/ossutil64 config -e xxx -i ${ak} -k ${sk} -L CH
    - name: restore maven cache
      run: |
        mkdir -p $HOME/.m2/
        ossutil64 cp oss://xxx/maven-cache.tar . && tar xf maven-cache.tar -C $HOME/.m2/ || echo "cache not exist"
    - name: clone
      run: git clone git@github.com:${GITHUB_REPOSITORY}.git -b ${GITHUB_REF_NAME}
    - name: git version
      id: git_version
      working-directory: ${{env.APP}}
      run: echo "REVISION=$(git describe HEAD)" >> $GITHUB_OUTPUT
    - name: Build with Maven
      id: build
      working-directory: ${{env.APP}}
      run: mvn -B install --file pom.xml -s $HOME/settings.xml -DskipTests -U
    - name: Build Docker image
      id: build-image
      working-directory: ${{env.APP}}
      run: |
        IMAGE_NAME="REPO/hyperpaas/${APP}:${{steps.git_version.outputs.REVISION}}"
        docker build -f Dockerfile_without_build . -t ${IMAGE_NAME}
        echo "IMAGE_NAME=$IMAGE_NAME" >> $GITHUB_OUTPUT
    - name: Push image
      run: |
        docker push ${{steps.build-image.outputs.IMAGE_NAME}}
    - name: Save cache
      run: |
        tar cf maven-cache.tar -C $HOME/.m2/ repository
        ossutil64 cp maven-cache.tar oss://xxx/maven-cache.tar

  deploy:
    needs: build-and-push
    if: (github.event_name == 'push' && github.ref == 'refs/heads/main') || github.event_name == 'workflow_dispatch'
    runs-on: [self-hosted, aliyun]
    steps:
    - name: Setup kubectl
      run: |
        curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
        chmod +x kubectl
        mkdir $HOME/bin && mv kubectl $HOME/bin/
        echo "$HOME/bin" >> $GITHUB_PATH
    - name: Replace image
      run: |
        echo "update image to ${{needs.build-and-push.outputs.image_name}}"
        kubectl set image deployment/${APP} ${APP}=${{needs.build-and-push.outputs.image_name}} -n test
    - name: Wait for rollout
      run: |
        kubectl rollout status deployment/${APP} -n test --timeout=120s
```

说明：
1. Setup oss：由于不能使用action自带的cache（需要往github上传下载，速度更慢了），这里直接把构建缓存打包放到阿里云的oss上（k8s使用了阿里云ack,因此这里可以直接使用oss vpc内部节点，避免网络费用）。
2. restore maven cache：从oss下载之前maven构建出来的repository目录，用于加速后续构建。注意这里需要兼容不存在的情况。因为要实现github cache的查找逻辑用ossutil命令比较困难（cache key和restore key查询匹配，按照时间排序等），这里暂时只用了一个大文件，没有对每次构建单独缓存。
3. clone：前文提到过，由于众所周知的原因，用github action提供的checkout，默认使用的是https方式clone,经常会因为网络问题导致clone很慢甚至异常。为了方便，我直接将git clone时候的ssh key打到了镜像里面。
4. Build with Maven：由于用了缓存，这里构建需要加上-U来强制升级snapshot。另外，也是为了配置方便，直接把settings放到了镜像里面，可以直接使用。之前是用了action来生成settings,比较麻烦。
5. Save cache：将构建完成的m2 repository目录打包并上传到oss上，加快下次构建速度。
6. deploy job,配置了仅在push main和手工触发的时候构建，避免push tags的时候重新部署。
7. 部署流程是下载安装kubectl,然后用kubectl命令替换deployment的镜像。替换完成之后等待deployment更新完成。由于pod已经关联了service account,因此在操作当前集群的时候，无需再配置kube config。

## 之后的改进
以上就完成了一个github action的runner配置，以及运行在这个runner上的action配置。

后续使用规模上来之后，还可以继续改进：

1. 支持前端代码的测试、构建、发布;
2. 支持动态增加runner节点;
3. 优化缓存方式，避免并行执行的时候缓存文件覆盖。