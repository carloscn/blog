# 0x23_LinuxKernel_内核活动（三）中断体系结构（中断下文）

内核为了不影响整个系统的性能，在设计上更倾向于**让内核避免禁用中断**，因为如果禁用中断，那么内核禁用中断执行中断上文的时间片内不会接收来自外设的请求（`IRQF_DISABLED`），例如这个时候输入键盘或者鼠标就会卡顿[^1]，会miss掉网络数据等等。但有些场合**内核禁用中断是必要的**，例如，在SMP场景下，其他CPU没有中断处理函数和延迟函数（软中断），当获取自旋锁之前，内核必须禁用本地中断，否则中断处理程序可能被冻结，除非被该中断程序释放自旋锁[^2]。

如果内核允许在禁用中断的情况下，花费过多的时间处理一个ISR，那么可能就会miss掉一些对系统正确性必不可少的中断。因此，内核为了解决该问题，将中断分为上文和中断下文。中断上文参考[# 0x22_LinuxKernel_内核活动（二）中断体系结构（中断上文）](https://github.com/carloscn/blog/issues/68)，本文着眼于中断下文部分，即内核为了节约ISR处理时间把一部分不重要的操作移动到中断下文中处理，而中断下文释放了中断资源是在内核进程中执行，这就极大的提高了内核响应中断的效率[^3]。

中断的下文可以分为：
* 软中断（soft irq）
* 软中断的延伸 (tasklet)
* 等待队列(wait queue)和完成量(completion)
* 工作队列(work queue)

# 1. 中断下文故事[^4]

最早的Linux只提供了“bottom half”这种机制来实现下半部。它提供了一种静态创建的且由32个bottom halves阻证的链表。中断上文通过一个32位整数中的一个位来表示出哪个bottom half可以执行。每个BH在全局范围内同事同步。即便是分属于不同的处理器，也不允许任何两个BH同事执行。

> 在书籍《Professional Linux Kernel Architecture》page 879 一页中对于术语 下半部`bottom half` 有一个定义上的区分：
> **第一种，就是上文提到的BH机制**，特指这个BH机制，并不是一类操作的统称。这部分已经被替换掉了，不会出现在内核里面了。因此，术语`bottom half`不再使用代表这个机制。而且**BH大部分操作已经并入到tasklet中**。
> **第二种，就是我们传统意义的ISR代码的下半部**，包含tasklet/workqueue等等等。

这样的效率无疑是低下的，kernel开发者引入了**任务队列**来实现下文工作的延迟执行，并且取代了BH机制。内核定义了一组队列，其中每个队列都包含一个由等待调用的函数组成的链表。根据所处的队列的位置，这些函数会在某个时刻执行。因此，驱动程序可以把他们自己的下半部注册到合适的队列上去。工作队列更**类似于公交车**的运行原理，我们在到达公交车站等待公交车的到来上车，公交车会根据路况某个时刻才能到站。

在内核2.3版本中，引入了软中断和tasklet机制。软中断是一组静态定义的中断下文接口，有32个。可以在所有的处理器上同时执行，即便是两个类型相同的也是可以的。两个不同类型的tasklet可以在不同的处理器上同时执行。tasklet实际上是一种性能和易用性之间寻求平衡的产物。**对于大部分中断下文来说，用tasklet就足够了，像网络这样对于性能要求非常高的情况下才需要使用软中断**。可是，使用软中断需要注意的是，因为两个相同的软中断可能被同时执行，存在竞争的可能性。而且，软中断在编译阶段就进行了静态注册，tasklet可以通过代码动态注册。

另外一个可以延迟操作的手段是**内核定时器**。内核定时器把操作推迟到某个确定的时间段执行。也就是是说，当你必须保证在一个确定的时间段过去以后在执行，可以选择内核定时器。

工作队列是另外一种延迟执行的形式，工作队列可以把工作推后，交给一个内核线程去执行，因此这个中断下文总是会在进程上下文中执行。这样，通过工作队列执行的代码能占尽进程上下文的所有优势。最重要的就是工作队列允许重新调度和睡眠。

# 2. 软中断

“软中断”可以使得内核延期执行任务。因为它们的运作方式和“硬中断”类似，但完全是用软件逻辑实现，所以成为软中断（software interrupt）或者softIRQ。同时这里还要区分一个在ARM处理器中的一个“软中断”的术语，在ARMv7架构中同样也定义了“软中断”的指令`SWI`，**这个和Kernel里面的软中断不是一个概念**，ARM架构中的软中断是一个特权模式切换的系统调用指令[^5]。不过在ARM很多文档中已经加备注将`SWI`更名为`SVC`，可能考虑到和kernel的定义冲突吧。

> #### Note
> 
> SVC was previously called SWI, Software Interrupt, and this name is still found in some documentation.

软中断执行的时间片是内核在`do_IRQ`的末尾处理所有待决的软中断，因而可以确保软中断可以定期得到处理。从流程上来看，软中断可以描述为一种延迟稍后处理的内核活动。

## 2.1 softirq数据结构

软中断机制的核心部分是一个表，包含32个`softirq_action`类型的数据项。该数据类型结构非常简单，只包含两个成员： 
https://elixir.bootlin.com/linux/v2.6.24/source/include/linux/interrupt.h#L265
```C
struct softirq_action
{
	void	(*action)(struct softirq_action *);
	void	*data;
};
```

（1）`action`是一个指向处理程序的指针，在软中断发生时候由内核执行该回调函数；
（2）`data`是一个指向处理程序函数的私有数据的指针。

软中断必须先注册，然后内核才可以执行。`open_softirq`函数即用于该目的。

```C
void open_softirq(int nr, void (*action)(struct softirq_action*), void *data)
{
	softirq_vec[nr].data = data;
	softirq_vec[nr].action = action;
}
```

请注意`int nr`这个参数，各个软中断都有一个唯一的编号，软中断的编号是稀缺的，**不能由各个设备驱动程序和内核组件随意使用**。默认情况下，系统上只有**32个软中断**。但这个没有什么关系，因为中断下文还有tasklet和工作队列机制等可以给驱动程序使用。

> /* PLEASE, avoid to allocate new softirqs, if you need not _really_ high
   frequency threaded job scheduling. For almost all the purposes
   tasklets are more than enough. F.e. all serial device BHs et
   al. should be converted to tasklets, not to softirqs.
 */

在内核代码里面也可以看到相关的提示，请避免去使用softirq，除非你真的需要高频的操作的任务。对于一般性的操作，tasklets已经够用了。以下类型是内核默认注册的softirq的机制：

https://elixir.bootlin.com/linux/v2.6.24/source/include/linux/interrupt.h#L247

```C
enum
{
	HI_SOFTIRQ=0,    /* 实现tasklet机制 */
	TIMER_SOFTIRQ,   /* 定时器 */
	NET_TX_SOFTIRQ,  /* 网络发送 */
	NET_RX_SOFTIRQ,  /* 网络接受 */
	BLOCK_SOFTIRQ,   /* 基于块层实现异步请求 */
	TASKLET_SOFTIRQ, /* 实现tasklet机制 */
	SCHED_SOFTIRQ,   /* 用于调度器，SMP系统的负载均衡 */
#ifdef CONFIG_HIGH_RES_TIMERS
	HRTIMER_SOFTIRQ, /* 高分辨率定时器 */
#endif
};
```

`raise_softirq(int nr)`用于引发一个软中断（类似于普通中断）。该函数通过设置CPU变量`irq_stat[smp_processor_id].__softirq_pending`中的对应比特位，可以同时在不同的CPU上执行。

使用`cat /proc/softirq`可以看到“列表”和“每个CPU发生的中断次数”：

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220729141237.png" width="60%" /> </div>

**启动软中断的可以通过两种途径**：
* 在中断上下文调用 `raise_softirq`
* 非中断上下文，则间接的调用`wakeup_softirqd`唤醒软中断的守护进程，间接使用守护进程启动软中断。

以armv8为例，展开softirq的处理流程如下图所示[^8]：

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220729141435.png" width="90%" /> </div> 

## 2.2 开启软中断处理

开启软中断处理的的流程函数在`do_softirq`。如果平台依赖于kernel内部的实现需要定义:[__ARCH_HAS_DO_SOFTIRQ](https://elixir.bootlin.com/linux/v2.6.24/C/ident/__ARCH_HAS_DO_SOFTIRQ) （少部分架构会有自己的实现，ARM是直接用kernel的实现的）

https://elixir.bootlin.com/linux/v2.6.24/source/kernel/softirq.c#L256

```C
asmlinkage void do_softirq(void)
{
	__u32 pending;
	unsigned long flags;

	if (in_interrupt())   // ————————————（1）
		return;

	local_irq_save(flags);   

	pending = local_softirq_pending();   // ————————————（2）

	if (pending)
		__do_softirq(); 

	local_irq_restore(flags);
}
```

<div align='left'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220729102756.png" width="90%" /> </div> 

（1）  该函数首先确认是否在中断上下文中使用系统调用`in_interrupt`函数。如果处于中断上下文中立刻结束。
（2） 通过`local_softirq_pending`，确定当前cpu软中断位图所有置位的比特位。如果软中断处于等待处理，则调用`__do_softirq`。
（3） 在`__do_softirq`中会循环查询是否有中断处理，重启次数没有超过`MAX_SOFTIRQ_RESTART`（通常设定为10），则会一直重启循环；如果超过了这个值，仍有没有处理的中断，那么内核将调用`wakeup_softirqd`唤醒守护进程处理软中断。

## 2.3 软中断守护进程

软中断守护进程的任务是，与其余的内核代码异步执行软中断。**在SMP系统中，有几个Core那么就会有几个守护进程**。守护进程被称为`ksoftirqd`。*The SoftIRQ Daemon*

启动守护进程正如上文所说，`wakeup_softirqd`函数就可以唤醒守护进程。
https://elixir.bootlin.com/linux/v2.6.24/source/kernel/softirq.c#L57

```C
static inline void wakeup_softirqd(void)
{
	/* Interrupts are disabled: no need to stop preemption */
	struct task_struct *tsk = __get_cpu_var(ksoftirqd);

	if (tsk && tsk->state != TASK_RUNNING)
		wake_up_process(tsk);  // 优先级19
}
```

**调用守护进程的位置**：
* 在`do_softirq`中，如前面所说。
* 在`raise_softirq_irqoff`末尾。该函数在`raise_softirq`内部调用，如果内核当前停用了中断，也可以直接使用。

系统在启动`initcall`机制调用`init`不久，就创建了系统软中断的守护进程。在初始化之后，各个守护进程都执行以下无限循环：

https://elixir.bootlin.com/linux/v2.6.24/source/kernel/softirq.c#L486

```C
static int ksoftirqd(void * __bind_cpu)
{
	set_current_state(TASK_INTERRUPTIBLE);

	while (!kthread_should_stop()) {
		preempt_disable();
		if (!local_softirq_pending()) {
			preempt_enable_no_resched();
			schedule();
			preempt_disable();
		}

		__set_current_state(TASK_RUNNING);

		while (local_softirq_pending()) {
			/* Preempt disable stops cpu going offline.
			   If already offline, we'll be on wrong CPU:
			   don't process */
			if (cpu_is_offline((long)__bind_cpu))
				goto wait_to_die;
			do_softirq();
			preempt_enable_no_resched();
			cond_resched();
			preempt_disable();
		}
		preempt_enable();
		set_current_state(TASK_INTERRUPTIBLE);
	}
	__set_current_state(TASK_RUNNING);
	return 0;

wait_to_die:
	preempt_enable();
	/* Wait for kthread_stop */
	set_current_state(TASK_INTERRUPTIBLE);
	while (!kthread_should_stop()) {
		schedule();
		set_current_state(TASK_INTERRUPTIBLE);
	}
	__set_current_state(TASK_RUNNING);
	return 0;
}
```

每次被唤醒时，守护进程首先检查是否有标记出的待解决的中断， 否则明确地调用调度器，并将控制转交到其他进程。如果有标记出来的软中断，那么守护进程接下来将处理中断。进程在一个while循环中重复调用两个函数`do_softirq`和`cond_resched`，直到没有标记出的软中断为止。`cond_resched`确保对当前进程设定了`TIF_NEED_RESCHED`标志情况下调用调度器。

## 2.4 网卡使用softirq实例

这部分待完善，这部分涉及：
* Linux块驱动
* softirq机制
* ....
我们在完成Linux驱动设备部分之后再来给出实例，参考[^6]

# 3. tasklet
软中断是将操作推迟到未来某个时刻（do_ISR或异步守护进程），但是softirq机制处理起来十分复杂，因为多个处理器可以同时且独立地处理软中断，同一个软中断的处理可以在几个CPU上面同时运行。**这对于程序执行效率是一个有效的增益，例如网络在多处理器系统上的处理，但同时也引入了多个执行者模型中同步的问题，线程安全的可重入性必须考虑，其次必须考虑代码的一些临界区**。

从上文中知道`HI_SOFTIRQ`和`TASKLET_SOFTIRQ`的ID是给tasklets机制预留的，因此tasklet是基于softirq的更高一层次的封装（`work_queue`也是）。tasklets的设计更倾向于让用户在设备驱动中更容易使用。tasklet在内核中的地位是“小进程”，执行一些迷你任务。

整体结构如图所示[^7]

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora0x23_01.drawio.svg" width="80%" /> </div>


## 3.1 创建tasklet

tasklet的结构体表述为：
```C
struct tasklet_struct
{
	struct tasklet_struct *next; /*tasklet是一个链表结构*/
	unsigned long state; // （1）
	atomic_t count;      // （2）
	void (*func)(unsigned long); // （3）
	unsigned long data;
};
```

* （1） `state`表示任务的当前状态，类似于真正的进程。但是只有两个选项
	* `TASKLET_STATE_SCHED`等待调度执行
	* `TASKLET_STATE_RUN`表示tasklet正在执行。该状态只有在SMP模式下有用，用于保护tasklet在多个处理器上并行执行。
* （2） `count`为原子计数器用于禁用已经调度的tasklet。如果其值不等于0，被忽略。
* （3） `func`最重要的成员，真正的action实现，中断延迟任务就挂在上面。

## 3.2 注册tasklet

`tasklet_schedule`将一个tasklet注册到系统中：

<div align="center"> <b><i>static inline void tasklet_schedule(struct tasklet_struct *t); </i></b></div>

把自己的tasklet_schedule挂接到链表上。

## 3.3 执行tasklet

tasklet生命周期最重要的就是执行，因为tasklet基于软中断实现，他们总是在处理中断的时候执行。talsket关联到`TASKLET_SOFTIRQ`软中断。因而，调用`raise_softirq(TASKLET_SOFTIRQ)`，即可在下一个适当的时机执行当前的tasklet。内核使用`tasklet_action`作为该软中断的`action`函数。

https://elixir.bootlin.com/linux/v2.6.24/source/kernel/softirq.c#L385
```C
static void tasklet_action(struct softirq_action *a)
...
while (list) {
	struct tasklet_struct *t = list;
	list = list->next;
	if (tasklet_trylock(t)) {
		if (!atomic_read(&t->count)) {
			if (!test_and_clear_bit(TASKLET_STATE_SCHED, &t->state))
				BUG();
				t->func(t->data);
				tasklet_unlock(t);
				continue;
			}
			tasklet_unlock(t);
		}
		...
	}
	...
}
```

在while循环中执行tasklet，类似于处理软中断使用机制。因为一个tasklet只能在一个处理器上执行一次，但是其他tasklet可以并行运行，所欲需要特定的tasklet锁机制。`state`状态变量作为锁变量。`tasklet_trylock`检查tasklet的状态是否为`TASKLET_STATE_RUN`。（检查是否在其他core上面运行了）

<interrupt.h>
```C
static inline int tasklet_trylock(struct tasklet_struct *t)
{
	return !test_and_set_bit(TASKLET_STATE_RUN, &(t)->state);
}
```

除了普通的tasklet之外，内核还使用了taskelt，它具有“较高”的优先级。
* 使用`HI_SOFTIRQ`作为软中断，相关的action函数变为`tasklet_hi_action`
* 同时，注册tasklet的CPU相关变量在`tasklet_hi_vec`中排队。

**大部分声卡程序都利用了HI_SOFTIRQ**，因为操作延迟时间太长可能损害音频输出的音质。

> Note， Linux在软中断上下文中是不能睡眠的，原因在于Linux的软中断实现上下文有可能是中断上下文，如果在中断上下文中睡眠，那么会导致Linux无法调度，直接的反应是系统Kernel Panic，并且提示dequeue_task出错。所以，在软中断上下文中，**我们不能使用信号量等可能导致睡眠的函数**，这一点在编写IO回调函数时需要特别注意。在最近的一个项目中，我们在dm-io的callback函数中去持有semaphore访问竞争资源，导致了系统的kernel panic。其原因就在于dm-io的回调函数在scsi soft irq中执行，scsi soft irq是一个软中断，其会在硬中断发生之后被执行，执行上下文为中断上下文。

# 4. 队列
在Linux中最常见的三个队列[^9]：
* **等待队列** + **完成量**
* **工作队列**
* ~请求队列~ (**不在本文scope， 参考kernel网络部分的相关文章**)

在内核里，**等待队列**是有很多用处的，尤其是在中断、进程同步、定时的场合。可以使用等待队列在实现阻塞进程的唤醒。**工作队列**将一个work提交到workqueue上，而这个workqueue是挂到一个特殊内核进程上，当这个特殊内核进程被调度时，会从workqueue上取出work来执行。

## 4.1 等待队列与完成变量

**等待队列**(wait queue)用于使进程等待某一特定事件发生，而无需频繁轮询。进程在等待期间睡眠，在事件发生时候由内核自动唤醒。与等待队列并行的概念**完成量**(completion)机制基于等待队列，内核利用该机制等待某一操作结束。**这两种机制的使用都比较繁琐**，主要用于设备驱动程序。

以进程阻塞和唤醒的过程为例，等待队列的使用场景可以简述为：进程A因等待某些资源（依赖进程B的某些操作）而不得不进入阻塞状态，便将当前进程加入到等待队列Q中。进程B在一系列操作后，可通知进程A所需资源已到位，便调用唤醒函数`wake up`来唤醒等待队列上Q的进程，注意此时所有等待在队列Q上的进程均被置为可运行状态[^12]。

### 4.1.1 等待队列

#### 数据结构

等待队列的数据结构表示如图所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220729160812.png)

每个等待队列都有一个队列头，用如下的结构体表示：
[struct __wait_queue_head](http://lxr.linux.no/#linux+v3.1.1/include/linux/wait.h#L50)

```C
typedef struct __wait_queue_head wait_queue_head_t;

struct __wait_queue_head {
        spinlock_t lock;
        struct list_head task_list;
};  
```

（1） 因为等待队列可以在中断时修改，在操作队列之前必须持有lock；
（2）`task_list`是一个双链表结构（队列）。

队列成员是以下数据结构的实例：
https://elixir.bootlin.com/linux/v2.6.24/source/include/linux/wait.h#L32
```C
typedef struct __wait_queue wait_queue_t;

/* wait_queue_entry::flags */
#define WQ_FLAG_EXCLUSIVE   0x01
#define WQ_FLAG_WOKEN       0x02
#define WQ_FLAG_BOOKMARK    0x04

struct __wait_queue {
	unsigned int flags;
#define WQ_FLAG_EXCLUSIVE	0x01
	void *private;
	wait_queue_func_t func;
	struct list_head task_list;
};
```

`flags` [^12]

*  `WQ_FLAG_EXCLUSIVE`  
	* 当某进程调用`wake up`函数唤醒等待队列时，队列上所有的进程均被唤醒，在某些场合会出现唤醒的所有进程中，只有某个进程获得了期望的资源，而其他进程由于资源被占用不得不再次进入休眠。如果等待队列中进程数量庞大时，该行为将影响系统性能。  
	* 内核增加了“独占等待” (`WQ_FLAG_EXCLUSIVE`)来解决此类问题。一个独占等待的行为和通常的休眠类似，但有如下两个重要的不同：
		* 等待队列元素设置`WQ_FLAG_EXCLUSIVE`标志时，会被添加到等待队列的尾部，而非头部。
		* 在某等待队列上调用`wake up`时，执行独占等待的进程每次只会唤醒其中第一个（所有非独占等待进程仍会被同时唤醒）。

* `WQ_FLAG_WOKEN`  暂时还未理解，TODO
* `WQ_FLAG_BOOKMARK`  用于`wake up`唤醒等待队列时实现分段遍历，减少单次对自旋锁的占用时间。

（1） 为了是当前进程在一个等待队列中睡眠，需要调用`wait_event`函数（或某个等价函数），让进程进入到睡眠状态，该进程的控制权交到了调度器手里。内核通常会在向块设备发出传输数据的请求后，调用该函数。因为传输不会立即发生，而在此期间无事可做，所以进程可以睡眠，将CPU时间让给系统中的其他进程。

（2） 在内核的另一处，就我们的例子而言，是来自块设备的数据到达后，必须要调用`wake_up`函数来唤醒等待队列中的睡眠进程。

#### 队列初始化[^12]

等待队列头的定义和初始化有两种方式：`init_waitqueue_head(&wq_head)`和宏定义`DECLARE_WAIT_QUEUE_HEAD(name)`。

```C
#define init_waitqueue_head(wq_head)                            \
    do {                                                        \
        static struct lock_class_key __key;                     \
        __init_waitqueue_head((wq_head), #wq_head, &__key);     \
    } while (0)

void __init_waitqueue_head(struct wait_queue_head *wq_head, const char *name, struct lock_class_key *key)
{
    spin_lock_init(&wq_head->lock);
    lockdep_set_class_and_name(&wq_head->lock, key, name);
    INIT_LIST_HEAD(&wq_head->head);
}

#define DECLARE_WAIT_QUEUE_HEAD(name)                       \
    struct wait_queue_head name = __WAIT_QUEUE_HEAD_INITIALIZER(name)

#define __WAIT_QUEUE_HEAD_INITIALIZER(name) {               \
    .lock       = __SPIN_LOCK_UNLOCKED(name.lock),          \
    .head       = { &(name).head, &(name).head } }

```

#### 等待队列元素的创建和初始化[^12]

创建等待队列元素较为普遍的一种方式是调用宏定义`DECLARE_WAITQUEUE(name, task)`，将定义一个名为`name`的等待队列元素，`private`数据指向给定的关联进程结构体`task`，唤醒函数为`default_wake_function()`。后文介绍唤醒细节时详细介绍唤醒函数的工作。

```c
#define DECLARE_WAITQUEUE(name, tsk)                        \
    struct wait_queue_entry name = __WAITQUEUE_INITIALIZER(name, tsk)

#define __WAITQUEUE_INITIALIZER(name, tsk) {                \
    .private    = tsk,                                      \
    .func       = default_wake_function,                    \
    .entry      = { NULL, NULL } }
```

内核源码中还存在其他定义等待队列元素的方式，调用宏定义`DEFINE_WAIT(name)`和`init_wait(&wait_queue)`。
这两种方式都将**当前进程(current)**关联到所定义的等待队列上，唤醒函数为`autoremove_wake_function()`，注意此函数与上述宏定义方式时不同（上述定义中使用default_wake_function）。
下文也将介绍`DEFINE_WAIT()`和`DECLARE_WAITQUEUE()`在使用场合上的不同。

```c
#define DEFINE_WAIT(name)   DEFINE_WAIT_FUNC(name, autoremove_wake_function)

#define DEFINE_WAIT_FUNC(name, function)                    \
    struct wait_queue_entry name = {                        \
        .private    = current,                              \
        .func       = function,                             \
        .entry      = LIST_HEAD_INIT((name).entry),         \
    }
#define init_wait(wait)                                     \
    do {                                                    \
        (wait)->private = current;                          \
        (wait)->func = autoremove_wake_function;            \
        INIT_LIST_HEAD(&(wait)->entry);                     \
        (wait)->flags = 0;                                  \
    } while (0)
```

#### 添加和移除等待队列[^12]

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220729175932.png) [^12]

内核提供了两个函数（定义在`kernel/sched/wait.c`）用于将等待队列元素`wq_entry`添加到等待队列`wq_head`中：`add_wait_queue()`和`add_wait_queue_exclusive()`。

-   `add_wait_queue()`：在等待队列头部添加普通的等待队列元素（非独占等待，清除`WQ_FLAG_EXCLUSIVE`标志）。
-   `add_wait_queue_exclusive()`：在等待队列尾部添加独占等待队列元素（设置了`WQ_FLAG_EXCLUSIVE`标志）。

```C
void add_wait_queue(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry)
{
    unsigned long flags;

    // 清除WQ_FLAG_EXCLUSIVE标志
    wq_entry->flags &= ~WQ_FLAG_EXCLUSIVE;
    spin_lock_irqsave(&wq_head->lock, flags);
    __add_wait_queue(wq_head, wq_entry);
    spin_unlock_irqrestore(&wq_head->lock, flags);
}   

static inline void __add_wait_queue(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry)
{
    list_add(&wq_entry->entry, &wq_head->head);
}
```

`remove_wait_queue()`函数用于将等待队列元素`wq_entry`从等待队列`wq_head`中移除。

```C
void add_wait_queue_exclusive(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry)
{
    unsigned long flags;

    // 设置WQ_FLAG_EXCLUSIVE标志
    wq_entry->flags |= WQ_FLAG_EXCLUSIVE;
    spin_lock_irqsave(&wq_head->lock, flags);
    __add_wait_queue_entry_tail(wq_head, wq_entry);
    spin_unlock_irqrestore(&wq_head->lock, flags);
}

static inline void __add_wait_queue_entry_tail(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry)
{
    list_add_tail(&wq_entry->entry, &wq_head->head);
}
```

#### 等待事件[^12]
内核中提供了等待事件`wait_event()`宏（以及它的几个变种），可用于实现简单的进程休眠，等待直至某个条件成立，主要包括如下几个定义：

```
wait_event(wq_head, condition)
wait_event_timeout(wq_head, condition, timeout) 
wait_event_interruptible(wq_head, condition)
wait_event_interruptible_timeout(wq_head, condition, timeout)
io_wait_event(wq_head, condition)
```

上述所有形式函数中，`wq_head`是等待队列头（采用”值传递“的方式传输函数），`condition`是任意一个布尔表达式。使用`wait_event`，进程将被置于非中断休眠，而使用`wait_event_interruptible`时，进程可以被信号中断。
另外两个版本`wait_event_timeout`和`wait_event_interruptible_timeout`会使进程只等待限定的时间（以**jiffy**表示，给定时间到期时，宏均会返回0，而无论`condition`为何值）。

https://elixir.bootlin.com/linux/v2.6.24/source/include/linux/wait.h#L164

```C
#define __wait_event(wq, condition) 					\
do {									\
	DEFINE_WAIT(__wait);						\
									\
	for (;;) {							\
		prepare_to_wait(&wq, &__wait, TASK_UNINTERRUPTIBLE);	\
		if (condition)						\
			break;						\
		schedule();						\
	}								\
	finish_wait(&wq, &__wait);					\
} while (0)
```

在`DEFINED_WAIT`建立等待队列成员之后，这个宏产生了一个无限循环。使用`prrepare_to_wait`使进程在等待队列上睡眠。每次进程被唤醒之后，内核都会检查指定的条件是否满足，如果条件满足则退出循环；如果条件不满足，将权限交给调度器，进程再次睡眠。

示例[^12]

```C
void
prepare_to_wait(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry, int state)
{
    unsigned long flags;

    wq_entry->flags &= ~WQ_FLAG_EXCLUSIVE;
    spin_lock_irqsave(&wq_head->lock, flags);
    if (list_empty(&wq_entry->entry))
        __add_wait_queue(wq_head, wq_entry);
    set_current_state(state);
    spin_unlock_irqrestore(&wq_head->lock, flags);
}
```

可以看到`prepare_to_wait()`实际做的事情也就是将等待队列元素加入到等待队列中，然后更新当前进程状态。可以看出此过程依旧符合之前介绍的等待队列一般使用流程，只是内核源码将部分流程封装成为此函数。  

`prepare_to_wait()`配合`finish_wait()`函数可实现等待队列。

#### 队列唤醒[^12]

前文已经简单提到，`wake_up`函数可用于将等待队列上的所有进程唤醒，和`wait_event`相对应，`wake_up`函数也包括多个变体。主要包括：

```C
#define wake_up(x) __wake_up(x, TASK_UNINTERRUPTIBLE | TASK_INTERRUPTIBLE, 1, NULL)
#define wake_up_nr(x, nr) __wake_up(x, TASK_UNINTERRUPTIBLE | TASK_INTERRUPTIBLE, nr, NULL)
#define wake_up_all(x) __wake_up(x, TASK_UNINTERRUPTIBLE | TASK_INTERRUPTIBLE, 0, NULL)
#define wake_up_interruptible(x) __wake_up(x, TASK_INTERRUPTIBLE, 1, NULL)
#define wake_up_interruptible_nr(x, nr) __wake_up(x, TASK_INTERRUPTIBLE, nr, NULL)
#define wake_up_interruptible_all(x) __wake_up(x, TASK_INTERRUPTIBLE, 0, NULL)
```

`wake_up`可以用来唤醒等待队列上的所有进程，而`wake_up_interruptible`只会唤醒那些执行可中断休眠的进程。因此约定，`wait_event`和`wake_up`搭配使用，而`wait_event_interruptible`和`wake_up_interruptible`搭配使用。  

前文提到，对于独占等待的进程，`wake_up`只会唤醒第一个独占等待进程。`wake_up_nr`函数提供功能，它能唤醒给定数目`nr`个独占等待进程，而不是只有一个。

从函数调用过程中可以看到，`default_wake_function()`实现唤醒进程的过程为：

```text
default_wake_function() 
		|--> try_to_wake_up() 
			|--> ttwu_queue() 
				|--> ttwu_do_activate() 
					|--> ttwu_do_wakeup()
```

值得一提的是，**`default_wake_function()`的实现中并未将等待队列元素从等待队列中删除**。因此，编写程序时不能忘记添加步骤将等待队列元素从等待队列元素中删除。

`autoremove_wake_function()`相比于`default_wake_function()`，在成功执行进程唤醒工作后，会自动将等待队列元素从等待队列中移除。

#### 源码实例[^12]

等待队列在内核中有着广泛的运用，此处以MMC驱动子系统中`mmc_claim_host()`和`mmc_release_host()`来说明等待队列的运用实例。  

`mmc_claim_host()`的功能为：借助等待队列申请获得MMC主控制器(host)的使用权，相对应，`mmc_release_host()`则是放弃host使用权，并唤醒所有等待队列上的进程。

```C
static inline void mmc_claim_host(struct mmc_host *host)
{
    __mmc_claim_host(host, NULL, NULL);
}

int __mmc_claim_host(struct mmc_host *host, struct mmc_ctx *ctx, atomic_t *abort)
{
    struct task_struct *task = ctx ? NULL : current;

    // 定义等待队列元素，关联当前进程，唤醒回调函数为default_wake_function()
    DECLARE_WAITQUEUE(wait, current);
    unsigned long flags;
    int stop;
    bool pm = false;

    might_sleep();

    // 将当前等待队列元素加入到等待队列host->wq中
    add_wait_queue(&host->wq, &wait);
    spin_lock_irqsave(&host->lock, flags);
    while (1) {
        // 当前进程状态设置为 TASK_UPINTERRUPTIBLE，此时仍未让出CPU
        set_current_state(TASK_UNINTERRUPTIBLE);
        stop = abort ? atomic_read(abort) : 0;
        // 真正让出CPU前判断等待的资源是否已经得到
        if (stop || !host->claimed || mmc_ctx_matches(host, ctx, task))
            break;
        spin_unlock_irqrestore(&host->lock, flags);
        // 调用调度器，让出CPU，当前进程可进入休眠
        schedule();
        spin_lock_irqsave(&host->lock, flags);
    }
    // 从休眠中恢复，设置当前进程状态为可运行(TASK_RUNNING)
    set_current_state(TASK_RUNNING);
    if (!stop) {
        host->claimed = 1;
        mmc_ctx_set_claimer(host, ctx, task);
        host->claim_cnt += 1;
        if (host->claim_cnt == 1)
            pm = true;
    } else
        // 可利用abort参数执行一次等待队列唤醒工作
        wake_up(&host->wq);
    spin_unlock_irqrestore(&host->lock, flags);

    // 等待队列结束，将等待队列元素从等待队列中移除
    remove_wait_queue(&host->wq, &wait);

    if (pm)
        pm_runtime_get_sync(mmc_dev(host));

    return stop;
}

void mmc_release_host(struct mmc_host *host)
{
    unsigned long flags;

    WARN_ON(!host->claimed);

    spin_lock_irqsave(&host->lock, flags);
    if (--host->claim_cnt) {
        /* Release for nested claim */
        spin_unlock_irqrestore(&host->lock, flags);
    } else {
        host->claimed = 0;
        host->claimer->task = NULL;
        host->claimer = NULL;
        spin_unlock_irqrestore(&host->lock, flags);

        // 唤醒等待队列host->wq上的所有进程
        wake_up(&host->wq);
        pm_runtime_mark_last_busy(mmc_dev(host));
        if (host->caps & MMC_CAP_SYNC_RUNTIME_PM)
            pm_runtime_put_sync_suspend(mmc_dev(host));
        else
            pm_runtime_put_autosuspend(mmc_dev(host));
    }
}
```

从源码实现过程可以看到，此实例中等待队列的使用总结得基本过程一致，使用到的函数依次为：
-   `DECLARE_WAITQUEUE(wait, current)`
-   `add_wait_queue(&host->wq, &wait)`
-   `set_current_state(TASK_UNINTERRUPTIBLE)`
-   `schedule()`
-   `set_current_state(TASK_RUNNING)`
-   `remove_wait_queue(&host->wq, &wait)`

#### 队列总结[^12]

综上文分析，等待队列的使用主要有三种方式： 

(1) **等待事件方式**  
`wait_event()`和`wake_up()`函数配合，实现进程阻塞睡眠和唤醒。  

(2) **手动休眠方式1**
```C
DECLARE_WAIT_QUEUE_HEAD(queue);
DECLARE_WAITQUEUE(wait, current);

for (;;) {
    add_wait_queue(&queue, &wait);
    set_current_state(TASK_INTERRUPTIBLE);
    if (condition)
        break;
    schedule();
    remove_wait_queue(&queue, &wait);
    if (signal_pending(current))
        return -ERESTARTSYS;
}
set_current_state(TASK_RUNNING);
remove_wait_queue(&queue, &wait);
```

(3) **手动休眠方式2（借助内核封装函数）**
```C
DELARE_WAIT_QUEUE_HEAD(queue);
DEFINE_WAIT(wait);

while (! condition) {
    prepare_to_wait(&queue, &wait, TASK_INTERRUPTIBLE);
    if (! condition)
        schedule();
    finish_wait(&queue, &wait)
}
```

## 4.2 工作队列
工作队列是将操作延期执行的另一种手段。他们**通过守护进程在用户上下文执行，函数可以睡眠任意长时间**。对于每个工作队列来说，内核都会创建一个守护进程，延期任务使用上下文描述的等待队列机制，在进程上下文中执行（工作队列是在进程中创建的一个线程，属于进程内的概念）。

### 4.2.1 queue概念

新的工作队列通过`create_workqueue`或`create_workqueueu_singlethread`函数来创建。`create_workqueue`在所有的CPU上面传建一个工作线程，而`create_workqueue_singlethread`仅仅在第一个CPU上面创建一个线程。而两个函数都使用了`__create_workqueue_key`：


```C
// kernel/workqueue.c
struct workqueue_struct *__create_workqueue(
		const char *name,
		int singlethread)
// name: 表示创建守护进程在进程列表中显示的名称；
// int singlethread： 0：则在每个CPU上面都创建一个线程，否则只在第一个CPU上创建。
```

所有推送到工作队列上的任务，都必须打包为`work_struct`结构体实例，从工作队列的用户的角度来看，该结构体的下述成员是比较重要的：

```C
// <workqueue.h>
struct work_struct;
typedef void (*work_func_t)(struct work_struct *work);

struct work_struct {
	atomic_long_t data; // 原子类型确保不会有并发问题
	struct list_head entry;
	work_func_t func;
}

/*
 * Note，在曾经的版本中没有使用atomic_long_t这个结构体来定义data，只使用void*
 * 这是一个比较重要的内核技巧：
 * `atomic_long_t`作为指向任意数据的指针的数据类型，而不是通常的void*，
 * 这是因为想要放更多的信息就进入到结构体。指针类型在所有体系结构上面都是对齐到4字节的边界；
 * 而前面的两个bit保证为0，因此可以“滥用”这两个bit位，将其用作标志位。在使用的时候使用宏来
 * 屏蔽标志位：
 * 
 * * #define WORK_STRUCT_FLAG_MASK 3UL
 * * #define WORK_STRUCT_WQ_DATA_MASK ~(WORK_STRUCT_FLAG_MASK)
 */
```

`entry`:  照例用于链表元素，用于将几个`work_struct`的实例嵌入到一个链表中。
`func`：回调延期函数真正的执行。

一个驱动程序后者内核模块要使用工作队列，创建一个work_struct结构，填充其中的func字段即可，之后调用schedule_work提交给对象即可。关于schedule_work后面我们在描述，下面开始展开内核对于工作队列的管理[^13]。

内核中既然把工作队列作为一种资源使用，其自然有其自身的管理规则，因此在内核中涉及到一下对象[^13]：
- worker：工作者，顾名思义为处理工作的单位；
- worker_pool：工作者池，每个worker必然属于某个worker_pool,一个worker_pool可以有多个worker；
- workqueue_struct：官方解释是对外部可见的workqueue；
- pool_workqueue：链接workqueue_struct  和worker_pool的中介，每个workqueue_struct 可以有多个worker_pool，而一个worker_pool只能属于一个workqueue_struct。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220805100027.png)

