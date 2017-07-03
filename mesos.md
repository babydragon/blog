# mesos + marathon集群搭建

## 机器准备
机器准备和mesos本身没有关系，主要由于本身机器申请机制有关，这段主要记录下如何通过配置systemd，将agent启动并且正确注册到服务器上。
这里的核心问题是需要开机获取本机IP，然后按照顺序启动docker和agent的镜像。

由于docker0设备的存在，之前agent获取本机IP的方式会出现问题，所有基于docker启动的agent都需要通过环境变量传入本机IP。因此首先需要在镜像启动前，获取本机IP。systemd本身可以通过```%H```来指代主机名，但是没有提供本机IP的变量。这边通过开机自动运行脚本，按照k=v的方式写入文件，提供给其他unit file作为environment file使用。

```
#cat ip.service
[Unit]
Description=get host ip and write to /etc/ip.txt

[Service]
ExecStart=/etc/host-ip.sh

[Install]
WantedBy=multi-user.target
```
对应的脚本文件内容为：
```bash
#cat /etc/host-ip.sh
#!/bin/sh
ip=$(/sbin/ifconfig eth0 | awk '/inet/ {print $2}')
echo IP=$ip > /etc/ip.txt;
```

这样每次服务启动之后，会写入到ip.txt文件，其内容类似：
```
IP=1.1.1.1
```

将unit文件放置在```/etc/systemd/system```目录中，别忘了设置成自动启动：
```bash
systemctl enable ip.service
```

搞定机器IP之后，就是agent的启动了。首先需要确保的是docker服务本身已经配置成自动启动。
```bash
systemctl enable docker
```
然后创建agent对应的unit文件：
```
#cat agent.service
[Unit]
Description=aenv agent
Requires=docker.service ip.service
After=docker.service ip.service

[Service]
EnvironmentFile=/etc/ip.txt
Restart=on-failure
RestartSec=10
ExecStartPre=-/usr/bin/docker kill agent
ExecStartPre=-/usr/bin/docker rm agent
ExecStart=/usr/bin/docker run --net host --env HOST_IP=$IP --name agent agent:1.0
ExecStop=-/usr/bin/docker stop agent

[Install]
WantedBy=multi-user.target
```

这里需要注意几个地方：

* 依赖关系：在unit部分中，必须指定当前配置文件的依赖关系，这里指定依赖docker和ip两个服务，同时指定当前服务在这两个服务启动之后再启动。
* 环境变量文件：之前的ip服务已经自动将ip地址写入到了ip.txt文件中，unit文件中可以直接将该文件设置为EnvironmentFile。这样可以通过引用文件中的KEY来获取实际的数据。
* 启动命令：这里指定了几个阶段的命令，分别是预启动命令，如果当前已经有agent容器，直接杀死并删除；启动命令，设置agent容器启动命令，特别注意设置I变量；关闭命令，关闭服务时执行的命令，直接关闭agent容器。

最后别忘了设置开机自动启动：
```bash
systemctl enable agent.service
```

