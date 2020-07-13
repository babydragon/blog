# criu初探——收集进程树

这是整个dump过程中第一个算是真的开始和目标进程交互的操作，也是第一个对目标进程有影响的操作。

因为会涉及到冻结目标进程，在开始之前，有两个准备动作：
1. 开始计时，用以统计实际冻结的时间
2. 设置定时器，确保当前进程未处理完时，可以通过SIGALARM来通知超时，以完成退出和清理工作

如果没有设置freeze-cgroup，会调用`compel_interrupt_task`函数冻结目标进程:

1. 首先通过`ptrace`发送`PTRACE_SEIZE`请求开始追踪目标进程，`PTRACE_SEIZE`和`PTRACE_ATTACH`相比最大的优势是不会停止目标进程。
2. 通过`ptrace`发送`PTRACE_INTERRUPT`请求暂停目标进程

中断之后，会通过`compel_wait_task`函数来检查目标进程状态，这个过程大致为：
1. 通过`wait4`来判断目标进程状态，同时结合读取proc中的status文件来判断进程状态，如果此时进程退出或是僵尸状态，则会退出。只有为T状态时才会继续（正常都应该在这个状态，因为上一步已经通过`ptrace`让目标进程进入T状态了）
2. 通过`ptrace`发送`PTRACE_GETSIGINFO`请求查询进程收到的信号，如果不是因为ptrace让进程进入T状态，则尝试重新恢复进程，并重试这个过程。
3. 一切正常则继续往下执行，返回最终的进程状态为`COMPEL_TASK_ALIVE`

其实前面也基本上算是准备工作，往下才真正调用`collect_task`函数开始收集进程信息。

为了确保在搜索进程包含的线程和子进程的时候，不会有新增，所有的查询都包了一层重试，当到达重试次数（5次）或者扫描没有新的线程/子进程的时候，才会停止。

首先获取的是线程数据。获取的流程(`collect_threads`)：
1. 通过`/proc/$PID/task`目录读取所有的线程信息
2. 循环遍历每个线程，查询之前是否已经处理过。
3. 对没有处理过的线程，和主进程一样，通过`PTRACE_SEIZE`追踪这个线程（`compel_interrupt_task`）
4. 查询线程状态是否为T（`compel_wait_task`）
5. 保存线程信息

然后是通过深度优先的方式遍历所有子线程。读取方式为（`collect_children`）：
1. 打开`/proc/$PID/task`目录
2. 读取目录中的每个子目录下的`children`文件，既`/proc/$PID/task/$TID/children`文件
3. 读取文件中的pid
4. 对于每个pid，判断之前是否已经处理过了
5. 对于没有处理过的pid，首先判断定时器是否超时，超时就不能再处理了
6. 通过`PTRACE_SEIZE`追踪这个进程（`compel_interrupt_task`）
7. 查询进程状态是否为T（`compel_wait_task`）
8. 对于是T状态的进程，再次递归调用`collect_task`函数，递归收集其线程和子进程

注意，查询了manpage，对于`/proc/$PID/task/$TID/children`文件的描述，该文件仅倾向于给CR应用使用，仅当所有子进程都已经进入T状态，内容才是可靠的。该文件在4.2版本的Kernel之前通过`CONFIG_CHECKPOINT_RESTORE`配置控制，4.2版本以后通过`CONFIG_PROC_CHILDREN`配置控制。