当前之定义了一个标志位：`WORK_STRUCT_PENDING`用来查找当前是否有待解决（该标志位1）的可延迟工作选项。辅助宏`work_pending(work)`用来检查该标志位。使用`INIT_WORK(work, func)`宏，他向一个现存的`work_struct`实例提供一个延期执行函数。

有两种方法可以向一个工作队列添加`work_struct`实例，分别是`queue_work`和`queue_work_delayed`。第一个函数的原型如下：

```C
//kernel/workqueue.c
int fastcall queue_work(struct workqueue_struct *wq, struct work_struct *work)
```

它将work添加到工作队列wq，work本身所指定的工作，其执行时间待定。为了确保排队工作在提交作业之后一定时间工作，需要扩展`work_struct`，添加一个定时器：

```C
// <workqueue.h>
struct delayed_work {
	struct work_struct work;
	struct timer_list timer;
};
```

`queue_delayed_work`用于向工作队列提交`delayed_work`实例，它确保在延期工作执行之前，至少delay一段时间(jiffies为单位)。

```C
//kernel/workqueue.c
int fastcall queue_delayed_work(struct workqueue_struct *wq,
				struct delayed_work *dwork, 
				unsigned long delay)
```

内核创建一个标准的队列，成为`events`。内核的各个部件中，凡是没有必要创建独立的工作队列者，均可以使用该队列。内核提供了一下两个函数，可以用于将新的工作添加到标准队列里面：