## mesos+marathon集群搭建
mesos和marathon的基本安装可以参照mesosphere的[文档](https://open.mesosphere.com/getting-started/install/)进行。这次尝试安装使用的操作系统镜像是Centos 7,因此只要按照文中关于Centos 7的步骤即可。由于之前已经完成了zk集群的搭建，这里不再需要安装zk。

为了能够组建高可用集群，mesos master和marathon都使用3台机器搭建。因此需要修改部分配置文件。

对于mesos master，首先修改/etc/mesos中的zk文件，里面修改成已经搭建完成的zk集群，多个地址之前用逗号分割，mesos master之间通过zk来相互感知和进行选举。其他需要修改的文件都在/etc/mesos-master目录中，对于集群来说，首要设置的是quorum文件。按照官方文档说明，对于3台master，quorum需要设置为2,以进行单点失效后的选举。按照文档，这两个文件设置完成之后，就可以启动三台mesos master机器上的mesos-master服务了：
```bash
systemctl start mesos-master
```
但是，此时当三台机器都启动完成之后，访问任意一台的5050端口，会出现两个问题：

1. 当访问机器不是当前leader时，网页无法正确跳转，直接跳转到了服务器的hostname（目前主机名只能通过机房DNS才能解析）;
2. 跳转到leader机器之后，过几秒钟就会跳转到其他机器，查看日志发现mesos master在不停的失效，不停的选举；

对于问题1,由于mesos master默认获取的主机名在非机房DNS中都无法解析，最方便的做法就是将hostname直接设置成机器IP。既在/etc/mesos-master目录中创建文件hostname，并将其内容设置为机器IP。

对于问题2,查了很多文档，也尝试了很多方式，最终尝试给mesos-master指定ip为当前机器IP后解决。既在/etc/mesos-master目录中创建文件ip，并将其内容设置为机器IP。这里需要特别注意的是，mesos master会将内容缓存在work_dir中，其值通过/etc/mesos-master目录中的work_dir文件内容指定。通过mesosphere rpm包安装之后默认值为/var/lib/mesos。因此修改了ip文件之后，需要清空其中内容，才会生效。

完成上面两步之后，mesos master集群启动成功。之后可以将mesos-master服务设置为开机自动启动：
```bash
systemctl enable mesos-master
```

然后就是安装marathon集群，还是直接使用rpm包进行安装。其中只有一个需要注意的地方（另一个大坑后面会介绍），就是设置机器的hostname为IP，原因前面已经介绍过。marathon的配置在/etc/marathon/conf目录中，对于主机名，创建hostname文件，并在其中设置当前机器IP即可。marathon启动之后，默认监听8080端口。

最后将marathon服务设置为开机自动启动：
```bash
systemctl enable marathon
```

## 添加slave和执行docker任务
前面已经介绍了mesos master集群和marathon集群的搭建，要真正能够分配资源并运行任务，还需要向master注册slave。slave和master使用了相同的机器镜像，确保rpm包已经正确安装，同时zk都已经完成配置（slave也使用/etc/mesos/zk的配置）。对于slave的所有配置，都在/etc/mesos-slave目录中，配置方式和mesos相同，防止和参数名称相同的文件即可。最简单的配置和master类似，设置hostname即可。既创建/etc/mesos-slave/hostname文件，并写入当前机器IP。其他服务器可用资源可以让slave自动检测并上报。

设置完成之后，就可以启动slave了。可以在master上看见注册上来的slave及其资源信息。同时记得将slave设置为开机自动启动：
```bash
systemctl enable mesos-slave
```

启动之后，尝试在marathon上提交一个docker任务：启动```redis:alpine```镜像。但是发现任务没有能够正确的在slave上启动。参照[marathon文档](https://mesosphere.github.io/marathon/docs/native-docker.html)介绍，要让slave支持docker，需要设置containerizers参数。执行：
```bash
echo 'docker,mesos' > /etc/mesos-slave/containerizers
echo '5mins' > /etc/mesos-slave/executor_registration_timeout
```
后面那个参数主要是由于docker任务启动的时候，需要去远程拉取镜像，一般刚开始容易超时。（虽然这样设置了之后还是会超时，不过由于marathon的重试机制，超时的任务会稍后重新启动，并且docker镜像本地也会缓存，目前感觉影响不大。）

完成这些配置之后，再重启mesos-slave服务，即可在marathon上提交刚才无法成功的redis:alpine镜像了。

另外，marathon还提供了任务的健康检测。按照其文档描述，在mesos master v0.23.1和v0.24.1以后，针对docker任务也已经支持通过命令来进行健康检测（实际查看slave日志后发现，执行的命令最终被转化成了```docker exec```命令，进入容器之后运行指定的命令）。对于redis任务，我们可以配置```redis-cli info```命令来进行健康检测，如果命令执行超时则判定为失败。不过marathon的健康检测还不支持对命令返回值的断言。

## 本地存储卷
前面示例已经启动了redis镜像，然后为了验证marathon的资源隔离（主要是内存）特性，做了以下尝试：

1. 将redis容器的内存限制为32M;
2. 通过脚本循环向redis写入长字符串;
3. 发现当任务资源监控超过16M的时候，内存占用又会重新清零。

这个过程我们可以知道，将数据写入redis之后，任务监控到redis实例的内存占用在不断增加。然后超过内存限制的一半之后，任务被kill，几秒钟之后marathon重新启动了任务，本地redis客户端重新可用。这里内存超过一半是因为默认redis开启了rdb文件保存（bgsave），redis需要新fork出进程进行rdb文件写入，导致此时虚拟内存翻倍，容器占用的内存超过了cgroup限制，被内核oom kill。我们可以通过内核日志发现一次oom kill的日志。最后marathon虽然重新启动了任务，但是由于docker容器的隔离性，之前写入硬盘的内容（rdb）文件都已经消失，导致内存清零，之前写入的大字符串消失。

如果我们希望redis任务重新启动之后能够保留数据（虽然对于测试场景无效，因为内存不足，如果数据恢复，进程也会立即被内核kill），就需要将容器中的redis工作目录持久化保存。

marathon文档中描述了两种持久化的方式，对于我们来说，最简单的就是使用本地卷进行持久化。本地卷的大致方式是向mesos master申请指定大小的卷，并将该卷和任务容器关联起来（对于docker容器使用volume-from参数关联即可）。

启动redis任务，其中和卷相关的配置为：
```json
"volumes": [
     {
       "containerPath": "/data",
       "hostPath": "redis",
       "mode": "RW"
     },
     {
       "containerPath": "redis",
       "mode": "RW",
       "persistent": {
         "size": 1024
       }
     }
    ]
```
该配置表示在slave机器上申请名为redis的卷，大小为1024MB，权限为读写。同时设置将主机的redis卷挂载到容器的/data目录中，挂载权限为读写。

提交任务之后，发现使用了本地卷之后的任务一直在unsheduled状态，slave上没有任务分派的数据。查了半天文档也没有好的解释。于是重新按照[marathon文档](https://mesosphere.github.io/marathon/docs/persistent-volumes.html)进行了设置，特别关注其中的Prerequisites章节。其中描述了marathon启动参数中**必须**设置```mesos_authentication_principal```和```mesos_role```两个参数。但是其中没有太明确描述需要设置什么值。

由于这两个参数主要作为在mesos master执行操作时后的标识，首先先在marathon机器上的/etc/marathon/conf/目录中分别创建mesos_authentication_principal和mesos_role两个文件：
```bash
echo 'marathon' > /etc/marathon/conf/mesos_authentication_principal
echo 'marathon' > /etc/marathon/conf/mesos_role
```
重启marathon服务之后，在任务详情中就可以看见新增的volume标签页。但情况并没有好转，任务长时间处于waiting状态，卷的状态一直为detached。最终在mesos master（leader那台）上找到了一些错误日志：

> Dropping RESERVE offer operation from framework ... A reserve operation was attempted with no principal, but there is a reserved resource in the request with principal 'marathon' set in `ReservationInfo`

紧接着一条错误：

> Dropping CREATE offer operation from framework...Invalid CREATE Operation: Insufficient disk resources

查询了文档之后大概知道，由于principal设置的问题，mesos master没有认可marathon发起的RESERVE指令（既让slave预留一部分资源，这里是硬盘空间），随后由于没有预留足够的硬盘空间，发送的创建指令由于没有剩余空间而失败。

从错误日志基本可以断定是由于principal设置问题导致的，从[marathon任务相关文档](https://mesosphere.github.io/marathon/docs/framework-authentication.html)中，终于了解了mesos master acl相关的配置，，以及这些配置对frameworks的影响。直接抄文档中的acls内容会有问题，因为其中只是指定了run_tasks和register_frameworks两个操作的acl，通过[mesos文档](http://mesos.apache.org/documentation/latest/authorization/)，我们可以查阅到mesos master支持的所有acl项。对于我们的需求，需要授权marathon支持reserve_resources操作，最终将acls参数内容更新为：
```json
{
    "run_tasks": [
        {
            "principals": {
                "type": "ANY"
            },
            "users": {
                "type": "ANY"
            }
        }
    ],
    "register_frameworks": [
        {
            "principals": {
                "type": "ANY"
            },
            "roles": {
                "type": "ANY"
            }
        }
    ],
    "reserve_resources": [
        {
            "principals": {
                "type": "ANY"
            },
            "roles": {
                "type": "ANY"
            },
            "resources": {
                "type": "ANY"
            }
        }
    ],
    "unreserve_resources": [
        {
            "principals": {
                "type": "ANY"
            },
            "roles": {
                "type": "ANY"
            },
            "reserver_principals": {
                "type": "ANY"
            }
        }
    ],
    "create_volumes": [
        {
            "principals": {
                "type": "ANY"
            },
            "roles": {
                "type": "ANY"
            }
        }
    ],
    "destroy_volumes": [
        {
            "principals": {
                "type": "ANY"
            },
            "roles": {
                "type": "ANY"
            },
            "creator_principals": {
                "type": "ANY"
            }
        }
    ]
}
```
这里只是为了能够正常分配，没有做任何访问控制。将上述内容保存到文件（/etc/mesos-acls）然后添加到mesos-master的参数：
```bash
echo 'file:///etc/mesos-acls' > /etc/mesos-master/acls
```

重启所有mesos master，之前一直在waiting状态的任务，已经可以正常申请卷，并启动容器。进入分配到的slave服务器，我们可以看见多了一个挂载点：
```
/dev/vda1 on /var/lib/mesos/slaves/fb9d0623-c0bf-402e-9288-06206b30ea54-S4/frameworks/487f9562-5998-4cfb-bc92-bbc3a441c69c-0000/executors/redis_single.bd21b767-41b8-11e6-9d7a-024259ae7178/runs/8c9304a1-b960-4aa0-a169-f368a4d77ed9/redis type xfs (rw,relatime,attr2,inode64,noquota)
```
同时通过docker inspect命令查看redis容器的参数，可以发现其中卷挂载有如下配置：
```
"Mounts": [
    {
        "Source": "/var/lib/mesos/slaves/fb9d0623-c0bf-402e-9288-06206b30ea54-S4/frameworks/487f9562-5998-4cfb-bc92-bbc3a441c69c-0000/executors/redis_single.bd21b767-41b8-11e6-9d7a-024259ae7178/runs/8c9304a1-b960-4aa0-a169-f368a4d77ed9/redis",
        "Destination": "/data",
        "Mode": "rw",
        "RW": true,
        "Propagation": "rprivate"
    },
    {
        "Source": "/var/lib/mesos/slaves/fb9d0623-c0bf-402e-9288-06206b30ea54-S4/frameworks/487f9562-5998-4cfb-bc92-bbc3a441c69c-0000/executors/redis_single.bd21b767-41b8-11e6-9d7a-024259ae7178/runs/8c9304a1-b960-4aa0-a169-f368a4d77ed9",
        "Destination": "/mnt/mesos/sandbox",
        "Mode": "",
        "RW": true,
        "Propagation": "rprivate"
    }
]
```
既创建的挂载点已经被挂到容器的指定路径了。

然后我们进行验证：
首先通过redis客户端连接到redis上，然后设置工作路径为我们挂载的路径：
```
redis-cli -h 1.1.1.1 CONFIG SET dir /data
```
然后写入数据：
```
redis-cli -h 1.1.1.1 set a b
```
为了确保能够正常持久化，执行：
```
redis-cli -h 1.1.1.1 bgsave
```

查看挂载卷路径，可以发现已经有dump.rdb文件产生。此时kill掉redis任务，等待marathon重新启动。再次启动之后，通过redis-cli连接进入redis之后，可以看见之前设置的key仍然存在：
```
> redis-cli -h 1.1.1.1 keys *
1) "a"
```
然后获取这个key的值：
```
> redis-cli -h 1.1.1.1 get a
1) "b"
```

从这个实验可以看出，marathon的本地存储卷可以确保服务重新启动之后原来的数据仍然存在，对于存储类的应用（如redis、mysql等）非常有用。
