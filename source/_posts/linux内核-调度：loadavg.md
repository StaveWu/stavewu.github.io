---
title: 调度：loadavg
date: 2025/04/04 00:00:00
cover: /images/covers/09-diary.jpg
thumbnail: /images/covers/09-diary.jpg
toc: true
categories: 
 - linux内核
tags:
 - 调度
---

前面我们了解了[PELT算法](/linux内核-调度：PELT算法/)，它提供了一种计算任务负载的方式。在linux系统中，除了PELT外，还有另一种观测负载的方式 —— loadavg，具体可以查看`/proc/loadavg`文件：

```bash
root@ubuntu-server:~# cat /proc/loadavg 
0.00 0.09 0.15 1/356 34944
```

其中，前3个数值分别表示系统在1分钟、5分钟和15分钟的负载。关于负载的作用本文不再撰述，感兴趣的可以翻看前面PELT文章介绍，本文将重点讨论loadavg这三个数值都是怎么计算的，以及为什么这么算。

<!-- more -->

## 负载建模

内核对loadavg的计算采用了[指数滑动平均](https://zh.wikipedia.org/wiki/%E7%A7%BB%E5%8B%95%E5%B9%B3%E5%9D%87#%E6%8C%87%E6%95%B8%E7%A7%BB%E5%8B%95%E5%B9%B3%E5%9D%87)，公式如下：
$$
L_n = \alpha L_{n -1} + (1 - \alpha)x_n  \tag{1}
$$
其中，$L_n$表示截止至当前时刻的负载，$\alpha \in (0, 1)$为衰减系数，$x_n$为当前时刻系统中活跃的任务数，$L_0$等于0，$x_0$等于0。

如果我们将该递推公式展开，就能看到滑动平均的本质，即对历史的所有${x_i}$做了衰减系数平均：
$$
\begin{split}
L_n &= \alpha L_{n -1} + (1 - \alpha)x_n \\\\
&= \alpha (\alpha L_{n-2} + (1 - \alpha)x_{n-1}) + (1 - \alpha)x_n \\\\
&= \alpha^2L_{n-2} + (1-\alpha)(\alpha x_{n-1} + x_n) \\\\
&= \alpha^n L_0 + (1-\alpha)(\alpha^{n-1}x_1 + ... + x_n) \\\\
&= (1-\alpha)\sum_{i=1}^{n}\alpha^{i}x_{n-i} \\\\
&= \frac{\sum_{i=1}^{n}\alpha^{i}x_{n-i}}{\frac{1}{1-\alpha}} \\\\
&= \frac{\sum_{i=1}^{n}\alpha^{i}x_{n-i}}{\sum_{i=0}^{+\infty}\alpha^i}
\end{split} \tag{2}
$$
公式(2)中，$1-\alpha$应用了等比数列求和公式转为分母$\sum_{i=0}^{+\infty}\alpha^i$。

有没有发现公式(2)和PELT公式有点相似？回忆下PELT中计算最近某段观测窗口$\Delta t_{win}$内的公式：
$$
\frac{\Delta t_{win}}{T} 
= \frac{d_1 y^p + 1024\sum_{i=1}^{p-1}y^i + d_3y^0}{1024\sum_{i=1}^{+\infty}y^i + d_3}  \tag{3}
$$
假设观测窗口是周期$PERIOD$的整数倍，那么$d_1 = d_3 = PERIOD == 1024$，代入公式(3)将得到：
$$
\begin{split}
\frac{\Delta t_{win}}{T}
&= \frac{1024 y^p + 1024\sum_{i=1}^{p-1}y^i + 1024y^0}{1024\sum_{i=1}^{+\infty}y^i + 1024} \\\\
&= \frac{1024\sum_{i=0}^{p}y^i}{1024\sum_{i=0}^{+\infty}y^i}
\end{split}  \tag{4}
$$
公式内分子和分母的1024含义各有不同，分子中1024是运行时间，分母中1024是周期$PERIOD$。由于在实际场景中，运行时间不总是满周期，往往因调度等原因被碎片为一个运行时间序列$[\Delta t_1, \Delta t_2, ..., \Delta t_n]$，故公式(4)实际为：
$$
\begin{split}
\frac{\Delta t_{win}}{T}
&= \frac{\sum_{i=0}^{p}y^i\Delta t_i}{1024\sum_{i=0}^{+\infty}y^i}
&= \frac{\sum_{i=0}^{p}y^i\frac{\Delta t_i}{1024}}{\sum_{i=0}^{+\infty}y^i}
\end{split}  \tag{5}
$$
其中，$\frac{\Delta t_i}{1024}$为单周期的运行占比。此时若将其看作$x_i$，那么公式(5)就和前面我们推导的loadavg公式(2)一致。这就揭露了两个事实：

1. PELT的本质也是指数滑动平均！
2. 指数滑动平均的本质其实就是对历史的值序列做指数衰减。在PELT内，该值序列是运行占比，而在loadavg内值序列则是系统某一时刻的活跃任务数。

## 衰减系数选取

接下来看下1/5/15分钟的负载各分别是怎么算的。

1/5/15分钟是一个观测窗口大小，表示过去x分钟内的负载情况。已知loadavg的值序列是活跃任务数，那么1/5/15分钟的负载就代表了过去x分钟内的平均活跃任务数。同样，这里的“平均”涉及到了衰减周期和衰减系数。内核中衰减周期取5s；衰减系数取$e^{-5/win}$，其中$win$对应到1/5/15分钟。

我们将衰减系数代入到公式(2)，将获得：
$$
\begin{split}
L_n &= e^{-5/win} L_{n -1} + (1 - e^{-5/win})x_n \\\\
&= \frac{\sum_{i=1}^{n} (e^{-5/win})^{i} x_{n-i}}{\sum_{i=0}^{+\infty}(e^{-5/win})^{i}}
\end{split}  \tag{6}
$$
我们知道，前面PELT的衰减系数是固定的，但这里loadavg的系数却是跟随1/5/15分钟的取值而变，应如何理解？

笔者尝试从概率的角度来其背后的含义。

已知，$x_i$是以5s为周期粒度观测到的任务数，设$y_i$为以$win$窗口粒度观测到的任务数，则$x_i$可改写为$x_i = \frac{5}{win}y_{i}$。将其代入到公式(6)，可得：
$$
\begin{split}
L_n &= \frac{\sum_{i=1}^{n} (e^{-5/win})^{i} (\frac{5}{win}y_{n-i})}{\sum_{i=0}^{+\infty}(e^{-5/win})^{i}} \\\\
&= \frac{\sum_{i=1}^{n} \lambda e^{-\lambda i} (y_{n-i})}{\sum_{i=0}^{+\infty}e^{-\lambda i}},\ \lambda = \frac{5}{win}
\end{split}  \tag{7}
$$
现在从概率的角度来看公式中的$\lambda e^{-\lambda i}$，它其实是一个服从参数为$\lambda$的[指数分布](https://zh.wikipedia.org/wiki/%E6%8C%87%E6%95%B0%E5%88%86%E5%B8%83)的概率密度函数，也就是：
$$
f(x;\lambda) = \lambda e^{-\lambda x}, \ x \ge 0 \tag{8}
$$
已知$i$时刻的活跃任务数为$y_{n-i}$，概率密度为$f(i;\lambda)$，则公式(7)分子可以写成：$\sum_{i=1}^{n}y_{n-i}f(i;\lambda)$，仔细想想，这不就是在求$n$次观测的平均期望吗？

所以，衰减系数选取$e^{-5/win}$背后的依据其实是建立在一个任意时刻观测活跃任务的数量应服从指数分布的概率假设上，而$5/win$则是为了对当前时刻活跃任务数在不同观测窗口粒度内进行转换。

## 内核实现细节

关于loadavg的计算代码位于include/linux/sched/loadavg.h。衰减周期和系数定义如下：

```c
#define FSHIFT		11		/* nr of bits of precision */
#define FIXED_1		(1<<FSHIFT)	/* 1.0 as fixed-point */
#define LOAD_FREQ	(5*HZ+1)	/* 5 sec intervals */
#define EXP_1		1884		/* 1/exp(5sec/1min) as fixed-point */
#define EXP_5		2014		/* 1/exp(5sec/5min) */
#define EXP_15		2037		/* 1/exp(5sec/15min) */
```

原本通过$e^{-5/win}$计算获得的`EXP_{1/5/15}`应为小数，这里却为整数，原因和PELT一样，仍然是为了规避浮点运算所做的左移放大，具体可以拿`EXP_1`验证：

```python
>>> 1 / math.exp(5/60) * (2**11)
1884.2509611608539
```

所以，当前`EXP_*`定义的值即为经过`1 << FSHIFT`（对应`FIXED_1`宏）放大后的结果。放大后，loadavg计算公式为：
$$
\begin{split}
L_n &= \frac{FIXED\_1 (e^{-5/win} L_{n -1} + (1 - e^{-5/win})x_n)}{FIXED\_1} \\\\
&= \frac{FIXED\_1\ e^{-5/win}L_{n-1} + (FIXED\_1 - FIXED\_1\ e^{-5/win})x_n}{FIXED\_1} \\\\
&= \frac{EXP_{win} L_{n-1} + (FIXED\_1 - EXP_{win})x_n}{FIXED\_1}
\end{split} \tag{9}
$$
因此代码实现如下：

```c
/*
 * a1 = a0 * e + a * (1 - e)
 */
static inline unsigned long
calc_load(unsigned long load, unsigned long exp, unsigned long active)
{
	unsigned long newload;

	newload = load * exp + active * (FIXED_1 - exp);
	if (active >= load)
		newload += FIXED_1-1;

	return newload / FIXED_1;
}
```
可以看到，和前面的公式(9)几乎无异。

那么这个函数在什么时候调用呢？可以看如下调用路径，内核通过一个定时器在每次tick时就尝试调用`calc_load`更新loadavg：

`tick_handle_periodic => tick_periodic => do_timer(1) => calc_global_load => calc_load`

其中，`calc_global_load`完成了对前一次负载`load`、`exp`、`active`的传值：

```c
// file: kernel/sched/loadavg.c
void calc_global_load(void)
{
	unsigned long sample_window;
	long active, delta;

	sample_window = READ_ONCE(calc_load_update);
	if (time_before(jiffies, sample_window + 10))
		return;  // 限制更新频率，在还没到sample_window的时间前不计算。+10是一个经验值，确保CPU已经更新了calc_load_tasks
    ...
	active = atomic_long_read(&calc_load_tasks);
	active = active > 0 ? active * FIXED_1 : 0;  // interesting, active也做了放大。是担心任务数值太小导致最终load变化不明显？
	
    // 计算1,5,15分钟的负载
	avenrun[0] = calc_load(avenrun[0], EXP_1, active);
	avenrun[1] = calc_load(avenrun[1], EXP_5, active);
	avenrun[2] = calc_load(avenrun[2], EXP_15, active);

	WRITE_ONCE(calc_load_update, sample_window + LOAD_FREQ);  // sample_window往后LOAD_FREQ递增，表示在LOAD_FREQ后再进行计算
}
```

注意到上述函数中当前时刻的活动任务数存储在全局变量`calc_load_tasks`，而更新该全局变量则是通过如下路径：

`sched_tick => calc_global_load_tick`

其中`sched_tick`由每个CPU上的时钟触发。在`calc_global_load_tick`中，通过获取各CPU rq上记录的上一次任务数与当前时刻任务数的差值来更新全局的活跃任务数`calc_load_tasks`：

```c
// file: kernel/sched/loadavg.c
void calc_global_load_tick(struct rq *this_rq)
{
	long delta;

	if (time_before(jiffies, this_rq->calc_load_update))
		return;  // 限制更新频率

	delta  = calc_load_fold_active(this_rq, 0);  // 获取rq上任务数变化情况
	if (delta)
		atomic_long_add(delta, &calc_load_tasks);  // 更新全局任务数

	this_rq->calc_load_update += LOAD_FREQ;
}
```