```C
//kernel/workqueue.c
int schedule_work(struct work_struct *work)
int schedule_delayed_work(struct delay_work *dwork, unsigned long delay)
```

### 4.2.2 queue实例[^14]

从一个驱动模块的角度分析工作队列的应用场景，包括队列的创建，工作初始化，工作入队列等。**内核源码版本v3.19.8**。

%%![](https://raw.githubusercontent.com/carloscn/images/main/typora202208%%05101942.png)

上图显示了工作队列的基本结构，我们以USB Hub驱动为例，分析驱动如何使用工作队列。USB控制器通过外部中断与CPU进行异步通讯，**由于中断对时间的要求特别敏感，所以USB驱动把主要的工作放在工作队列中处理**，这样一来可以有效的提高系统的响应能力。

#### A.创建队列

第一步，看看hub模块如何创建队列：
https://elixir.bootlin.com/linux/v3.19.8/source/drivers/usb/core/hub.c#L5155

```C
int usb_hub_init(void)
{
	if (usb_register(&hub_driver) < 0) {
		printk(KERN_ERR "%s: can't register hub driver\n",
			usbcore_name);
		return -1;
	}

	/*
	 * The workqueue needs to be freezable to avoid interfering with
	 * USB-PERSIST port handover. Otherwise it might see that a full-speed
	 * device was gone before the EHCI controller had handed its port
	 * over to the companion full-speed controller.
	 */
	hub_wq = alloc_workqueue("usb_hub_wq", WQ_FREEZABLE, 0);
	if (hub_wq)
		return 0;

	/* Fall through if kernel_thread failed */
	usb_deregister(&hub_driver);
	pr_err("%s: can't allocate workqueue for usb hub\n", usbcore_name);

	return -1;
}
```

USB hub模块在初始化的时候先注册hub驱动，接着创建工作队列。`alloc_workqueue`是创建队列的宏，负责创建一个新的队列。如上述`__create_workqueue_key`。5272行，创建全局唯一`hub_wq`队列，参数`WQ_FREEZABLE`表示工作线程在挂起时候，需要先完成当前队列的所以工作任务之后才能挂起。创建好队列后，需要定义一个任务用于完成实际的工作。

#### B. 初始化任务

工作队列创建成功后工作任务就有了栖身之所，以后只要往队列里添加任务就可以异步执行了。像USB hub这种公共型的模块，工作任务都是反复执行的，所以初始化一个`struct work_struct`实例即可。

https://elixir.bootlin.com/linux/v3.19.8/source/drivers/usb/core/hub.c#L1694

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220805102522.png)

