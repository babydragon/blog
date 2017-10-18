# 离线平台学习

## husky框架
第一次了解大数据相关的框架，虽然是一个标准的maven工程，但是拉下代码之后，有点无从下手：

* 实际java代码量不大，但是配置文件巨多无比
* 传输序列化使用了protobuf，直接导入工程之后很多类会各种红线

关于husky，首先先学习了下[husky使用手册](http://twiki.corp.taobao.com/bin/view/Taobao_Algo/HuskyODPS)了解下大致配置结构和入口：工程核心在flow和node的配置上。

flow是基于xml定义的每个节点，每个节点会在task目录中有对应的xml文件来描述其运行。flow文件中，通过node来组装所有节点任务，node之间可以通过dependencies标签来设置依赖关系。

flow中的每个节点名称，具体的定义都在task目录中的对应名称xml文件中。这些文件除了任务基本描述信息之外还会有任务的执行描述。这里的任务大致分为两种：sql任务和map reduce任务。

sql任务会指定对应的sql文件，其中的sql语句会在odps上运行，通常是将原有的表进行整合，放到一个中间结果表中。map reduce任务配置相对比较复杂，特别需要关心的是```husky.inputs```、```husky.inputs.xxx.table```、```husky.inputs.xxx.mapper```、```husky.output.table```。

首先是输入配置```husky.inputs```，其值为输入的名称，后面几个输入相关的内容都以此为前缀（既xxx都以次值替换）。输入的来源，可以额从```husky.inputs.xxx.table```中获取，既其上游数据来源的表。对数据的处理，通过```husky.inputs.xxx.mapper```、```odps.mapred.reduce.class```类指定对应的mapper和reduce类。最后的输出，需要通过```husky.output.table```来指定输出的表。

## sql
sql类的数据处理，可以在D2上直接进行调试。首先需要[申请加入项目](http://tenant.alibaba-inc.com/layout/project.html)，然后可以在[d2平台](http://d2.alibaba-inc.com/)上进入项目进行sql编写和执行。创建文件之前，需要在规定目录中创建自己的目录，然后创建sql文件，即可执行sql了。

## map reduce
这类数据处理需要通过编码的方式进行。husky框架封装了对应的基础类。

* Mapper<KEYIN, VALUEIN, KEYOUT, VALUEOUT>：所有mapper类都需要继承该类，实现其中的map方法。
* Reducer<KEYIN, VALUEIN, KEYOUT, VALUEOUT>：所有reduce类都需要集成该类，实现其中的reduce方法。

两个类都有setup和cleanup方法，用于配置初始化和清理工作。

处理结果只需要调用context的write方法写入即可。
