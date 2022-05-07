# linode上的Gentoo从openRC改成systemd

linode上的vps已经很久没去搞了，上面还是基于openRC的Gentoo，对于自动启动、服务管理什么的，都已经不太友好了。因此打算切换成systemd。
另外，linode提供的lish也可以不基于网络，万一踩坑了，还可以搞定。

## 系统准备
Gentoo对于使用systemd有比较完备的[文档](https://wiki.gentoo.org/wiki/Systemd)，整体操作起来并不是太复杂。

整个过程可以分成几部：
1. 系统配置切换
2. 软件包USE修正
3. 搞定内核和grub

当然，在做以上三步之前，首先需要确认已经完成了整个world的更新（特别是portage什么的），避免切换systemd的时候再掺杂其他软件包冲突什么的。

### 系统配置切换

基于当前最新版本的portage，通过：

```bash
eselect profile list
```
找到一个基于systemd的系统配置。由于是用在vps上的，不需要图形界面，我这里选了“default/linux/amd64/17.1/systemd”，通过命令：

```
eselect profile set XXX
```
来进行设置。这里xxx就是选中profile对应的序列号。

修改配置之后，立即进行全局的更新，确保所有systemd相关的包都会更新，同时一些包提供了systemd USE的，也都开启：

```bash
emerge -avDuN1 @world
```

最后，参照文档做一些简单的系统改动：
```
ln -sf /proc/self/mounts /etc/mtab
```

这样即可完成上面的两步了。

### 搞定内核和grub
由于之前这个系统在linode上使用的是linode提供的最新版本内核，因此需要自己安装内核和grub。

首先是安装内核源码：

```bash
emerge -av gentoo-sources
```

然后通过`eselect kernel set 1`来设置当前使用的内核。

进入到内核源码目录（`/usr/src/linux`，如果没有设置当前使用的内核，默认没有这个软连接），首先将当前内核使用的配置复制过来，
这样可以避免驱动什么的问题：

```bash
cat /proc/config.gz | gunzip > .config
```

然后就可以开始选择需要的内核选项了：

```bash
make menuconfig
```

这里不做过多选择，直接检查Gentoo提供的[systemd文档](https://wiki.gentoo.org/wiki/Systemd#Kernel)里面必须开启的内核参数即可。如果有耐心，可以裁剪一下多余的模块。

确认依赖的模块和选项都正常开启之后，开始编译和安装内核：

```bash
make -j 3 && make modules_install && make install
```

搞定内核之后，还需要搞定grub。还是因为之前用了默认的内核，因此在虚拟机内部都没有安装引导器。

```
emerge -av grub
```

安装grub之后，参照linode自定义内核文档，修改一下grub配置（这里不太清楚为什么需要设置这些内核输出。。。）。
修改：`/etc/default/grub`文件：

```
GRUB_DISTRIBUTOR="Gentoo"
GRUB_CMDLINE_LINUX="console=ttyS0,19200n8 net.ifnames=0 init=/usr/lib/systemd/systemd"
GRUB_TERMINAL=serial
GRUB_GFXPAYLOAD_LINUX=text
GRUB_SERIAL_COMMAND="serial --speed=19200 --unit=0 --word=8 --parity=no --stop=1"
```

这里和systemd唯一有关的，实际就只有设置内核参数：`init=/usr/lib/systemd/systemd`

设置完成之后，可以通过：

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```
来生成grub的配置。

### 网络配置
这里是本地切换最大的踩坑点，下面会详细提到。这里先写下解决方案。

创建`/etc/systemd/network/50-static.network`文件，内容为：
```
[Match]
Name=eth0

[Network]
DHCP=yes
```

这里大致意思就是配置eth0网络为通过dhcp动态获取。linode的文档上给的是竟然配置方式（当然，实际给vps的肯定也是静态固定的IP），
但是经过尝试，这里使用动态获取IP地址也没问题，实际eth0网卡上获取到的还是给vps分配的IP地址（包括v4和v6地址）。

### linode控制台修改
之前的vps设置的内核是最新的64位内核，没法保证systemd依赖的选项都开启。所以前面我们已经完成了内核的编译和安装。
现在需要去linode控制台修改下使用的内核。

在控制台 https://cloud.linode.com/ 页面找到对应的vps，进去之后选择“configurations”，选择edit当前配置，将Boot Settings中
的内核，改成grub即可。

完成上述改动后重启vps。这里由于更新systemd的时候把openrc都给删了，会导致`reboot`、`shutdown`等命令会失效，最终还是vps上强制关机了。

## 重新开机后的配置以及踩坑
完成vps重启之后，会发现本地ssh连不上了！！！这里有几个原因：

1. systemd的网络服务没有开启，同时我实际配置的时候，没有创建对应的网络配置
2. 从openRC迁移到systemd之后，原先的系统服务并不会自动迁移，即使是已经提供了systemd的unit文件，也不会启动。

还好linode提供了网页版的lish，能够方便的登录没有网络的vps。通过lish登录机器，然后创建前面说的网络配置，并开启systemd的网络服务：

```
systemctl enable systemd-networkd.service
systemctl start systemd-networkd.service
```

服务正常启动之后，通过`ifconfig`就能看见eth0设备了，上面的IP地址也是vps原本分配的地址。

此时其他的服务都还没有开启的，再通过：
```
systemctl start sshd
```
先开启sshd，这样本地就可以直接通过ssh来连接vps了。

解决网络问题之后，其他问题就很方便了，vps上的大部分服务（例如apache, php, mysql等）都已经提供了systemd unit文件了，
直接一个个start一下，同时enable一下，确保以后会开机自动启动即可。

总结一下，迁移过程中有两个比较恶心的地方：
1. 升级完成后，openrc提供的命令都没了，systemd又因为1号进程不是systemd的daemon进程，导致没法直接reboot
2. 网络设置最好能够提前做好，还好lish这种访问方式可以事后弥补，但是登录不上机器还是感觉慌兮兮的。