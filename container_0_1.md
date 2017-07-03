# 从0开始构建容器

Docker掀起了容器热潮，但是Docker不等于容器，本文介绍如何使用Linux内核提供的特性，一步步创建一个容器的过程。**注意：本文需要使用两个shell环境。shell提示符$表示主机环境，#表示chroot环境。**

## 准备文件系统
Docker镜像一般都会完整包含Linux系统的一些标准目录，为了方便，我们首先从busybox镜像中导出整个目录结构：

```
mkdir rootfs
docker export $(docker create busybox) | tar -C rootfs -xvf -
```

生成的目录结构为：
```
$ tree -d
.
├── bin
├── dev
│   ├── pts
│   └── shm
├── etc
├── home
├── proc
├── root
├── sys
├── tmp
├── usr
│   └── sbin
└── var
    ├── spool
    │   └── mail
    └── www

16 directories
```
这样，基本上和主机的目录结构一致了。

## chroot
顾名思义，```chroot```命令就是修改root目录位置，执行之后的shell里面，/目录将会切换到新的目录。通过该命令，可以将文件系统的操作范围约束到指定目录中。

```
sudo chroot rootfs /bin/sh
```

注意，chroot需要root权限才能操作。上述命令执行之后，当前shell的```/```目录变成了主机操作系统的rootfs目录中。并且将shell切换成```rootfs/bin/sh```（因为用busybox容器导出，没有bash）。

然后我们在chroot环境中启动一个进程：
```
nc -l -p 8888
```
该命令会监听8888端口，主机可以通过TCP连接8888端口的方式和它进行交互。

## 使用namespace进行隔离
之前我们在chroot环境中启动了一个nc进程，但此时chroot环境和外部环境是没有任何权限隔离的。为了验证，我们先在chroot环境外部也启动一个nc进程，监听8889端口：
```
nc -l -p 8888
```

此时，在chroot外部查看nc进程结果为：
```
$ ps -ef | grep -w nc
root     19063 18323  0 17:26 pts/9    00:00:00 nc -l -p 8888
babydra+ 19192   794  0 17:29 pts/7    00:00:00 nc -l -p 8889
```
我们可以看见两个nc进程，由于chroot必须使用root权限，所以chroot环境中启动的nc进程的所属用户是root。同样因为这个原因，在chroot环境中，可以直接kill掉chroot之外的这个nc进程：

```
# kill 19192
```

此时，再再chroot之外环境查看，nc进程只剩下了一个：
```
$ ps -ef | grep -w nc
root     19063 18323  0 17:26 pts/9    00:00:00 nc -l -p 8888
```

这样的权限显然是不符合容器隔离的要求的，此时虽然隔离了文件系统，但容器可以操作非容器启动的进程。为了解决这个问题Linux内核引入了namespace，在命令行中，可以通过```unshare```命令来激活该特性对chroot中的环境进行隔离。

```
unshare -p -f -m --mount-proc  chroot rootfs /bin/sh
mount -t proc none /proc
```

第一个命令将pid、fork、mount都进行了隔离，第二个命令是mount了proc目录，以确保ps等命令可以运行。下面我们就用ps命令查看下chroot环境中的进程：
```
# ps ux
PID   USER     TIME   COMMAND
    1 root       0:00 /bin/sh
    4 root       0:00 ps ux
```

我们可以看见，在chroot环境中的进程pid和外部已经隔离，最初始的sh进程pid是1。这样，chroot环境已经和Docker容器类似，无法再查看外部的进程了。

