---
title: linux中断
date: 2025/01/04 00:00:00
toc: true
categories: 
 - 计算机组成
tags: 
 - 底层软件
---

本文目标：linux中断有哪些类别？分别用在什么场景？怎么使用？

<!-- more -->

中断有哪些类别？

- 硬中断
- 软中断

为了解决中断处理执行时间过长和中断丢失的问题，中断又分为：

- 上半部（top half，th）
- 下半部（bottom half，bh）

其中上半部是硬中断hardirq，下半部可以是softirq、tasklet、workqueue



## 硬中断

### hardirq

硬中断主要来自外围设备，比如每个处理器上的定时器

通过/proc/interrupts可以查看硬中断的运行情况：

```bash
root@ubuntu-server:~# cat /proc/interrupts 
            CPU0       CPU1       
   0:         14          0   IO-APIC    2-edge      timer
   1:          0        111   IO-APIC    1-edge      i8042
   8:          0          0   IO-APIC    8-edge      rtc0
   9:          0          0   IO-APIC    9-fasteoi   acpi
  12:        413        176   IO-APIC   12-edge      i8042
  14:          0          0   IO-APIC   14-edge      ata_piix
  15:          0          0   IO-APIC   15-edge      ata_piix
  16:       9378       7914   IO-APIC   16-fasteoi   vmwgfx, snd_ens1371
  17:     108849     192469   IO-APIC   17-fasteoi   ehci_hcd:usb1, ioc0
  18:        627       1362   IO-APIC   18-fasteoi   uhci_hcd:usb2
  19:      11143      61727   IO-APIC   19-fasteoi   ens33
  24:          0          0   PCI-MSI 344064-edge      PCIe PME, pciehp
  25:          0          0   PCI-MSI 346112-edge      PCIe PME, pciehp
  26:          0          0   PCI-MSI 348160-edge      PCIe PME, pciehp
```

不同种类的中断控制器的访问方法存在差异，linux通过irq_chip结构体统一：include/linux/irq.h

```c
struct irq_chip {
	const char	*name;  // 对应到/proc/interrupts内名称
	unsigned int	(*irq_startup)(struct irq_data *data);
	void		(*irq_shutdown)(struct irq_data *data);
	void		(*irq_enable)(struct irq_data *data);
	void		(*irq_disable)(struct irq_data *data);
```

ARM提供了标准的中断控制器GIC（generic interrupt controller），其irq_chip如下：drivers/irqchip/irq-gic.c

```c
static const struct irq_chip gic_chip = {
	.irq_mask		= gic_mask_irq,
	.irq_unmask		= gic_unmask_irq,
	.irq_eoi		= gic_eoi_irq,
	.irq_set_type		= gic_set_type,
	.irq_retrigger          = gic_retrigger,
	.irq_set_affinity	= gic_set_affinity,
	.ipi_send_mask		= gic_ipi_send_mask,
	.irq_get_irqchip_state	= gic_irq_get_irqchip_state,
	.irq_set_irqchip_state	= gic_irq_set_irqchip_state,
	.irq_print_chip		= gic_irq_print_chip,
	.flags			= IRQCHIP_SET_TYPE_MASKED |
				  IRQCHIP_SKIP_SET_WAKE |
				  IRQCHIP_MASK_ON_SUSPEND,
};
```

一个系统可能有多个中断控制器，且控制器之间可能存在级联关系，为了将硬件中断号映射到唯一的linux虚拟中断号，内核定义了中断域irq_domain，每个中断控制器有自己的中断域。

硬件中断号和linux中断号映射接口：include/linux/irqdomain.h

```c
static inline unsigned int irq_create_mapping(struct irq_domain *host, irq_hw_number_t hwirq);
static inline unsigned int irq_find_mapping(struct irq_domain *domain, irq_hw_number_t hwirq);
```

arm64在dts中描述硬件中断控制器信息，以timer为例：arch/arm64/boot/dts/arm/foundation-v8.dtsi

