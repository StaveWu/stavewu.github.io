---
title: watchdog：hard lockup
date: 2025/06/21 00:00:00
cover: /images/covers/08-office.jpg
thumbnail: /images/covers/08-office.jpg
toc: true
categories:
 - linux内核
tags:
 - watchdog
---

hard lockup是内核常见的维测手段之一，了解hard lockup的机制有助于在看到该类报错后明确定位方向。本文要搞清楚：

- hard lockup是什么？
- hard lockup是怎么拉起的？
- hard lockup是怎么监控的？

<!-- more -->

## hard lockup是什么？

hard lockup指CPU长时间（默认10秒）不响应中断，包括不可屏蔽中断（NMI）。

一种简单的hard lockup复现：

```c
// 内核函数里执行
local_irq_disable();
mdelay(100000);  // 100s，关中断时间过长，引发hard lockup
local_irq_enable();
```

具体hard lockup判定时长由watchdog的几个参数决定（不只是hard lockup，soft lockup也受这些参数控制）：

```bash
root@ubuntu-server:/proc/sys/kernel# ll watchdog*
-rw-r--r-- 1 root root 0 Jun 21 03:17 watchdog
-rw-r--r-- 1 root root 0 Jun 21 03:17 watchdog_cpumask
-rw-r--r-- 1 root root 0 Jun 21 03:17 watchdog_thresh
```

其中，watchdog是使能开关，watchdog_cpumask指要使能哪些cpu上的watchdog，watchdog_thresh是喂狗超时阈值。



## hard lockup是怎么拉起的？

hard lockup在内核启动时通过如下调用链拉起：

```bash
start_kernel
rest_init
kernel_init
kernel_init_freeable
lockup_detector_init
	watchdog_hardlockup_probe()
	lockup_detector_setup()
```



## hard lockup是怎么监控的？

hard lockup有两种实现，1个是perf，1个是buddy，这里用了c语言的weak技巧：具体为在kernel/watchdog.c中定义一个weak函数：

```c
int __weak __init watchdog_hardlockup_probe(void)
{
	return -ENODEV;
}
```

然后在kernel/watchdog_perf.c和kernel/watchdog_buddy.c中分别定义相同函数的具体实现：

```c
int __init watchdog_hardlockup_probe(void)
{
	int ret;

	if (!arch_perf_nmi_is_available())
		return -ENODEV;

	ret = hardlockup_detector_event_create();

	if (ret) {
		pr_info("Perf NMI watchdog permanently disabled\n");
	} else {
		perf_event_release_kernel(this_cpu_read(watchdog_ev));
		this_cpu_write(watchdog_ev, NULL);
	}
	return ret;
}
```

最后在Makefile中，通过选项控制哪个文件的编译：kernel/Makefile

```makefile
obj-$(CONFIG_LOCKUP_DETECTOR) += watchdog.o
obj-$(CONFIG_HARDLOCKUP_DETECTOR_BUDDY) += watchdog_buddy.o
obj-$(CONFIG_HARDLOCKUP_DETECTOR_PERF) += watchdog_perf.o
```

在perf实现里，hard lockup通过watchdog_hardlockup_probe使能，具体为创建1个perf event事件，并为该事件设置一个采样周期，当周期到时，事件overflow，触发1个中断，在该中断内处理lockup检测。

事件定义如下：kernel/watchdog_perf.c

```c
static struct perf_event_attr wd_hw_attr = {
	.type		= PERF_TYPE_HARDWARE,
	.config		= PERF_COUNT_HW_CPU_CYCLES,
	.size		= sizeof(struct perf_event_attr),
	.pinned		= 1,
	.disabled	= 1,
};
```

事件使能：

```c
static int hardlockup_detector_event_create(void)
{
	unsigned int cpu;
	struct perf_event_attr *wd_attr;
	struct perf_event *evt;

	/*
	 * Preemption is not disabled because memory will be allocated.
	 * Ensure CPU-locality by calling this in per-CPU kthread.
	 */
	WARN_ON(!is_percpu_thread());
	cpu = raw_smp_processor_id();  // 获取cpu id
	wd_attr = &wd_hw_attr;  // 获取事件属性
	wd_attr->sample_period = hw_nmi_get_sample_period(watchdog_thresh);  // 设置事件的采样周期为watchdog_thresh

	/* Try to register using hardware perf events */
	evt = perf_event_create_kernel_counter(wd_attr, cpu, NULL,
					       watchdog_overflow_callback, NULL);  // 在该cpu上注册事件，其中watchdog_overflow_callback为事件overflow时触发的回调
    ...
	this_cpu_write(watchdog_ev, evt);
	return 0;
}
```

