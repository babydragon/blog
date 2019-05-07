# Gentoo蓝牙鼠标问题

最近买了个罗技MX master 2s，今天第一次试用。电脑蓝牙可以顺利发现和配对鼠标，但是连接之后，鼠标没有任何反应。

首先，按照Gentoo的[蓝牙文档](https://wiki.gentoo.org/wiki/Bluetooth) 检查蓝牙相关的内核配置，
发现内核配置没有问题，之前的M505鼠标使用也没有任何问题。

使用`bluetoothctl`命令看了下连接的情况，也没有任何问题。

通过`journalctl`命令查看蓝牙的日志（`journalctl -u bluetooth.service`），发现和鼠标配对之后，
有如下错误：

```
input-hog profile accept failed for xxx
```

搜索了下，在Gentoo论坛里有这么个帖子： https://forums.gentoo.org/viewtopic-t-1073060-start-0.html
蓝牙鼠标需要加载`uhid`内核模块才能生效。于是也尝试了下，发现之前根本没有编译这个模块。这个模块配置是
`CONFIG_UHID`，将其设置成模块之后，重新编译内核。然后通过命令：

```bash
modprobe uhid
```
加载该内核模块，鼠标终于可以使用了。

另外，MX master 2s的按钮默认都支持，包括前进、后退按钮、侧向滚动和手势按钮。通过Arch关于此款鼠标的[文档](https://wiki.archlinux.org/index.php/Logitech_MX_Master)，手势按钮（就是大拇指下面这个按钮）会发送`Ctrl+Alt+Tab`组合键。

对于KDE图形界面来说，通过在系统设置中设置全局快捷键，既可将这个按钮绑定到“切换显示窗口”，既将所有窗口排列到屏幕上。