```bash
	timer {
		compatible = "arm,armv8-timer";
		interrupts = <GIC_PPI 13 (GIC_CPU_MASK_SIMPLE(4) | IRQ_TYPE_LEVEL_LOW)>,
			     <GIC_PPI 14 (GIC_CPU_MASK_SIMPLE(4) | IRQ_TYPE_LEVEL_LOW)>,
			     <GIC_PPI 11 (GIC_CPU_MASK_SIMPLE(4) | IRQ_TYPE_LEVEL_LOW)>,
			     <GIC_PPI 10 (GIC_CPU_MASK_SIMPLE(4) | IRQ_TYPE_LEVEL_LOW)>;
		clock-frequency = <100000000>;
	};
```

其中，GIC_PPI为中断类型；13为中断号；(GIC_CPU_MASK_SIMPLE(4) | IRQ_TYPE_LEVEL_LOW)为中断触发方式。

GIC中断类型有哪些：

- 软件生成的中断（Software Generated Interrupt，SGI），中断号0~15，处理器间中断
- 私有外设中断（Private Peripheral Interrupt，PPI），中断号16~31，处理器私有的中断源
- 共享外设中断（Shared Peripheral Interrupt，SPI），中断号32~1020，中断控制器可以将中断转发给多个处理器
- 局部特定外设中断（Locality-specific Peripheral Interrupt，LPI），基于消息的中断

中断触发方式：边沿触发、电平触发

linux硬中断处理函数有2层，第1层是handle_irq，根据中断类型设置对应的处理函数；第2层是irq_desc.action，一般是设备驱动程序注册

include/linux/irqdesc.h

```c
struct irq_desc {
    ...
	irq_flow_handler_t	handle_irq;
	struct irqaction	*action;	/* IRQ action list */

// 链表组织多个action
struct irqaction {
	irq_handler_t		handler;
	void			*dev_id;
	struct irqaction	*next;
```

硬中断怎么用？

1、注册硬中断：include/linux/interrupt.h

```c
static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
	    const char *name, void *dev)  // irq：linux中断号，handler：处理函数
```

2、执行硬中断：以GIC v2控制器为例

```bash
el0_interrupt
handle_arch_irq
gic_handle_irq  # 通过set_handle_irq注册，这里将handle_arch_irq注册为gic_handle_irq
generic_handle_domain_irq
	return handle_irq_desc(irq_resolve_mapping(domain, hwirq));
generic_handle_irq_desc
	desc->handle_irq(desc);
handle_fasteoi_irq  # 通过gic_irq_domain_map注册，如果是共享外设中断，则将handle_irq注册为handle_fasteoi_irq
handle_irq_event
__handle_irq_event_percpu
    for_each_action_of_desc(desc, action) {  # 遍历action并调用
        res = action->handler(irq, action->dev_id);
    }
```



## 软中断

### softirq

softirq类别：include/linux/interrupt.h

```c
enum
{
	HI_SOFTIRQ=0,
	TIMER_SOFTIRQ,
	NET_TX_SOFTIRQ,
	NET_RX_SOFTIRQ,
	BLOCK_SOFTIRQ,
	IRQ_POLL_SOFTIRQ,
	TASKLET_SOFTIRQ,
	SCHED_SOFTIRQ,
	HRTIMER_SOFTIRQ,
	RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */

	NR_SOFTIRQS
};
```

通过/proc/softirqs可以查看软中断的运行情况：

```bash
root@ubuntu-server:~# cat /proc/softirqs | awk '{print $1"\t"$2"\t"$3}'
CPU0    CPU1    CPU2
HI:     143     1
TIMER:  153720  89373
NET_TX: 21      827
NET_RX: 19078   71117
BLOCK:  203462  163429
IRQ_POLL:       0       0
TASKLET:        918     529
SCHED:  248027  200577
HRTIMER:        0       0
RCU:    416872  349341
```

softirq类别介绍：