中断回调如下：

```c
/* Callback function for perf event subsystem */
static void watchdog_overflow_callback(struct perf_event *event,
				       struct perf_sample_data *data,
				       struct pt_regs *regs)
{
	/* Ensure the watchdog never gets throttled */
	event->hw.interrupts = 0;

	if (!watchdog_check_timestamp())
		return;

	watchdog_hardlockup_check(smp_processor_id(), regs);
}
```

watchdog_check_timestamp检查当前时刻距离上一时刻是否超过了watchdog_thresh * 5 / 2的时间间隔，如果不是，则跳过当前时刻的lockup检查。这么做可能是因为watchdog_overflow_callback调用频率会高于watchdog_thresh * 5 / 2，然后用watchdog_check_timestamp限频？

之后是watchdog_hardlockup_check检查了。其判断是否lockup的依据是通过per cpu的watchdog_hardlockup_touched布尔量。

如果watchdog_hardlockup_touched为true，则lockup检测将该变量重置为false。那该变量是什么时候会置为true？由内核内各处业务逻辑决定。watchdog提供了touch_nmi_watchdog函数，内核中任何有可能触发lockup的地方主动调用该函数即可避免lockup上报，该动作俗称“喂狗”。

如果watchdog_hardlockup_touched为false，则lockup检测还将做第二层判断来对lockup定性。具体通过hrtimer_interrupts和hrtimer_interrupts_saved变量，前者为当前时刻的中断计数，后者为上一时刻的中断计数。如果二者相等，则说明该cpu被关闭了中断，此时可以判断为hard lockup。

```c
static bool is_hardlockup(unsigned int cpu)
{
	int hrint = atomic_read(&per_cpu(hrtimer_interrupts, cpu));

	if (per_cpu(hrtimer_interrupts_saved, cpu) == hrint)
		return true;

	/*
	 * NOTE: we don't need any fancy atomic_t or READ_ONCE/WRITE_ONCE
	 * for hrtimer_interrupts_saved. hrtimer_interrupts_saved is
	 * written/read by a single CPU.
	 */
	per_cpu(hrtimer_interrupts_saved, cpu) = hrint;

	return false;
}

void watchdog_hardlockup_check(unsigned int cpu, struct pt_regs *regs)
{
	if (per_cpu(watchdog_hardlockup_touched, cpu)) {
		per_cpu(watchdog_hardlockup_touched, cpu) = false;
		return;
	}

	/*
	 * Check for a hardlockup by making sure the CPU's timer
	 * interrupt is incrementing. The timer interrupt should have
	 * fired multiple times before we overflow'd. If it hasn't
	 * then this is a good indication the cpu is stuck
	 */
	if (is_hardlockup(cpu)) {
		...
	} else {
		per_cpu(watchdog_hardlockup_warned, cpu) = false;
	}
}
```

最后看下hrtimer_interrupts，其更新是通过一个timer来做，该timer的周期为watchdog_thresh * 2 / 5：

```c
static void watchdog_enable(unsigned int cpu)
{
    ...
	hrtimer->function = watchdog_timer_fn;
	hrtimer_start(hrtimer, ns_to_ktime(sample_period),
		      HRTIMER_MODE_REL_PINNED_HARD);
}

/* watchdog kicker functions */
static enum hrtimer_restart watchdog_timer_fn(struct hrtimer *hrtimer)
{
    ...
	watchdog_hardlockup_kick();  // hrtimer_interrupts加1
    ...
}
```

## 总结

touch_nmi_watchdog、timer、perf event的协同关系总结如下：

- 各模块业务根据自身的执行上下文判断喂狗频率，通过touch_nmi_watchdog，最终效果是watchdog_hardlockup_touched被置为true
- timer按照watchdog_thresh * 2 / 5的频率触发并对hrtimer_interrupts加1
- perf event按照watchdog_thresh的频率触发并检测watchdog_hardlockup_touched和hrtimer_interrupts：
  - 如果watchdog_hardlockup_touched为true，则复位为false并return；
  - 如果watchdog_hardlockup_touched为false，则检查hrtimer_interrupts是否和前一次perf event事件看到的值是否有递增，如无递增则认为hard lockup。

perf event这么做的本质是给业务方两次错过喂狗的机会，如果两次都错过，则在watchdog_thresh超时时触发hard lockup。