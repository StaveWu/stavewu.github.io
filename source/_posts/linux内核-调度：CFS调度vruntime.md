---
title: 调度：CFS调度vruntime
date: 2025/04/19 00:00:00
cover: /images/covers/04-pencils.jpg
thumbnail: /images/covers/04-pencils.jpg
toc: true
categories: 
 - linux内核
tags:
 - 调度
---

CFS调度器（Completely Fair Scheduer），顾名思义，即想要实现多任务在同一硬件上能够被完全公平的调度。内核中对“完全公平”的定义如下：

<!-- more -->

> "Ideal multi-tasking CPU" is a (non-existent  :-)) CPU that has 100% physical power and which can run each task at precise equal speed, in parallel, each at 1/nr_running speed.  For example: if there are 2 tasks running, then it runs each at 50% physical power --- i.e., actually in parallel.
>
> *—— Documentation/scheduler/sched-design-CFS.rst*

大致意思为：对于有100%算力的CPU，如果有n个任务在其上面跑，那么这些任务应能够均分100%算力，也就是每个任务各领1/n。

在过去，芯片比较简陋，每个芯片只有1个CPU资源，那么CFS调度器要解决的是在单CPU上如何公平调度的问题；现如今，芯片逐步发展为多核SMP架构，CFS除了考虑单CPU的调度以外，还需要考虑多核之间如何实现负载均衡。好在这两个问题是互补的，在解决单CPU的公平调度问题后，多CPU的负载均衡可以在单CPU上进行扩展。

本文将围绕单CPU的公平调度展开，为限制篇幅，本文仅讨论vruntime的设计以及其和我们在linux系统中常见的nice值的关系。

## vruntime是什么

vruntime，即虚拟时间virtual runtime，是对任务运行时间的衡量。vruntime是通过物理时间（也叫墙上时间wall time，即我们日常手表真实走过的时间）伸缩而来。之所以要引入vruntime，是因为要简化完全公平算法的设计。

考虑这样的一个场景：

给定3个任务ABC运行100ms，在没有其他约束的情况下，完全公平算法应做到：

- 任务A运行33ms
- 任务B运行33ms
- 任务C运行33ms

也就是三个任务均分100ms的时间片。算法设计也较简单，只需在任意时刻统计三个任务的运行时间并对比，挑选运行时间最短的任务继续运行，确保每一时刻任务的运行时间尽可能相等即可。

现在来了一个需求说，任务A比较重要且工作量较大，希望给更多的运行时间，任务BC工作量小，少给点时间，假设希望的运行时间比例是2:1:1，则在100ms内算法应做到：

- 任务A运行50ms
- 任务B运行25ms
- 任务C运行25ms

那么算法在任意时刻要如何挑选下一个任务运行呢？假设100ms内的某一时刻，任务ABC均跑了5ms，则任务A离50ms只跑了10%，任务BC离25ms跑了20%，经过比对后确认，应该挑选任务A继续运行。可以看到，该过程已经不再是维护运行时间相等的方式了，而是要时刻维护好各任务与目标运行时间的差值比例。这就给算法带来了一定的复杂度。

那么vruntime是怎么做的？

仍然以上面场景为例，将三个任务的物理时间按比例伸缩：

- 任务A希望运行的物理时间为50ms，比例为0.5，伸缩后就是50 / 0.5 = 100ms
- 任务B希望运行的物理时间为25ms，比例为0.25，伸缩后就是25 / 0.25 = 100ms
- 任务C希望运行的物理时间为25ms，比例为0.25，伸缩后就是25 / 0.25 = 100ms

可以看到，伸缩后的时间相等。伸缩后的时间作为虚拟时间vruntime，和前面一样，算法只要在任意时刻挑选虚拟时间最短的任务运行即可。通过这种方式，就可以达到简化算法且兼顾不同任务运行比例的目的。

## vruntime计算

任务运行比例可以认为是一种权重weight。内核预定义了40个weight值，如下所示：

```c
// file: kernel/sched/core.c
const int sched_prio_to_weight[40] = {
 /* -20 */     88761,     71755,     56483,     46273,     36291,
 /* -15 */     29154,     23254,     18705,     14949,     11916,
 /* -10 */      9548,      7620,      6100,      4904,      3906,
 /*  -5 */      3121,      2501,      1991,      1586,      1277,
 /*   0 */      1024,       820,       655,       526,       423,
 /*   5 */       335,       272,       215,       172,       137,
 /*  10 */       110,        87,        70,        56,        45,
 /*  15 */        36,        29,        23,        18,        15,
};
```

这些值和nice值一一对应。比如`sched_prio_to_weight[0]`对应到nice = -20，`sched_prio_to_weight[39]`对应到nice = 19。[-20, 19]的nice区间，就是我们在打开top时常看到的nice列。值越大越nice（越好搞定），越不需要太多的时间片，因此`sched_prio_to_weight[]`权重也越小。

![](/images/调度：CFS调度vruntime/image-20250419202916459.png)

