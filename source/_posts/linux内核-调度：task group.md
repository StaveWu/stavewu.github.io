---
title: 调度：task group
date: 2025/04/13 00:00:00
cover: /images/covers/03-computer.jpg
thumbnail: /images/covers/03-computer.jpg
toc: true
categories: 
 - linux内核
tags:
 - 调度
---

task group，即所谓的任务组调度，旨在解决指定的一组任务如何做CPU带宽控制的问题。

为什么需要对一组任务做带宽控制？或者说什么场景需要这种能力？

<!-- more -->

典型的应用就是容器场景，不同业务的容器运行在同一物理机上，通过CPU带宽控制可实现一定程度的容器之间的资源隔离性。有了带宽控制，我们也可以对不同租户基于带宽进行差异化收费。。。

了解其背景后，我们再来看下task group是怎么实现带宽控制的。

## 数据结构

在前面介绍[调度框架](/linux内核-调度：调度框架/)的时候，我们已经对task group的数据结构有了初步的了解，这里再简单讲下。

![](/images/调度：taskgroup/image-20250406133800134.png)

已知一个task group通常具有多个任务，每个任务可能运行在不同的cpu上，为了便于管理，在`struct task_group`中分别声明了两个变量：`**se`和`**cfs_rq`，二者本质上都是数组，根据cpu index可以追踪到对应cpu上属于该task group的se和cfs_rq，比如：`*se[0]`对应的就是cpu0上的se。这两个变量的具体作用需切换到单cpu视角上来看。

一个cpu上可能存在多个任务正在运行，其中这些任务有一部分属于某个task group，那么对于这一部分任务，为便于追踪，避免低效的任务遍历，调度设计上使用了一个私有的cfs_rq容纳，这样只要通过`**cfs_rq[i]`就可以pick出该task group在某cpu上的所有任务子集了。

但现在又面临了一个问题：已知cpu上的调度是通过一个rq队列维护，现在新增了一个私有的cfs_rq队列，要如何参与到该cpu的调度里去？这就引入了一个特殊的se：group se。该se作为私有cfs_rq的代表，加入到cpu调度队列。当该group se被pick时，继续下钻到私有cfs_rq内pick；当进行vruntime统计时，group se汇总私有cfs_rq内的所有任务vruntime并上报。这样就完成了调度操作的闭环。

group se不需要新定义，只需对普通se做一个小小的扩展，使其能够追踪到task group在该cpu上私有的cfs_rq。对于vruntime等统计信息，普通se本身就已携带，故无需改动。另外，为了方便私有cfs_rq内的任务能够快速追踪到group se，定义了一个`parent`字段，在后面我们也将看到该字段是用于带宽控制遍历的一个关键。完整的group se扩展如下：

```c
// file: include/linux/sched.h
struct sched_entity {
    #ifdef CONFIG_FAIR_GROUP_SCHED
        int				depth;
        struct sched_entity		*parent;  // 指向group se
        /* rq on which this entity is (to be) queued: */
        struct cfs_rq			*cfs_rq;
        /* rq "owned" by this entity/group: */
        struct cfs_rq			*my_q;  // group se字段，指向私有cfs_rq
        /* cached value of my_q->h_nr_running */
        unsigned long			runnable_weight;
    #endif
}
```

pick任务时顺着`my->q`字段不断往下追溯，直到找到普通的se（也就是具体的任务）执行：

```c
// file: kernel/sched/fair.c
pick_next_task_fair
    pick_task_fair
        do {
            se = pick_next_entity(rq, cfs_rq);
            cfs_rq = group_cfs_rq(se);  // return grp->my_q;
        } while (cfs_rq);
```

最后，我们将所有cpu上的group se汇总，就得到了前面所述task group的`**se`数组。

## cgroup带宽控制参数

带宽控制是task group的主要课题。因为task group对应的是cgroup的CPU子系统，所以本节将从cgroup的带宽控制参数切入，先对带宽控制能力有一个直观感受。

带宽控制的预期应该是什么样的？

以下图为例，假设一个任务运行完成需要200ms，如果不受带宽限制，则将满载运行；如果做了带宽控制，则将会是走走停停：

![](/images/调度：taskgroup/image-20250412123826717.png)

为了达成这样的效果，cgroup引入了cpu.max文件。

### cpu.max

cpu.max中有两个值，分别对应到quota和period，单位us，表示一个period周期内该cgroup最多可使用quota的时间。上图中period即为100ms，quota为40ms。

