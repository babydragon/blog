## load是什么

通常如果感觉系统变慢了，都会去看下系统load，一般开发也会比较关注线上机器的load情况。

查看系统load的方式很多，比较常用的有`uptime`和`top`命令。这些命令都会输出三个数值，分别表示1分钟load、5分钟load和15分钟load。
这些数字能够给用户一些直观的概念：

* 如果load为0.0,说明系统没有任何负载。
* 如果1分钟load比5分钟load或者15分钟load高，说明系统负载在上升，当前可能有比较消耗CPU的进程在运行。
* 如果1分钟load比5分钟load或者15分钟load低，说明系统负载在下降，说明之前可能有比较消耗CPU的进行。
* 如果load数值比CPU核数高，系统**可能**遇到性能问题。


## 三个数值怎么来的？

前面提到了load的数值，通常意义来说，load为1表示一个CPU内核满负荷运行。参照load计算代码（[loadavg.c](https://github.com/torvalds/linux/blob/master/kernel/sched/loadavg.c)）注释来看：

```
 * Once every LOAD_FREQ:
 *
 *   nr_active = 0;
 *   for_each_possible_cpu(cpu)
 *	nr_active += cpu_of(cpu)->nr_running + cpu_of(cpu)->nr_uninterruptible;
 *
 *   avenrun[n] = avenrun[0] * exp_n + nr_active * (1 - exp_n)
```

计算的伪代码很简单，统计每个CPU上的处于运行态和不可中断状态的进程数量，然后计算得出。`exp_n`定义在[loadavg.h头文件中](https://github.com/torvalds/linux/blob/master/include/linux/sched/loadavg.h)。

```c
#define FSHIFT		11		/* nr of bits of precision */
#define FIXED_1		(1<<FSHIFT)	/* 1.0 as fixed-point */
#define LOAD_FREQ	(5*HZ+1)	/* 5 sec intervals */
#define EXP_1		1884		/* 1/exp(5sec/1min) as fixed-point */
#define EXP_5		2014		/* 1/exp(5sec/5min) */
#define EXP_15		2037		/* 1/exp(5sec/15min) */

#define CALC_LOAD(load,exp,n) \
	load *= exp; \
	load += n*(FIXED_1-exp); \
	load >>= FSHIFT;
```

[http://www.teamquest.com/import/pdfs/whitepaper/ldavg1.pdf](http://www.teamquest.com/import/pdfs/whitepaper/ldavg1.pdf)