## 使用nsenter命令进入容器
前面通过```unshare```命令将chroot环境运行的进程放入了独立的namespace中，如果要进入这个namespace，可以使用```nsenter```命令。```nsenter```命令封装了[```setns```](https://linux.die.net/man/2/setns)系统调用，可以将进程设置为对应的namespace。

使用该命令前，我们首先需要找到chroot环境中的sh进程对应的namespace。首先需要在chroot环境之外，获取sh进程的PID：
```
ps -ef | grep -w /bin/sh
```

找到对应的进程PID为4899，内核会将进程的namespace信息暴露在```/proc/$PID/ns```目录中。
```
$ ls -l /proc/4899/ns
总用量 0
lrwxrwxrwx 1 root root 0 2月  28 09:47 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 root root 0 2月  28 09:47 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 root root 0 2月  28 09:47 mnt -> mnt:[4026532625]
lrwxrwxrwx 1 root root 0 2月  28 09:47 net -> net:[4026531957]
lrwxrwxrwx 1 root root 0 2月  28 09:47 pid -> pid:[4026532632]
lrwxrwxrwx 1 root root 0 2月  28 09:47 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 2月  28 09:47 uts -> uts:[4026531838]
```

在**主机**新的shell中运行：
```
nsenter --pid=/proc/4899/ns/pid chroot rootfs /bin/sh
```

其中需要给```nsenter```命令pid namespace文件，进入chroot环境。然后再重新挂载下proc文件系统，以确保ps命令可以正常输出：
```
mount -t proc none proc
```

这时，我们可以用ps命令查看现在chroot环境中的进程：
```
# ps
PID   USER     TIME   COMMAND
    1 root       0:00 /bin/sh
   13 root       0:00 /bin/sh
   16 root       0:00 ps
```

这个例子，是不是让我们联想到了```docker exec```命令。

## cgroups
cgroups是control groups的简称，它是内核提供的资源隔离方式，目前可以隔离的资源包括CPU、内存、磁盘IO等。

```
$ ls /sys/fs/cgroup/
blkio  cpu  cpuacct  cpu,cpuacct  cpuset  devices  freezer  hugetlb  memory  net_cls  net_cls,net_prio  net_prio  perf_event  pids  systemd
```

如果要对内存进行限制，需要先在cgroup中创建一个目录，配置资源限制，然后再将需要限制的进程PID加入到其中。
```
mkdir /sys/fs/cgroup/memory/demo
echo "1048576" > /sys/fs/cgroup/memory/demo/memory.limit_in_bytes
echo "0" > /sys/fs/cgroup/memory/demo/memory.swappiness
echo '4899' > /sys/fs/cgroup/memory/demo/tasks
```
第一个命令在cgroup的memory目录中创建一个新的目录，用于限制内存使用。第二个命令设置内存限制为1M字节。第三个命令限制强制不使用swap。第四个命令，将刚才chroot环境中的sh进程加入改cgroup中。这样，我们就完成了对chroot环境的内存设置。

然后我们在chroot环境中进行验证：
```
for i in $(seq 1 1048576); do str="$i$str"; echo ;done
```
这里模拟创建一个很大的字符串，以占用内存。执行之后，sh进程被OOM kill，shell会提示：
```
Killed
已杀死
```

当创建的cgroup中的进程都已经退出，可以通过```rmdir```命令删除目录：
```
rmdir /sys/fs/cgroup/memory/demo
```

以上，模拟的步骤和Docker容器启动时使用-m参数限制内存效果相同。

## capabilities
capabilities是内核限制进程能够执行操作的方式之一，和SELinux类似，通过capabilities可以限制root用户在内的所有用户能够执行的操作。capabilities的所有限制可以参考[文档](https://linux.die.net/man/7/capabilities)。

在shell中，我们可以通过```capsh```命令来模拟capabilities的剥夺。
```
capsh --drop=CAP_NET_BIND_SERVICE --
nc -l -p 80
```
这里我们都以root用户来执行，第一个命令通过capsh来剥夺root用户bind权限。默认情况下，root用户可以bind所有端口，非root用户可以bind大于1024的端口。执行了该命令之后，再通过nc命令来尝试bind 80端口，结果为：
```
Can't grab 0.0.0.0:80 with bind : Permission denied
```

从这里我们可以看见，即使是root用户，也没有权限再去bind 80端口。

再回到模拟容器操作来：
```
capsh --drop=cap_chown -- -c 'unshare -p -f -m --mount-proc  chroot rootfs /bin/sh'
```

该命令在执行unshare之前，先通过capsh来剥夺了chown的权限。因此我们在chroot环境内，已经无法再执行该命令：
```
# whoami
root
# chown nobody /bin/sh
chown: /bin/sh: Operation not permitted
```

以上，模拟Docker中的--privileged和--cap-add等参数，具体可以参照Docker run[文档](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities)。

## 总结
容器 != Docker，Docker也不仅仅是容器。OCI正在试图规范从容器镜像格式到运行时的一系列标准。本文介绍了容器是如何通过内核提供的种种特性，做到资源的隔离，既能充分利用资源，又能避免安全问题。
