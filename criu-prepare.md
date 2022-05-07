# criu初探——dump前准备

从入口来看最基本的准备流程有：
1. 入口函数：cr_dump_tasks，函数入参为需要dump的PID
2. 修改当前进程打开文件描述符数量：rlimit_unlimit_nofile，通过prlimit函数修改当前dump进程的RLIMIT_NOFILE值。为了尽可能设置的大，cuid会去读取`/proc/sys/fs/nr_open`文件来获取内核支持的最大值。如果读取失败，设置一个默认值：1024k
3. 初始化进程数结构：alloc_pstree_item，实际调用__alloc_pstree_item(false)函数，初始化一个pstree_item链表结构体
4. 执行dump前脚本：run_scripts
5. 初始化状态记录：init_stats，初始化dstats对象，为其分配内存，用于后续统计每个阶段耗时。
6. 初始化插件：cr_plugin_init，暂时不深入了解插件系统
7. 检查LSM参数：lsm_check_opts，检查Linux安全模块（Linux Security Module）设置，不深入了解
8. 加载inode逆向映射缓存：irmap_load_cache。irmap（Inode Reverse MAP）是criu维护的从inode到文件路径的逆向映射关系，保存它的原因是因为criu在dump inotify的时候，只知道inode信息，但是无法知道这个inode对应文件系统的哪个文件。具体用途描述参见[文档](https://criu.org/Irmap)这里只是加载之前缓存下来的镜像。
9. 初始化CPU信息：cpu_init，检查是否有不支持的CPU指令集
10. 探测vdso支持：vdso_init_dump，检测是否支持虚拟动态链接库（virtual dynamic shared object）
11. 初始化cgroup list:cgp_init
12. 加载cgroup的控制器：parse_cg_info
13. 初始化dump清单：prepare_inventory
14. 尝试连接page server：connect_to_page_server_to_send
15. 设置SIGALRM信号处理器：setup_alarm_handler，后续因为需要阻塞进程，需要通过定时器信号来实现超时什么的。

整个准备流程比较清晰，大部分是对全局变量初始化，以及被dump进程检查。

### vdso_init_dump
该方法用于初始化vsdo的dump

基本流程：

1. vdso_parse_maps：解析`/proc/PID/maps`文件，该文件保存了进程所有的内存映射（mmap）信息。对于vdso，解析其中的vdso和vvar两个内存映射，获取其内存映射的开始和结束地址。
2. 检查长度和内核数据kdat中获取的是否相同
3. 检查kdat中页表加载完成，通过`vaddr_to_pfn`函数将vdso_start地址转换成pfn（page frame number，页帧号）

发现这里需要依赖内核的kdat数据结构，这个数据结构在命令刚开始执行的时候，就已经调用`kerndat_init`函数完成。