> cgroup分v1和v2版本，cpu.max是v2版本的文件，对应到v1里的cpu.cfs_period_us和cpu.cfs_quota_us。其他详细差异可以查询[该文档](https://www.alibabacloud.com/help/zh/alinux/support/differences-between-cgroup-v1-and-cgroup-v2#921d08df2c654)。

如果设置quota / period = 0.5，则表示当前cgroup可用cpu核为0.5个，在cgroup满载时top看到的占用率会是50%；如果设置quota / period = 2，则表示可用的cpu核为2个，在cgroup满载时top看到的占用率将会是200%。一个cgroup里如果有多个任务满载运行，则这些任务将平分50%或200%的cpu占用率。

quota和period分别对应到task group中的quota和period属性，和带宽控制相关的参数统一由`struct cfs_bandwidth`组织：

```c
// file: kernel/sched/core.c
{
    .name = "max",
    .flags = CFTYPE_NOT_ON_ROOT,
    .seq_show = cpu_max_show,
    .write = cpu_max_write,
},

// cat cpu.max文件时从task group中的cfs_bandwidth读取
cpu_max_show
    tg_get_cfs_period
        tg->cfs_bandwidth.period  // tg为task group
    tg_get_cfs_quota
        tg->cfs_bandwidth.quota
```

task group和cfs_bandwidth数据结构如下：

```c
// file: kernel/sched/sched.h
struct task_group {
    struct cgroup_subsys_state css;  // 对应cgroup做了扩展
    struct cfs_bandwidth	cfs_bandwidth;  // 带宽控制
};

struct cfs_bandwidth {
    ktime_t			period;  // 周期时长
    u64			quota;  // 单周期可用时间配额
};
```

### cpu.weight和cpu.weight.nice

cpu.weight和cpu.weight.nice用于调整cgroup的可用cpu权重。该权重在以下场景不会生效：

- 系统中只有1个cgroup时
- 系统尚未达到100%满载时

只有在系统达到100%满载时，task group才会基于权重考虑应该给各个cgroup分配多少比例的cpu。举个简单的例子：

假设系统有8个cpu，系统中有两个cgroup，cpu.weight分别配置为100和300，quota/period分别配置8个核，现在对两个cgroup进行满压（比如跑8个stress任务：`stress -c 8`），这种场景下，将看到cgroup之间的cpu比值将为1:3。

而如果我们将quota/period配置为4个核，cpu.weight权重不变，则两个cgroup满压时，二者的cpu比值则为1:1。因为在考虑权重前，两个cgroup的可用cpu先被quota/period受限，二者可用核数相加未超过系统总cpu数。

因此，触发cpu.weight的条件可归纳为：
$$
\sum_{i=1}^{N}\frac{quota_i}{period_i} > M
$$
其中N为cgroup数量，M为系统可用cpu核数。



cpu.weight.nice值和cpu.weight是联动的：

```bash
root@ubuntu-server:/sys/fs/cgroup/test# cat cpu.weight
100
root@ubuntu-server:/sys/fs/cgroup/test# cat cpu.weight.nice
0
root@ubuntu-server:/sys/fs/cgroup/test# echo 300 > cpu.weight
root@ubuntu-server:/sys/fs/cgroup/test# cat cpu.weight
300
root@ubuntu-server:/sys/fs/cgroup/test# cat cpu.weight.nice 
-5
```

cpu.weight.nice实际上是基于cpu.weight寻找最匹配的nice值的结果。



cpu.weight和cpu.weight.nice值根据task group的shares属性读取：

```c
// file: kernel/sched/core.c
{
    .name = "weight.nice",
    .flags = CFTYPE_NOT_ON_ROOT,
    .read_s64 = cpu_weight_nice_read_s64,
    .write_s64 = cpu_weight_nice_write_s64,
},

static s64 cpu_weight_nice_read_s64(struct cgroup_subsys_state *css,
				    struct cftype *cft)
{
    unsigned long weight = tg_weight(css_tg(css));  // return scale_load_down(tg->shares);
    int last_delta = INT_MAX;
    int prio, delta;

    /* find the closest nice value to the current weight */
    for (prio = 0; prio < ARRAY_SIZE(sched_prio_to_weight); prio++) {  // 基于weight寻找最匹配的nice值的过程
        delta = abs(sched_prio_to_weight[prio] - weight);
        if (delta >= last_delta)
            break;
        last_delta = delta;
    }

    return PRIO_TO_NICE(prio - 1 + MAX_RT_PRIO);
}
```

### cpu.max.burst

cpu.max.burst是指cgroup在单个period周期内除了quota配额外可预支未来多少时间。该参数的引入初衷是为了解决业务存在不定期突发请求且业务时延敏感的问题。这种业务通常在很长的一段时间内空闲，quota充足，但又存在不定期的突发，单个period的quota很快被耗尽，触发带宽控制，导致请求需等到下一个period重新分配quota时才能够继续运行，影响了请求时延。有了burst后，就容许请求在单period内运行超出quota的时间，确保请求在单period内处理完毕，无需等待。

burst参数对应到task group内的burst字段：

```c
struct cfs_bandwidth {
    ...
    u64			burst;
};
```

实际在充值时，burst基于如下规则分配给cgroup可用时间：

```c
__refill_cfs_bandwidth_runtime
    cfs_b->runtime += cfs_b->quota;
    cfs_b->runtime = min(cfs_b->runtime, cfs_b->quota + cfs_b->burst);  // 在quota配额上多增加了burst时间
```



## quota消耗、充值和再分配

### quota消耗

quota消耗的调用栈如下所示：

- 调用入口有很多，可以认为只要任务在运行，就会定期检查和消耗quota；
- 多个调用入口最终都汇聚到`__assign_cfs_rq_runtime`函数，该函数有一个`sched_cfs_bandwidth_slice()`入参，对应到`kernel.sched_cfs_bandwidth_slice_us`内核参数，表示每次任务耗尽本地可用时间时可补充多大的时间片。默认情况下该参数为5ms；
- `runtime_remaining`挂载在cfs_rq，该cfs_rq对应的即为task group在某cpu上的私有cfs_rq，说明`runtime_remaining`是一个per-cpu的，该cfs_rq里的任务运行时间均从`cfs_rq->runtime_remaining`扣。

```c
// file: kernel/sched/fair.c
update_curr/set_next_task_fair/enqueue_entity/... // 有很多入口
    account_cfs_rq_runtime
        __account_cfs_rq_runtime
            cfs_rq->runtime_remaining -= delta_exec;  // 计算剩余时间
            if (likely(cfs_rq->runtime_remaining > 0)) return;  // 如果还有剩余时间，则return
            assign_cfs_rq_runtime(cfs_rq)  // 否则，尝试从全局quota中申请
                __assign_cfs_rq_runtime(cfs_b, cfs_rq, sched_cfs_bandwidth_slice());  // 注意这里申请时，一次只申请sched_cfs_bandwidth_slice()的时间片，默认为5ms
```

`__assign_cfs_rq_runtime`函数实现如下：

```c
// file: kernel/sched/fair.c
static int __assign_cfs_rq_runtime(struct cfs_bandwidth *cfs_b,
				   struct cfs_rq *cfs_rq, u64 target_runtime)
{
    u64 min_amount, amount = 0;

    lockdep_assert_held(&cfs_b->lock);

    /* note: this is a positive sum as runtime_remaining <= 0 */
    min_amount = target_runtime - cfs_rq->runtime_remaining;  // 计算当前需补充多少时间

    if (cfs_b->quota == RUNTIME_INF)
        amount = min_amount;  // 如果quota没有限制，那就有多少给多少
    else {
        start_cfs_bandwidth(cfs_b);  // 确保period timer为激活状态，下文会展开

        if (cfs_b->runtime > 0) {  // 该runtime挂载在cfs_b，task group所属，说明是一个全局剩余可用时间，下文提到的quota充值就是对该runtime充值
            amount = min(cfs_b->runtime, min_amount);  // runtime能提供多少时间
            cfs_b->runtime -= amount;  // 开始扣除
            cfs_b->idle = 0;
        }
    }

    cfs_rq->runtime_remaining += amount;  // 补充到本地剩余可用时间runtime_remaining

    return cfs_rq->runtime_remaining > 0;
}
```

现在我们已经看到了两个属性：`cfs_b->runtime`和`cfs_rq->runtime_remaining`。总结下：

- `cfs_b->runtime`：对应到全局剩余可用时间，quota充值到这里
- `cfs_rq->runtime_remaining`：对应到本地per-cpu剩余可用时间，从全局`cfs_b->runtime`处按每次5ms（由`kernel.sched_cfs_bandwidth_slice_us`决定）的时间片申请。

### quota充值

quota充值调用栈如下所示。可以看到充值是通过period timer触发的，这也就符合前文所述的按照period周期限制quota时间的机制：

```c
// file: kernel/sched/fair.c
sched_cfs_period_timer
    do_sched_cfs_period_timer
        __refill_cfs_bandwidth_runtime
```

`__refill_cfs_bandwidth_runtime`函数实现如下：

```c
// file: kernel/sched/fair.c
void __refill_cfs_bandwidth_runtime(struct cfs_bandwidth *cfs_b)
{
    s64 runtime;

    if (unlikely(cfs_b->quota == RUNTIME_INF))
        return;  // quota没有限制，就没有充值一说，直接return

    cfs_b->runtime += cfs_b->quota;  // quota充值到全局runtime
    ...
    cfs_b->runtime = min(cfs_b->runtime, cfs_b->quota + cfs_b->burst);  // 前文已说明，能给burst的给点burst
}
```

### quota再分配

全局quota完成充值后，需重新分配给各个cpu上，这样挂起的任务才能继续运行。

分配函数为`distribute_cfs_runtime`，其实现如下所示。大体过程可以描述为：

- 遍历挂起列表，列表里每个cfs_rq透支的时间补齐
- 对已补齐时间的cfs_rq依次解挂

```c
// file: kernel/sched/fair.c
static bool distribute_cfs_runtime(struct cfs_bandwidth *cfs_b)
{
    int this_cpu = smp_processor_id();
    u64 runtime, remaining = 1;
    bool throttled = false;
    struct cfs_rq *cfs_rq, *tmp;
    struct rq_flags rf;
    struct rq *rq;
    LIST_HEAD(local_unthrottle);

    rcu_read_lock();
    list_for_each_entry_rcu(cfs_rq, &cfs_b->throttled_cfs_rq,
                throttled_list) {  // 遍历挂起列表
        rq = rq_of(cfs_rq);

        if (!remaining) {
            throttled = true;
            break;  // 时间都分配完了，退出循环
        }
        ...
        raw_spin_lock(&cfs_b->lock);
        runtime = -cfs_rq->runtime_remaining + 1;  // 该cfs_rq欠了多少时间
        if (runtime > cfs_b->runtime)
            runtime = cfs_b->runtime;  // 全局runtime只能给这么多了
        cfs_b->runtime -= runtime;  // 扣除可分配的时间
        remaining = cfs_b->runtime;  // 全局runtime还剩多少做下记录
        raw_spin_unlock(&cfs_b->lock);

        cfs_rq->runtime_remaining += runtime;  // 本地runtime_remaining补充可分配的时间

        /* we check whether we're throttled above */
        if (cfs_rq->runtime_remaining > 0) {  // 本地runtime_remaining回正了，说明有时间可用，将挂起的cfs_rq解挂
            if (cpu_of(rq) != this_cpu) {
                unthrottle_cfs_rq_async(cfs_rq);  // 如果cfs_rq不再本地cpu上，就发个消息异步解挂
            } else {  // 否则本地要解挂的cfs_rq加入到待解挂列表
                /*
                    * We currently only expect to be unthrottling
                    * a single cfs_rq locally.
                    */
                SCHED_WARN_ON(!list_empty(&local_unthrottle));
                list_add_tail(&cfs_rq->throttled_csd_list,
                            &local_unthrottle);
            }
        } else {
            throttled = true;
        }

next:
        rq_unlock_irqrestore(rq, &rf);
    }

    list_for_each_entry_safe(cfs_rq, tmp, &local_unthrottle,
                    throttled_csd_list) {  // 遍历待解挂列表
        struct rq *rq = rq_of(cfs_rq);

        rq_lock_irqsave(rq, &rf);

        list_del_init(&cfs_rq->throttled_csd_list);  // 从全局task group的挂起列表中删除

        if (cfs_rq_throttled(cfs_rq))
            unthrottle_cfs_rq(cfs_rq);  // 开始解挂

        rq_unlock_irqrestore(rq, &rf);
    }
    ...
}
```

## throttle和unthrottle

### throttle挂起

当全局quota耗尽时，表示task group在本周期内没有可用cpu，因此需将task group内的任务挂起不执行，达成throttle的效果。

具体实现见`throttle_cfs_rq`函数：

```c
// file: kernel/sched/fair.c
static bool throttle_cfs_rq(struct cfs_rq *cfs_rq)
{
    se = cfs_rq->tg->se[cpu_of(rq_of(cfs_rq))];  // 找到task group在该cpu上的group se

    // 遍历要出队的group se
    for_each_sched_entity(se) {  // 沿着group se->parent向上遍历
        struct cfs_rq *qcfs_rq = cfs_rq_of(se);  // 取出当前group se所在的cfs_rq队列

        /* throttled entity or throttle-on-deactivate */
        if (!se->on_rq)
            goto done;  // 如果se已经不在运行队列里，说明已经是挂起状态，退出循环

        dequeue_entity(qcfs_rq, se, flags);  // 否则出队，不参与运行
        ...
    }

    // 遍历剩下不用出队的se，这些se需要更新相关统计信息
    for_each_sched_entity(se) {
        struct cfs_rq *qcfs_rq = cfs_rq_of(se);
        /* throttled entity or throttle-on-deactivate */
        if (!se->on_rq)
            goto done;

        update_load_avg(qcfs_rq, se, 0);
        ...
    }
done:
    cfs_rq->throttled = 1;  // 标记已挂起
}
```

什么时候会触发挂起？

```c
// 1. 任务入队时检查
enqueue_entity
    check_enqueue_throttle
        throttle_cfs_rq

// 2. pick下一个任务时
pick_next_task_fair
    pick_task_fair
        check_cfs_rq_runtime
            throttle_cfs_rq

// 3. 统计前一个任务时
put_prev_task_fair
    put_prev_entity
        check_cfs_rq_runtime
            throttle_cfs_rq
```



### unthrottle解挂

解除挂起的流程基本和前面挂起流程反着来，具体实现位于`unthrottle_cfs_rq`：

```c
void unthrottle_cfs_rq(struct cfs_rq *cfs_rq)
{
    se = cfs_rq->tg->se[cpu_of(rq)];  // 取出本cpu上的group se
    cfs_rq->throttled = 0;  // 去掉挂起标记
    // 遍历group se及其parent
    for_each_sched_entity(se) {
        struct cfs_rq *qcfs_rq = cfs_rq_of(se);  // 取出se所在队列
        if (se->on_rq)
            break;  // 如果se已在运行，说明已经解除挂起了，退出循环
        enqueue_entity(qcfs_rq, se, ENQUEUE_WAKEUP);  // 否则入队
    }
    // 遍历剩下的se parent，更新统计信息
    for_each_sched_entity(se) {
        struct cfs_rq *qcfs_rq = cfs_rq_of(se);
        update_load_avg(qcfs_rq, se, UPDATE_TG);
        ...
    }
    ...
}
```

什么时候会发解挂？

```c
// 1. 不再限制带宽时
destroy_cfs_bandwidth
    __cfsb_csd_unthrottle

// 2. quota再分配时
distribute_cfs_runtime
    __unthrottle_cfs_rq_async
        unthrottle_cfs_rq
```



## period timer和slack timer

### period timer

前面已提到period timer决定了什么时间充值quota，具体timer回调函数实现如下：

```c
// file: kernel/sched/fair.c
sched_cfs_period_timer
    for (;;) {
        overrun = hrtimer_forward_now(timer, cfs_b->period);  // 尝试将过期时间推迟1个周期
        if (!overrun)
            break;  // 没能推迟成功，说明当前now还未过期，仍在当前周期内，退出循环
        idle = do_sched_cfs_period_timer(cfs_b, overrun, flags);  // 处理充值和再分配quota事项
```

在“quota消耗”章节中，我们看到为本地runtime_remaining充值时有一个`start_cfs_bandwidth`调用，该调用即为确保period timer总是激活：

```c
void start_cfs_bandwidth(struct cfs_bandwidth *cfs_b)
{
    lockdep_assert_held(&cfs_b->lock);

    if (cfs_b->period_active)
        return;  // 已激活就直接return

    cfs_b->period_active = 1;
    hrtimer_forward_now(&cfs_b->period_timer, cfs_b->period);  // 过期时间尝试推迟1个period周期
    hrtimer_start_expires(&cfs_b->period_timer, HRTIMER_MODE_ABS_PINNED);  // 开始计时
}
```

### slack timer

period timer已经能够周期性充值了，为什么还需要另一个slack timer？

考虑这样的一个场景：

CPU0从全局quota申请了5ms，未消耗完，有剩余；CPU1也从全局quota申请了5ms，但由于全局quota耗尽，CPU1只申请到了1ms，用完且还不够（此时CPU1上的任务被unthrottle挂起）。时间片是宝贵的，为了最大化利用，尝试让CPU0将剩余时间归还到全局quota，然后CPU1再申请，从而榨干最后一点时间。

为了解决这样的场景，引入了slack timer。

slack timer的触发调用栈如下所示：

```c
// file: kernel/sched/fair.c
dequeue_entity
    return_cfs_rq_runtime  // 归还剩余时间
        __return_cfs_rq_runtime
            start_cfs_slack_bandwidth
```

具体看下`__return_cfs_rq_runtime`和`start_cfs_slack_bandwidth`的实现。

`__return_cfs_rq_runtime`：

```c
// file: kernel/sched/fair.c
static void __return_cfs_rq_runtime(struct cfs_rq *cfs_rq)
{
    struct cfs_bandwidth *cfs_b = tg_cfs_bandwidth(cfs_rq->tg);
    s64 slack_runtime = cfs_rq->runtime_remaining - min_cfs_rq_runtime;

    if (slack_runtime <= 0)
        return;  // 无法归还，则退出

    raw_spin_lock(&cfs_b->lock);
    if (cfs_b->quota != RUNTIME_INF) {
        cfs_b->runtime += slack_runtime;  // 充值到全局runtime内

        /* we are under rq->lock, defer unthrottling using a timer */
        if (cfs_b->runtime > sched_cfs_bandwidth_slice() &&
            !list_empty(&cfs_b->throttled_cfs_rq))
            start_cfs_slack_bandwidth(cfs_b);  // 触发slack timer
    }
    raw_spin_unlock(&cfs_b->lock);

    /* even if it's not valid for return we don't want to try again */
    cfs_rq->runtime_remaining -= slack_runtime;  // 已归还了，本地剩余时间要扣除
}
```

`start_cfs_slack_bandwidth`：

```c
// file: kernel/sched/fair.c
static void start_cfs_slack_bandwidth(struct cfs_bandwidth *cfs_b)
{
    u64 min_left = cfs_bandwidth_slack_period + min_bandwidth_expiration;  // slack timer触发的最小时间间隔，一般为7ms（本意是不要急着触发，再等等）

    /* if there's a quota refresh soon don't bother with slack */
    if (runtime_refresh_within(cfs_b, min_left))
        return;  // period timer马上就要充值了，那就没必要触发slack timer

    /* don't push forwards an existing deferred unthrottle */
    if (cfs_b->slack_started)
        return;  // slack已经在执行，return
    cfs_b->slack_started = true;

    hrtimer_start(&cfs_b->slack_timer,
            ns_to_ktime(cfs_bandwidth_slack_period),
            HRTIMER_MODE_REL);  // 触发slack timer
}
```

那slack timer具体做什么事呢？其回调函数为`sched_cfs_slack_timer`，主要是调用`distribute_cfs_runtime`对全局可用时间进行再分配：

```c
// file: kernel/sched/fair.c
sched_cfs_slack_timer
    do_sched_cfs_slack_timer
        distribute_cfs_runtime
```

## 实际案例

以下是一个实际案例，清晰展示了前文所述的带宽控制过程。

假设系统有2个cpu，quota为20ms，period为100ms，下图呈现了全局quota（对应`cfs_b->runtime`）和per-cpu quota（对应`cfs_rq->runtime_remaining`）的剩余情况：

- CPU1申请了5ms，运行worker1，刚好用完，worker1主动休眠；
- CPU2申请了5ms，运行worker2，刚好用完，worker2主动休眠；
- CPU1申请了5ms，运行worker1，只跑了1ms，剩余4ms未使用；
- 过了7ms后，slack timer触发，CPU1将剩余4ms的归还（这里只归还了3ms，自己留了1ms）
- CPU2申请了5ms，运行worker2，用完，还不够；
- CPU2再次想要申请5ms，但此时全局quota只剩3ms了，所以CPU2只申请到了3ms，用完；
- 至此CPU2上的任务被throttle挂起；
- 而CPU1上因为保留了1ms，所以未被挂起。

![](/images/调度：taskgroup/wmremove-transformed.jpeg)

以上就是整个task group带宽控制的全过程。

## 参考

1. [组调度和带宽控制](https://zhuanlan.zhihu.com/p/350331074)