- NET_TX_SOFTIRQ、NET_RX_SOFTIRQ：网络收发报文的软中断
- BLOCK_SOFTIRQ：块设备软中断

从这里可以看出，softirq是内核编译时就定义好的，运行时不能添加或删除。



softirq怎么用？

1、注册softirq：kernel/softirq.c

```c
void open_softirq(int nr, void (*action)(void))
{
	softirq_vec[nr].action = action;
}
```

2、触发softirq：kernel/softirq.c

```c
void raise_softirq(unsigned int nr)
{
	unsigned long flags;

	local_irq_save(flags);
	raise_softirq_irqoff(nr);
	local_irq_restore(flags);
}
```

local_irq_save/restore是做什么用的？

- local_irq_save(flags)：将当前的中断状态保存到参数flags中，为后续恢复中断状态做准备。
- local_irq_restore(flags)：恢复中断状态
- local_irq_disable()：禁止中断
- local_irq_enable()：开启中断

以上接口仅能处理本处理器的中断，无法处理其他处理器的中断。local_irq_disable/enable不能嵌套使用，local_irq_save/restore可以。

以上接口也无法作用于不可屏蔽中断（NMI，Non Maskable Interrupt）

以上接口作用所有中断，还有一组仅作用于单个中断的接口：

```c
extern void enable_irq(unsigned int irq);
extern void disable_irq(unsigned int irq);
```



3、执行softirq：

- 通过中断处理程序执行
- 通过软中断线程执行
- 通过local_bh_enable函数开启softirq时执行

以下针对每一条路径进行说明：

通过中断处理程序执行：`irq_exit -> __irq_exit_rcu -> invoke_softirq -> __do_softirq -> handle_softirqs`

通过软中断线程执行：

每个处理器都有一个软中断线程

```bash
root@ubuntu-server:~# ps -ef | grep irq
root          13       2  0 09:09 ?        00:00:00 [ksoftirqd/0]
root          22       2  0 09:09 ?        00:00:03 [ksoftirqd/1]
```

线程执行run_ksoftirqd()：`run_ksoftirqd -> handle_softirqs`

kernel/softirq.c

```c
static struct smp_hotplug_thread softirq_threads = {
	.store			= &ksoftirqd,
	.thread_should_run	= ksoftirqd_should_run,
	.thread_fn		= run_ksoftirqd,
	.thread_comm		= "ksoftirqd/%u",
};

static void run_ksoftirqd(unsigned int cpu)
{
	ksoftirqd_run_begin();
	if (local_softirq_pending()) {
		/*
		 * We can safely run softirq on inline stack, as we are not deep
		 * in the task stack here.
		 */
		handle_softirqs(true);
		ksoftirqd_run_end();
		cond_resched();
		return;
	}
	ksoftirqd_run_end();
}
```

通过local_bh_enable函数开启软中断时执行：

`local_bh_enable -> __local_bh_enable_ip -> __do_softirq -> handle_softirqs`



以上3个softirq执行路径最终都调用了handle_softirqs，其实现为：

```c
static void handle_softirqs(bool ksirqd)
{
    ...
	pending = local_softirq_pending();  // 获取待处理中断位图
    ...

restart:
	/* Reset the pending bitmask before enabling irqs */
	set_softirq_pending(0);

	local_irq_enable();

	h = softirq_vec;

	while ((softirq_bit = ffs(pending))) {  // 从低到高扫描待处理位图
		...
		h->action();  // 执行中断处理函数
        ...
	}

	local_irq_disable();

	pending = local_softirq_pending();  // 如果软中断内又触发了软中断，则根据执行时长判断是否继续执行还是重新调度
	if (pending) {
		if (time_before(jiffies, end) && !need_resched() &&
		    --max_restart)  // 避免softirq占用太多CPU
			goto restart;

		wakeup_softirqd();
	}
    ...
}
```



### tasklet

tasklet是基于softirq扩展实现的，但tasklet和softirq又有区别：

- 可在运行时添加或删除
- 同一时刻只会在一个处理器上执行，不要求处理函数是可以重入的