* `hub＿probe`函数在发现hub设备的时候由内核调用，用于初始化hub设备对象；
* 1766行，分配hub设备对象空间，然后进行一系列初始化。USB驱动的逻辑比较复杂，我们另文分析；
* 1777行，初始化工作任务，`work_struct`实例最重要的成员就是func函数指针，这里把此指针初始化为`hub_event()`函数；
* 通过以上两个步骤，工作队列就可以使用了；

接下来，我们看看hub是如何触发工作任务的。

#### C. 激活任务

CPU收到USB中断事件后，在中断的上半部通过`hub＿irq()`处理，这个过程对时间要求比较敏感，所以处理完关键的工作后就把剩余的工作交给工作队列来处理。

https://elixir.bootlin.com/linux/v3.19.8/source/drivers/usb/core/hub.c#L634

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220805103236.png)

687行，踢工作队列一脚，触发工作队列的任务。从上一行注释中可以看到，内核收到了一些USB事件，但现在还不知道是什么内容，**由于时间紧迫，把它交给工作队列异步处理**。`kick＿hub＿wq()`函数实现如下：

https://elixir.bootlin.com/linux/v3.19.8/source/drivers/usb/core/hub.c#L574

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220805103300.png)

607行，把`work＿struct`对象入队列，此函数一经成功调用，同一个对象将不能重复入队列，只有等工作对象执行完毕后（即从`hub＿event`函数中返回后），才能再次入队列。由于USB的事件会不断地发生，这样可以有效的避免函数重入的问题，也就是说不会有多个处理函数同时在不同的CPU上调度执行。


