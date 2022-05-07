# 《Kubernetes in Action》读书笔记——k8s介绍

这是全文写完之后再写的概述，感觉本文基本上和这本书没什么关系了，主要记录下技术变革中参与和体会到的内容。

## 架构演进

### 业务应用
业务应用经历了从单体应用到微服务的转变。当年也参与过项目，将一个大型的应用拆分成多个基于springboot架构的小型应用。这个方向的转变，来自业务的压力，互联网公司总是面临着各种快速迭代的需求，个人感觉传统应用最先受到冲击的，是整个研发流程。由于需求多且变化快，对于整个研发流程影响有：

1. 应用分支多：不管是之前基于SVN还是现在的git，由于需求多样，每个需求都会有自己的开发分支。
2. 由于1导致了冲突的增加。大家都在并发的改，总是不可避免的产生各种代码冲突，如果要发布，就必须解决这些冲突，就个人感觉，这个工作量也不小，特别项目进入回归测试阶段。
3. 测试回归成本变高。由于并行需求和代码修改的增加，对于测试来说，如何回归和何时进行回归是个需要在项目初期就要慎重考虑的点。回归晚了，其他项目可能会影响当前项目的功能，在后期发现这种bug，是非常有可能影响整个项目进度的；回归早了，分支合并和冲突解决的成本会增加。特别是现在开发测试比越来越高，回归测试重要性也越来越大。
4. 应用发布策略影响。我刚到b2b的时候，严格规定了每周发布项目和小需求的窗口，导致的问题就是发布小需求的时候，每天光合并分支就要一上午时间，如果有需求退出，又要重新合并。印象最深的一次是过年前最后一个发布日，从早上开始合并代码，发布到预发环境，一直到晚上10点多还是11点多了，才通知可以进行预发验证。

然后，业界开始推广微服务了，特别是springboot的推出，可以通过一个jar包就能“快速”启动一个应用，是多么美妙的事情啊。微服务的“微”能解决很多事情：

* 应用职责单一，除了让开发上手更加方便，最关键的能够更加方便的解决各种jar包冲突的问题。
* 功能独立，对于测试来说，也更容易沉淀各种回归用例。另外由于微服务大多会通过RPC对外提供服务，相对来说也简化了部分测试，测试工具能够解决很多问题。
* 容量规划更加容易，一是因为功能单一，容易发现性能瓶颈所在。另外，扩容也会比较容易。

当然，微服务也会带来一些问题，产生一些奇怪的设计：

* 服务的增加，不可避免的增加了服务治理的难度。这个对于生产环境来说影响相对会小一些，因为虽然服务众多，依赖复杂，但是稳定的版本就这么几个。但是在测试环境就没有这么幸运了，测试环境是一个相对自由开放的环境，对于同一应用，也会运行着各种不同版本的代码。测试环境服务提供者的利己性，没有人会考虑自己提供服务的稳定性和可靠性，也不会关注自己的服务究竟会被谁调用。相对的，作为服务使用者，又都在抱怨别人提供的服务有多么的不稳定。
* 简单的拆分，实际大大增加了资源开销。由于有容灾、发布等的要求，一个再简单的微服务，在生产环境也起码得部署2份。实际这些微服务使用的资源可能很小，但是出于两个目的，一是大部分情况下，开发不会去修改应用的默认启动脚本，因此系统资源，特别是内存会有比较大的浪费，二是作为开发很少会关注机器本身的开销（虽然现在开始各种清算团队的成本）。解决这个问题，从运维端出发，最容易的就是增加机器超卖，但是这也增加了应用不稳定的风险，谁知道进程实际使用的内存到底有多少呢？从开发架构师出发，出现了各种合并部署的架构，将多个微服务部署在一个应用里面，但是从应用的角度来看，前面提到的好几个优势就都不存在了。

### DevOps
微服务的兴起，还带来了一个新的名词：DevOps。这个词其实挺让人迷惑的，到底是让Dev来做Ops，还是让Ops去做Dev呢？好吧，我都遇到过。

微服务和容器化的兴起，让开发能够控制生产环境了：Dockerfile。通过它，能够控制机器上运行什么版本软件，能够控制应用怎么启动，启动前后需要的操作等等。

但是，就我个人感觉，现实和理想差距还挺远。首先从分工上来说，基础镜像还是由原先的运维团队维护。当然了，不然我怎么知道机器上还要安装日志收集的组件，机器管理的agent呢？这些看上去和开发并没有多大关系。其次，从技能上来看，我还遇到过好多分不清在Dockerfile中写RUN指令和执行一个sh脚本区别的开发，更别说去RPM仓库找对应版本的rpm包安装了，好吧，很多人连基础镜像版本具体有什么不同也不知道。

那么Ops去做Dev呢？之前团队试图和运维一起做一个适用于内部的运维平台，主要目的就是能够开发出更多日常运维操作给开发，同时确保开发不会因为这些操作导致故障。结果呢？运维同学更多的承担了需求方和专家的角色，为系统注入了运维经验，我们在每一个线上操作之前加了一系列预检，确保了这些操作不会有故障产生，同时开发同学能够自主解决中间过程遇到的大部分问题。但是这几个运维同学加入到开发里来还是很困难的，一是因为编程语言，避免对于运维写的更多的是shell或者Python，对于Java上手还会快一些，前端框架（当时用的是antd）就太难为他们了。还有就是时间，一个网站只有一个业务运维同学，还是有大量申请审批和问题处理需要人肉参与，导致他们的时间极度碎片化。

感觉DevOps前几年挺火的“全栈”一样，毕竟术业有专攻，职责划分明确是更重要的事情。毕竟，事实上除了业务运维之外，更下面还有IDC、网工等个多参与运维工作的同学，这些对于业务开发来说接触的就更少了，真要遇到问题，也很难独自解决。

## kubernetes解决什么
好吧，前面扯了很多，实际上和这本书基本上没有半毛钱关系，更多的是我遇到的情况。说回k8s，在使用k8s之前，也是好几年前了，k8s刚出来的时候，还没开始关注，自己在几台服务器上搭建了mesos集群，通过Marathon来部署Docker化的Java应用。现在mesosphere公司都已经开始支持k8s了。

看了本书的第一章，k8s实现的基本功能和mesos类似，现在被定义为云操作系统，将分散的运算资源通过统一的API暴露出来，用户无需关注资源细节即可将任务提交并执行。不过感觉k8s的抽象更类似于操作系统，它的pod就像操作系统中的进程一样，独立运行，相对隔离，只有通过特定的通信方式才能与其通信。

使用k8s的优势：

* 简化部署：只需要简单配置，就能让应用在适合运行的节点上运行，无需关注其中的细节。
* 健康检查：监控应用健康状态，维持其正常状态的副本数量，做到自动故障转移和恢复。
* 进一步划分Dev、Ops职责边界：理论上应用的整个生命周期都可以由开发控制，从应用创建上线资源申请，到最终下线资源回收，包括中间的日常发布和扩缩容，通过k8s的管理，都不需要Ops参与。而Ops只需要制定最基础的规则和处理故障节点即可。