tasklet数据结构：include/linux/interrupt.h

```c
struct tasklet_struct
{
	struct tasklet_struct *next;  // 单向链表
	unsigned long state;  // 0：没被调度，1<<TASKLET_STATE_SCHED：被调度，即将执行；1<<TASKLET_STATE_RUN：正在执行
	atomic_t count;  // 0：允许执行；非0值：禁止执行
	bool use_callback;
	union {
		void (*func)(unsigned long data);  // 处理函数
		void (*callback)(struct tasklet_struct *t);
	};
	unsigned long data;  // 处理函数的入参
};
```

tasklet链表：kernel/softirq.c

```c
struct tasklet_head {
	struct tasklet_struct *head;
	struct tasklet_struct **tail;
};
// 每个处理器上定义2条链表
static DEFINE_PER_CPU(struct tasklet_head, tasklet_vec);  // 低优先级tasklet链表
static DEFINE_PER_CPU(struct tasklet_head, tasklet_hi_vec);  // 高优先级tasklet链表
```

1、定义tasklet：

```c
// 静态方式，include/linux/interrupt.h
#define DECLARE_TASKLET(name, _callback)  // 定义并运行执行
#define DECLARE_TASKLET_DISABLED(name, _callback)  // 定义并禁止执行

// 动态方式，kernel/softirq.c
void tasklet_init(struct tasklet_struct *t,
		  void (*func)(unsigned long), unsigned long data)
```

2、注册tasklet（加入到链表尾部）：

```c
static inline void tasklet_schedule(struct tasklet_struct *t)
    // raise_softirq_irqoff(TASKLET_SOFTIRQ)
static inline void tasklet_hi_schedule(struct tasklet_struct *t)
    // raise_softirq_irqoff(HI_SOFTIRQ)
```

3、执行tasklet：

```c
static void tasklet_action_common(struct tasklet_head *tl_head,
				  unsigned int softirq_nr)
{
	struct tasklet_struct *list;

	local_irq_disable();
    // 把所有的tasklet移到list
	list = tl_head->head;
	tl_head->head = NULL;
	tl_head->tail = &tl_head->head;
	local_irq_enable();

	while (list) {  // 遍历list
		struct tasklet_struct *t = list;

		list = list->next;

		if (tasklet_trylock(t)) {  // 确保tasklet在同一时刻只有一个在执行
			if (!atomic_read(&t->count)) {
				if (tasklet_clear_sched(t)) {
					if (t->use_callback) {
						trace_tasklet_entry(t, t->callback);
						t->callback(t);
						trace_tasklet_exit(t, t->callback);
					} else {
						trace_tasklet_entry(t, t->func);
						t->func(t->data);  // 执行处理函数
						trace_tasklet_exit(t, t->func);
					}
				}
				tasklet_unlock(t);
				continue;
			}
			tasklet_unlock(t);
		}
        
        // 表示锁任务失败，将tasklet重新添加回tl_head
		local_irq_disable();
		t->next = NULL;
		*tl_head->tail = t;
		tl_head->tail = &t->next;
		__raise_softirq_irqoff(softirq_nr);
		local_irq_enable();
	}
}
```



### workqueue

workqueue和tasklet的区别：

- tasklet运行在softirq上下文中，而workqueue运行在内核进程上下文中，这说明wq不能像tasklet那样是原子的
- takslet永远运行在指定处理器，这是初始化时就确定了，而wq可以通过配置修改这种行为

workqueue可以通过以下命令查询：