[^1]:[Linux内核中断系统笔记 - 墨海 - 博客园](https://www.cnblogs.com/yanyansha/archive/2011/02/27/1966338.html)
[^2]:[Can the kernel disable interrupts?](https://quick-advisors.com/can-the-kernel-disable-interrupts/)
[^3]:[细说内核中断机制_jwy2014的博客-CSDN博客_内核中断](https://blog.csdn.net/jwy2014/article/details/80159248)
[^4]:[中断处理程序下半部(软中断、tasklet、工作队列) - 百度文库](https://wenku.baidu.com/view/2aa640defbc75fbfc77da26925c52cc58bd690bb.html)
[^5]:[# Arm A-profile A32/T32 Instruction Set Architecture - SVC](https://developer.arm.com/documentation/ddi0597/2022-06/Base-Instructions/SVC--Supervisor-Call-?lang=en)
[^6]:[kernel网络之软中断_分享放大价值的博客-CSDN博客](https://blog.csdn.net/fengcai_ke/article/details/125858898)
[^7]:[Linux内核中的下半部机制之tasklet - 知乎](https://zhuanlan.zhihu.com/p/88746106)
[^8]:[# 中断延迟处理机制「interrupt delay processing」](https://chasinglulu.github.io/2019/07/16/%E4%B8%AD%E6%96%AD%E5%BB%B6%E8%BF%9F%E5%A4%84%E7%90%86%E6%9C%BA%E5%88%B6%E3%80%8Cinterrupt-delay-processing%E3%80%8D/)
[^9]:[# linux系统队列,Linux系统中的”队列”是什么？Linux “队列”相关总结](https://blog.csdn.net/weixin_36317943/article/details/116585447)
[^10]:[http://kernel.meizu.com/ - # Linux Workqueue](http://kernel.meizu.com/linux-workqueue.html)
[^11]:[Analyzing a wait queue.](http://www.cs.albany.edu/~sdc/CSI500/Fal11/Labs/Modules/WaitingRight.html)
[^12]:[Linux等待队列（Wait Queue）](https://hughesxu.github.io/posts/Linux_Wait_Queue/)
[^13]:[# Linux中的工作队列 - 尚先生的博客 - csdn博客](https://blog.csdn.net/weixin_39094034/article/details/110953176)
[^14]:[# Linux工作队列workqueue源码分析](http://mp.ofweek.com/soft/a756714222177)
[^15]:[]()
[^16]:[]()
[^17]:[]()
[^18]:[]()
[^19]:[]()
[^20]:[]()