那么权重如何跟vruntime关联？通过如下公式：
$$
vruntime = delta\\_exec \times \frac{NICE\\_LOAD\\_0}{weight}
$$
其中：$delta\\_exec$是物理时间；$NICE\\_LOAD\\_0$是一个常量，等于1024，也就是前面`sched_prio_to_weight`数组中nice值为0时对应的值；$weight$根据人为设置的nice值查`sched_prio_to_weight`数组后输入。

可以看出该公式本质就是以$NICE\\_LOAD\\_0$为基准伸缩权重，再对物理时间加权，从而得到虚拟时间vruntime。

`sched_prio_to_weight`数组中的40个权重都是如何获取的？

其来自于权重对运行时间的控制预期：希望相邻的两个nice值运行时间能够相差10%。比如nice值从0变更到-5，则任务的运行时间应增加10%。有了该控制预期后，基于基准值1024，就可以很快换算出每个nice档位的权重。具体推导过程如下：

假设相邻权重分别为$w_1$和$w_2$，则10%的控制预期可通过如下公式呈现：
$$
\frac{w_1}{w_1 + w_2} - \frac{w_2}{w_1 + w_2} = 10\%
$$
我们将分母消除简化，可得：
$$
w_1 = 1.22\times w_2
$$
内核实际以1.25系数来计算，但差异不会太大，最终效果仍然是相邻档位具有10%的运行时间差异。可以简单验证下nice值从0到-5的权重变化：1024 / 1.25 = 819.2，`sched_prio_to_weight`数组给出的值为820，基本相等。

调度无时无刻不在发生，每发生一次调度，必然要伴随至少1次对当前vruntime时间的观测，而观测vruntime就涉及到前面的公式计算。我们知道，除法、浮点运算相比整型运算开销不小，因此这里仍然要考虑一个问题：如何将公式中除法、浮点运算消除？

仍然是老套路：对上述公式先乘以$2^{32}$再右移32位，如下所示：
$$
vruntime = delta\\_exec \times \frac{NICE\\_LOAD\\_0 \times 2^{32}}{weight} >> 32
$$
weight值已经预定义在`sched_prio_to_weight`数组中，故可以将公式中$\frac{2^{32}}{weight}$提前计算，得到如下另一个预定义数组：

```c
// file: kernel/sched/core.c
const u32 sched_prio_to_wmult[40] = {
 /* -20 */     48388,     59856,     76040,     92818,    118348,
 /* -15 */    147320,    184698,    229616,    287308,    360437,
 /* -10 */    449829,    563644,    704093,    875809,   1099582,
 /*  -5 */   1376151,   1717300,   2157191,   2708050,   3363326,
 /*   0 */   4194304,   5237765,   6557202,   8165337,  10153587,
 /*   5 */  12820798,  15790321,  19976592,  24970740,  31350126,
 /*  10 */  39045157,  49367440,  61356676,  76695844,  95443717,
 /*  15 */ 119304647, 148102320, 186737708, 238609294, 286331153,
};
```

为此，后续nice值的调整只需通过查询`sched_prio_to_wmult`，然后分别与物理时间相乘，再右移32位，即可得到vruntime。过程不再涉及除法。

最后我们来看下内核的源码实现。

1、首先是nice值调整，触发`set_load_weight()`：

```c
// file: kernel/sched/core.c
void set_load_weight(struct task_struct *p, bool update_load)
{
	int prio = p->static_prio - MAX_RT_PRIO;  // 获取nice值档位
	struct load_weight lw;
	...
		lw.weight = scale_load(sched_prio_to_weight[prio]);  // 记录当前档位对应的weight
		lw.inv_weight = sched_prio_to_wmult[prio];
	...
	p->se.load = lw;
}
```

2、然后是在时钟`task_tick`时，获取当前物理时间，并触发vruntime计算：

```c
// file: kernel/sched/fair.c
task_tick_fair
    entity_tick
        update_curr
            delta_exec = update_curr_se(rq, curr);  // 获取任务运行了多少物理时间
            curr->vruntime += calc_delta_fair(delta_exec, curr);  // 转换为vruntime
```

3、具体计算在`calc_delta_fair()`中：

```c
// file: kernel/sched/fair.c
static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se)
{
	if (unlikely(se->load.weight != NICE_0_LOAD))
		delta = __calc_delta(delta, NICE_0_LOAD, &se->load);  // 计算vruntime

	return delta;  // 如果当前任务权重为NICE_0_LOAD时，则公式比值为1，没必要算了，直接返回物理时间
}

static u64 __calc_delta(u64 delta_exec, unsigned long weight, struct load_weight *lw)
{
	u64 fact = scale_load_down(weight);  // NICE_LOAD_0
	int shift = WMULT_SHIFT;
	...  // 省略了corner case处理
	fact = mul_u32_u32(fact, lw->inv_weight);  // NICE_LOAD_0 * inv_weight
	return mul_u64_u32_shr(delta_exec, fact, shift);  // delta * (NICE_LOAD_0 * inv_weight) >> shift，和前面公式一致
}
```

以上就是vruntime的设计以及计算全过程。
