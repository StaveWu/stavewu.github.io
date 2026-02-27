---
title: watchdog：soft lockup
date: 2026/02/27 00:00:00
cover: /images/covers/09-diary.jpg
thumbnail: /images/covers/09-diary.jpg
toc: true
categories:
 - linux内核
tags:
 - watchdog
---

soft lockup是内核常见的维测手段之一，了解soft lockup的机制有助于在看到该类报错后明确定位方向。本文要搞清楚：

- soft lockup是什么
- soft lockup检测原理

<!-- more -->

## soft lockup是什么

soft lockup是指内核长时间占用某CPU，导致该CPU上其他任务无法被调度执行的情况

一种简单的soft lockup复现：

```c
// 启动一个内核线程，做如下动作：
while (1) {
	do_something();  // 长时间不释放CPU，触发soft lockup
}
```



## soft lockup检测原理

1、定义timer：kernel/watchdog.c

```c
static DEFINE_PER_CPU(struct hrtimer, watchdog_hrtimer);
```

注意每CPU上都有一个timer。

2、启用timer：kernel/watchdog.c

```c
static void watchdog_enable(unsigned int cpu)
{
	struct hrtimer *hrtimer = this_cpu_ptr(&watchdog_hrtimer);

	hrtimer_setup(hrtimer, watchdog_timer_fn, CLOCK_MONOTONIC, HRTIMER_MODE_REL_HARD);  // 回调watchdog_timer_fn
	hrtimer_start(hrtimer, ns_to_ktime(sample_period),  // 触发周期sample_period
		      HRTIMER_MODE_REL_PINNED_HARD);
}
```

3、回调执行：

```c
static enum hrtimer_restart watchdog_timer_fn(struct hrtimer *hrtimer)
{
	hrtimer_forward_now(hrtimer, ns_to_ktime(sample_period));  // 设置下一次过期时间

	now = get_timestamp();
	period_ts = READ_ONCE(*this_cpu_ptr(&watchdog_report_ts));  // 获取上一次report时间
	touch_ts = __this_cpu_read(watchdog_touch_ts);  // 获取前一次喂狗时间
	duration = is_softlockup(touch_ts, period_ts, now);
 	if (unlikely(duration)) {
		update_report_ts();  // 要report了
		pr_emerg("BUG: soft lockup - CPU#%d stuck for %us! [%s:%d]\n",
			smp_processor_id(), duration,
			current->comm, task_pid_nr(current));
	}

	return HRTIMER_RESTART;
}
```