```bash
root@ubuntu-server:~# ps -ef | grep kworker
root           8       2  0 09:09 ?        00:00:00 [kworker/0:0H-events_highpri]
root          24       2  0 09:09 ?        00:00:00 [kworker/1:0H-events_highpri]
root          91       2  0 09:09 ?        00:00:27 [kworker/0:1H-kblockd]
root         140       2  0 09:09 ?        00:00:27 [kworker/1:1H-kblockd]
root         155       2  0 09:09 ?        00:00:00 [kworker/u257:0-hci0]
root         702       2  0 09:10 ?        00:00:00 [kworker/u257:1-hci0]
root        7892       2  0 10:46 ?        00:00:11 [kworker/0:1-inode_switch_wbs]
root        7899       2  0 10:46 ?        00:00:09 [kworker/1:0-cgroup_destroy]
root        9259       2  0 12:27 ?        00:00:00 [kworker/u256:2-events_unbound]
root       11349       2  0 12:47 ?        00:00:04 [kworker/1:2-events]
root       11351       2  0 12:47 ?        00:00:06 [kworker/0:0-events]
root       11529       2  0 13:19 ?        00:00:00 [kworker/u256:1-events_power_efficient]
root       11534       2  0 13:25 ?        00:00:00 [kworker/u256:0-events_power_efficient]
```

1、定义work：include/linux/workqueue.h

```c
// 静态方式，n为变量名称，f为处理函数
#define DECLARE_WORK(n, f)
#define DECLARE_DELAYED_WORK(n, f)

// 动态方式，_work为工作项地址，类型work_struct，_func为处理函数，入参类型为work_struct
#define INIT_WORK(_work, _func)
#define INIT_DELAYED_WORK(_work, _func)
```

2、注册work：include/linux/workqueue.h

```c
// 注册到全局队列
static inline bool schedule_work(struct work_struct *work)
{
	return queue_work(system_wq, work);
}
static inline bool schedule_work_on(int cpu, struct work_struct *work)
{
	return queue_work_on(cpu, system_wq, work);
}

// 申请局部队列并注册
__printf(1, 4) struct workqueue_struct *
alloc_workqueue(const char *fmt, unsigned int flags, int max_active, ...);
```

其中，在申请队列时flags指定队列类型：

```c
enum wq_flags {
	WQ_BH			= 1 << 0, /* execute in bottom half (softirq) context */
	WQ_UNBOUND		= 1 << 1, /* not bound to any cpu */
	WQ_FREEZABLE		= 1 << 2, /* freeze during suspend */
	WQ_MEM_RECLAIM		= 1 << 3, /* may be used for memory reclaim */
	WQ_HIGHPRI		= 1 << 4, /* high priority */
	WQ_CPU_INTENSIVE	= 1 << 5, /* cpu intensive workqueue */
	WQ_SYSFS		= 1 << 6, /* visible in sysfs, see workqueue_sysfs_register() */
	WQ_POWER_EFFICIENT
}
```

max_active表示每个处理器最大可同时执行的work数量。

workqueue整体设计涉及以下几个概念：

- work：处理函数
- workqueue：持有多个pool_workqueue，通过`pwq = rcu_dereference(*per_cpu_ptr(wq->cpu_pwq, cpu))`可以取出当前处理器的workqueue
- worker：每个工人对应一个内核线程
- worker_pool：持有多个workers
- pool_workqueue：类似一个中介，持有1个worker_pool和1个workqueue（谁持有了我）

工人线程处理工作：kernel/workqueue.c

```c
static int worker_thread(void *__worker)
{
	struct worker *worker = __worker;
	struct worker_pool *pool = worker->pool;
    
	do {
		struct work_struct *work =
			list_first_entry(&pool->worklist,
					 struct work_struct, entry);

		if (assign_work(work, worker, NULL))
			process_scheduled_works(worker);
	} while (keep_working(pool));
    ...
}
```

process_scheduled_works -> process_one_work

```c
static void process_one_work(struct worker *worker, struct work_struct *work)
{
	worker->current_work = work;
	worker->current_func = work->func;
	worker->current_pwq = pwq;
    ...
    worker->current_func(work);  // 执行处理函数
    ...
}
```



## 参考

1. [Linux 中断（IRQ/softirq）基础：原理及内核实现（2022）](https://arthurchiao.art/blog/linux-irq-softirq-zh/)
2. 《Linux内核深度解析》