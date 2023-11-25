# 0x33_LinuxKernel_同步管理_原子操作_内存屏障_锁机制等
当一个资源被多个使用者竞争的时候，必然要去做一些保护措施。这个竞争者可能是中断、也可能是另一个core。Linux在用户空间给我们提供了很多同步手段，这些手段在system-v和poxis标准里面有所体现，system-v更面向于随内核持续的资源，而poxis更面向于随进程持续的资源。这些用户空间的同步手段，最终还是调用内核的low-level同步函数，而且内核中的资源保护，也指向了这些low-level的同步函数。本文主要就是来描述，内核的low-level的同步函数是如何实现的，他们的设计思想和理念是什么。对于之前的知识，请复习：
* 文献[[06_ARMv8_指令集_一些重要的指令](https://github.com/carloscn/blog/issues/12)] 包含原子操作的ARM机制-独占监视器(Exclusive monitor)，及使用它的指令`LDXR` `STXR`。
* 文献[[Linux-用户空间-多线程与同步](https://github.com/carloscn/blog/issues/9)] 中POXIS标准的userspace的同步机制的接口。
* 文献[[20_ARMv8_barrier（一）流水线和一致性模型](https://github.com/carloscn/blog/issues/62)] 提供内存屏障指令支撑。

# 1. 同步操作论述
首先引入一个**内核控制路径(kernel control path) 简称：内核路径**的概念[^1]，内核执行路径可以是中断处理程序、内核线程等。我们可以简单的引用文献[^2]中的内容来解释内核路径中比较重要的事情*参考附录X：内核抢占*。

早起不支持SMP的Linux内核中，导致并发访问的因素是**中断服务程序**，只有在中断发生的时候，或者内核代码路径**显式要求重新调度并且执行另一个进程时候**，才有可能发生并发访问。而在支持SMP的Linux内核中，并发运行在不同的CPU中的**内核线程**完全有可能同一时刻访问共享数据。尤其是现在内核支持内核抢占。

**内核中产生并发访问的并发源主要有四种**：
* **中断和异常**：中断发生的时候中断服务程序和主线程序可能会发生并发访问，并发访问发生在中断上下文；
* **软中断和tasklet**：软中断和tasklet可能随时被调度执行，从而打断当前正在执行的进程上下文，并发访问发生在进程上下文；
* **内核抢占**：调度器支持可抢占特性，这会导致进程之间并发访问，还是发生在进程上下文；
* **多处理器并发执行**：多处理器上可以同时并行多个进程，并非进程上下文，仅仅是进程和进程之间的。

**若在single-core系统中**：
* tasklet和softirq本质已经不是中断而是一种优先级比较高的特殊的进程，因此可以被中断打断，**内核延迟机制和中断**之间造成资源竞争；
* tasklet和softirq有着更高的优先级，可能随时会打断普通进程，因此**内核延迟机制和普通进程**之间会造成资源竞争；
* 支持抢占的内核**普通进程和普通进程**会造成资源竞争；不支持抢占的内核，进程上下文不会产生资源竞争；

**若在SMP系统中**：
* **在单核系统中的所有可能都存在**！
* 同一个类型的中断不会并发，但是不同类型的中断有可能被送到不同的CPU，执行中断的过程是多条路径的，如果访问同一个资源，存在资源竞争问题；
* 同一类型的软中断可能会被dispatch到不同的CPU上并发执行，软中断并发执行，如果访问同一个资源，存在资源竞争问题；
* **同一类型的tasklet是串行执行的，不会在多个CPU上并发执行**；**因此这里有个tasklet和软中断一个最大的区别**！！！！
* 不同CPU上进程上下文的并发执行；

**处理一下情况应该十分小心**：
* 在一个未使用内核同步机制保护资源的设计中；
* 进程上下文的访问和修改临界资源时发生了抢占调度；
* 在spinlock临界区中主动睡眠让出CPU；
* 两个CPU同事修改某个临界区资源。

锁和同步工具的使用并不难，而真正的难点在于**如何发现内核代码存在并发访问**。理论上，多个内核代码路径可能访问的资源都要去保护（需要我们设计者在设计之初就要trace执行路径，把控中断、进程切换点、使用的内核延迟机制）；还要注意，我们保护的是资源，而不是代码，这些资源是：全局变量、共享的数据结构、缓存、链表、红黑树等隐形的数据结构。要考虑：
* 除了内核是否还有其他人调用，多核，工作者，tasklet，软中断；
* 要假定内核可以被抢占，被调度；
* 进程睡眠，持资源睡下去；

Linux提供了丰富的工具，包含：原子操作、自旋锁、信号量、互斥锁、读写锁、RCU等，我们来看看具体如何实现的。

# 2. 原子操作与内存屏障

## 2.1 原子操作论述
先说一下一个问题，如下代码，两个线程同时对i进行++，（执行完之后）最后i的值是多少？

```C
static int i = 0;

void thread_a(void)
{
    i ++;
}
void thread_b(void)
{
    i++;
}
```

这个数可能是2，也可能是1，为什么呢？这是由于ARM的流水线乱序执行导致的，我们的C语言虽然只有一条指令，但是转为汇编伪代码，大概意思是：
```assembly
LDR X0, [&i]   // 把i从内存中load到寄存器中
ADD X0, X0, #1 // 把i自加1
STR X0, &i     // 把i的值从寄存器回写到内存中
```
如果流水线执行顺序是这样呢：
```assembly
thread_a: LDR X0, [&i]   // 把i从内存中load到寄存器中
thread_b: LDR X1, [&i]   // 把i从内存中load到寄存器中
thread_a: ADD X0, X0, #1 // 把i自加1
thread_a: STR X0, &i     // 把i的值从寄存器回写到内存中
thread_b: ADD X1, X1, #1 // 把i自加1
thread_b: STR X1, &i     // 把i的值从寄存器回写到内存中
```
此时乱序执行thread_b最后把A的计算结果覆盖，最后得到的不是1，而是2。问题就出在了，thread_b加载数据的值的时机有问题，thread_b应该等thread_a回写完数据之后再去load数据。

讲到这里，可能有人就考虑使用内存屏障了（如下代码），我们在操作前加上mb()强制要求处理器执行完毕之后才可以执行后面的指令。
```C
static int i = 0;

void thread_a(void)
{
    mb();
    i ++;
}
void thread_b(void)
{
    mb();
    i++;
}
```

还有一种方法是加锁，的确可以达到效果，但是对于一个变量而言，这个代价未免有点大。对于一个变量，实际上我们可以使用原子操作类型的函数来处理：

```C
static atomic_t i = 0;

void thread_a(void)
{
    ATOMIC_ADD(1, i);
}
void thread_b(void)
{
    ATOMIC_ADD(1, i);
}
```

这种就可以实现保证没有上面的操作了，因为ATOMIC操作，把上面三个指令“合并”成了一条："读-修改-回写"变成了一条组合指令。

我们来复习一下底层的ARM知识，原子操作在架构层级提供了什么帮助，引用[06_ARMv8_指令集_一些重要的指令](https://github.com/carloscn/blog/issues/12) 包含原子操作的ARM机制-独占监视器(Exclusive monitor)，及使用它的指令`LDXR` `STXR`。

 >在介绍内存独占加载和存储指令之前，先科普一下ARMv8架构里面的一个机制-独占监视器(Exclusive monitor)[4](https://github.com/carloscn/blog/issues/12#user-content-fn-4-e9c3558fb4438c1c81612759ce8e065c) ，虽然这个这个是ARMv6比较老的架构上面的文章，但是这个原理是不变的。ARM里面有两个独占监视器，一个本地的独占监视器，还有一个是全局的独占监视器。本地的独占监视器用于监视non-shareble/shareble的地址访问，全局的独占监视器用于监视shareble的地址访问（多核，如图Cortex-A8/Cortex-R4）。LDXR指令会让监视器进入到独占状态，STXR存储只有当独占监视器还处于独占状态的时候才可以存储成功。
 > <div align='center'><img src="https://user-images.githubusercontent.com/16836611/159872686-40605513-3135-433f-bd6b-f83999e0b519.png" width="50%" /></div>

## 2.2 Linux基本原子操作函数
Linux内核提供了最基本的原子操作的函数包括：
### 2.2.1 不带返回值的
* `atomic_read(atomic_t *v)` ：该函数对原子类型的变量进行原子读操作，它返回原子类型的变量v的值。
* `atomic_set(atomic_t *v,int i)` ：该函数设置原子类型的变量v的值为i。
* `atomic_add(int i, atomic_t *v)` ：该函数给原子类型变量v增加i。  
* `atomic_sub(int i, atomic_t *v)`：该函数给原子类型变量v减去i。  
* `atomic_sub_and_test(int i, atomic_t *v) `：该函数从原子类型的变量v中减去i，并判断结果是否是0，如果为0，返回真，否则返回假。  
* `atomic_inc(atomic_t *v)`：该函数对原子变量v原子的增加1。  
* `atomic_dec(atomic_t *v)`：该函数对原子变量v原子的减少1。  
* `atomic_dec_and_test(atomic_t *v)`：该函数对原子类型的变量v原子的减少1，并判断结果是否是0，如果是0,返回真，否则返回假。  
* `atomic_inc_and_test(atomic_t *v)`：该函数对原子类型的变量v原子增加1，并判断结果是否是0，如果是0，返回真，否则返回假。  
* `atomic_add_negative(int i, atomic_t *v)`：该函数对原子类型的变量v原子的增加I，并判断结果是否是负数，如果是，返回真，否则返回假。  
* `atomic_add_return(int i, atomic_t *v)`：该函数对原子类型的变量v原子的增加i，并且返回指向v的指针。  
* `atomic_sub_return(int i, atomic_t *v)`：该函数对原子累心的变量v中减去i，并且返回指向v的指针。
* 
### 2.2.2 带返回值的
除了这些之外，还有一些带返回值的原子操作函数格式为`atomic_fetch_{add,sub,or,xor}`返回原子变量的值。

### 2.2.3 原子交换函数
Linux内核提供了一些原子交换函数：
* `atomic_cmpxchg(ptr, old, new)`：原子比较ptr和值是否和old相等，若相等，把new的值放在ptr地址中，返回old。
* `atomic_xchg(ptr, new)`：原子地把new的值放在ptr地址中并返回ptr的原值。
* `atomic_try_cmpxchg(ptr, old, new)`：与atomic_cmpxchg函数类似，但是返回值发生变换，返回一个bool值，以判断是否相等。

### 2.2.4 处理引用计数的原子操作
在内核代码中很多地方使用了引用计数机制，比如page的`_ref_count`和`_map_count`。
* `atomic_add_unless(atomic_t *v, int a, int u)`：比较v的值是否等于u；
* `atomic_inc_not_zero(v)`：比较v的值是否等于0；
* `atomic_inc_and_test(v)`：原子地给v加1，然后判断v的新值是否等于0；
* `atomic_dec_and_test(v)`：原子地给v减1，然后判断v的新值是否等于0；

### 2.2.5 内嵌memory barrier的原子操作函数
* `{}_relaxed`：不内嵌内存屏障原语的原子操作；
* `{}_acquire`：内置了`load-store`内存屏障原语的操作；
* `{}_release`：内置了`store-release`内存屏障源于的操作；

>   除了指定内存屏障共享属性和具体的访问顺序，可以根据**参数指定读写行为、读写方向**：
>
>   -   读内存屏障（Load-Load/Store）指令
>       -   参数后缀为LD：指示在内存屏障指令之前所有的**加载指令**必须完成，但是对存储指令不做要求。
>   -   写内存屏障（Store-Store）指令
>       -   参数后缀为ST：指示在内存屏障指令之前所有的**存储指令**必须完成，但是对加载指令不做要求。
>   -   读写内存屏障（Any-Any）指令
>       -   参数后缀为SY：指示在内存屏障指令之前所有的**存储+加载**指令必须完成。

## 2.4 原子操作底层实现示例
我们以atomic_add为例子，了解一下在Linux上面底层原子操作什么样子的。
```C
#define ATOMIC_OP(op, asm_op)						\
static inline void atomic_##op(int i, atomic_t *v)			\
{									\
	unsigned long tmp;						\
	int result;							\
									\
	asm volatile("// atomic_" #op "\n"				\
"1:	ldxr	%w0, %2\n"						\
"	" #asm_op "	%w0, %w0, %w3\n"				\
"	stxr	%w1, %w0, %2\n"						\
"	cbnz	%w1, 1b"						\
	: "=&r" (result), "=&r" (tmp), "+Q" (v->counter)		\
	: "Ir" (i));							\
}					
```
使用了内嵌汇编的方式实现，主要使用了ldxr和stxr独占访问内存的指令。除此之外，对于独占比较交换，还使用了`cas`等。

# 3. 内存屏障

ARM架构有3类内存屏障指令：
>ARMv8-A提供了3条内存屏障指令，这些屏障指令范围
>
>-   数据存储屏障（Data Memory Barrier, DMB）
>-   数据同步屏障（Data Synchronization Barrier, DSB）
>-   指令同步屏障（Instruction Synchronization Barrier, ISB）
>-   融入到新的加载存储指令的，单方向的内存屏障（acquire\release\load-acquire\store-release）

**Linux内核在底层则使用这些指令完成自己的接口**：

| 接口函数 | 描述 |
| ---- | ---- |
|barrier()|编译优化屏障，阻止编译器为了性能优化对指令进行重排|
|mb()|包括读写在内的内存屏障指令，用于SMP和UP|
|rmb()|读内存屏障，用于SMP和UP|
|wmb()|写内存屏障，用于SMP和UP|
|smp_mb()|用于SMP场景的内存屏障。对于UP场合不存在
内存顺序问题，在UP场合就是barrier()|
|smp_rmb()|用于SMP场合的读内存屏障|
|smp_wmb()|用于SMP场合的写内存屏障|
|smp_read_barrier_depends()|读依赖屏障|

在内核代码中这些函数实现很简单：

```C
#define mb() dsb(sy)
#define rmb() dsb(ld)
#define wmb() dsb(st)

#define dma_rmb() dmb(oshld)
#define dma_wmb() dmb(oshst)
```

# 4. 自旋锁
上一节我们说了原子操作的应用范围，临界区小到了一个变量，而当我们操作链表时候，或者临界区内是有函数调用的时候，这时候就该考虑锁机制了。谈起用户空间的自旋锁，pthread在用户空间提供了pthread_spinlock这类的函数，我们闭着眼睛就能说出spinlock的特性：
* 特性一：try锁的时候十分吃cpu（忙等待）；
* 特性二：不支持调度（不支持抢占），所以临界区不能有睡眠（死锁）；
* 特性三：**中断上下文可以使用spinlock**；
* 特性四：临界区不能太大，影响CPU性能；

## 4.1 经典自旋锁
spinlock和CPU架构息息相关。早起的自旋锁十分简单，描述自旋锁的机构提就是用一个变量表示，但这个方法性能十分低下，因为在激烈竞争锁的时候出现锁的持有者释放后会再次拿到锁，对于多线程的执行分给各个线程的时间片就非常不公平。自从Linux2.6.25内核之后，自旋锁实现了一套基于FIFO的排队机制。

### 4.1.1 spinlock的FIFO机制

定义slock分为next和owner，这个机制就类似于去饭店吃饭。假设饭店开张只有一个位置，初始的next和owner都是0：
* 顾客A到来，我们为顾客编号为0，位置被占所以next + 1，owner是0号顾客
* 顾客B到来，我们为顾客编号为1，位置因为被占，next + 1，owner还是0号的
* 顾客C到来，我们为顾客编号为2，位置因为被占，next + 1，owner还是0号的
* 顾客A离开了，owner + 1，请owner为1的顾客B用餐。

大概就是以上的意思。

### 4.1.2 spinlock

Linux中的经典自旋锁定义如下：

```C
typedef struct {
	volatile unsigned int slock;
} arch_spinlock_t;
///////////////////////////////////////////////////
typedef struct raw_spinlock {
	arch_spinlock_t raw_lock;
#ifdef CONFIG_GENERIC_LOCKBREAK
	unsigned int break_lock;
#endif
#ifdef CONFIG_DEBUG_SPINLOCK
	unsigned int magic, owner_cpu;
	void *owner;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	struct lockdep_map dep_map;
#endif
} raw_spinlock_t;
///////////////////////////////////////////////////
typedef struct spinlock {
	union {
		struct raw_spinlock rlock;

#ifdef CONFIG_DEBUG_LOCK_ALLOC
# define LOCK_PADSIZE (offsetof(struct raw_spinlock, dep_map))
		struct {
			u8 __padding[LOCK_PADSIZE];
			struct lockdep_map dep_map;
		};
#endif
	};
} spinlock_t;
```
spinlock结构体考虑了不同架构体系的支持，所以这里使用arch_spinlock定义。在ARM64的spinlock实现如下：

```C
static inline void arch_spin_lock(arch_spinlock_t *lock)
{
	unsigned int tmp;
	arch_spinlock_t lockval, newval;

	asm volatile(
	/* Atomically increment the next ticket. */
"	prfm	pstl1strm, %3\n"
"1:	ldaxr	%w0, %3\n"
"	add	%w1, %w0, %w5\n"
"	stxr	%w2, %w1, %3\n"
"	cbnz	%w2, 1b\n"
	/* Did we get the lock? */
"	eor	%w1, %w0, %w0, ror #16\n"
"	cbz	%w1, 3f\n"
	/*
	 * No: spin on the owner. Send a local event to avoid missing an
	 * unlock before the exclusive load.
	 */
"	sevl\n"
"2:	wfe\n"
"	ldaxrh	%w2, %4\n"
"	eor	%w1, %w2, %w0, lsr #16\n"
"	cbnz	%w1, 2b\n"
	/* We got the lock. Critical section starts here. */
"3:"
	: "=&r" (lockval), "=&r" (newval), "=&r" (tmp), "+Q" (*lock)
	: "Q" (lock->owner), "I" (1 << TICKET_SHIFT)
	: "memory");
}
```

这个实现就是按照我们上面说的原理，实现了next和owner之间的增加操作。而且在spinlock实现中，依然大量的使用独占内存访问的。

#### 4.1.2.1 raw_spin_lock

在内核代码里面经常看见`raw_spin_lock`的身影，甚至有些代码`spinlock`直接调用`raw_spin_lock`。这里面有个小故事，Linux有一些分支需要打RT-patch，这个RT-patch就是为了让Linux能够支持一些实时操作系统的特性。因此，就有个需求，**允许自旋锁在临界区被抢占，在临界区内允许自旋锁睡眠**。注意哈，普通的自旋锁这个两个完全不允许的，但是实时操作系统是必须要支持的。但是RT-patch如果主意的修改自旋锁调用位置，且一一甄别，那工作量十分巨大了，不过需要抢占的地方工程师们筛查出来仅仅只有一百多处，因此把这部分的自旋锁改为`raw_spin_lock`。

所以，对于实时Linux，`raw_spin_lock`和`spin_lock`是有区别的，raw能够支持在临界区抢占，能够支持在临界区睡眠；但是对于没有实时补丁的Linux而言，raw_spinlock和spinlock完全等价。

#### 4.1.2.2 spin_lock_irq

在需要依赖中断处理器的设备驱动开发，常常遇到这样的问题，驱动某一个链表中有很多设备需要访问和更新，例如open\ioctl等等。操作链表时候通常会把操作链表划为一个临界区，需要用自旋锁来保护。当处于临界区的时候，系统将暂停当前进程去执行中断。假如恰巧中断程序也要处理这个链表，链表被主进程锁住了，中断永远拿不到这把锁，这就是死锁。所以linux提供了spin_lock_irq来对这个问题进行解决，在加锁之前直接`preempt_disable()` 关闭内核抢占，`local_irq_disable()` 关闭中断和抢占。

注意，`preempt_disable`关闭了内核抢占，内核就不会去主动抢占。但编程者需要小心自己抢占自己，这种情况经常非常隐晦的发生在调用函数中。**例如kmalloc()函数放在了临界区，此时有可能因为内存不足而在函数内部进行了睡眠，但kmalloc被我们编程者放在了临界区内，这就发生了死锁现象，除非显式地给定kmalloc(GFP_ATOMIC)指定kmalloc为一个原子操作**。（这个例子我觉得非常好）。

另外，还有`spin_lock_irqsave`这个函数，他比lock_irq多了一个功能就是保存当前中断状态，内部调用了`local_irq_store`，会把CPU的状态寄存器保存起来（例如ARM的CPSR寄存器），这样的目的是防止中断状态被破坏。自旋锁还有个变种就是`spin_lock_bh()`用于处理进程和延迟处理机制导致的并发访问的互斥问题。


## 4.2 MCS锁
MCS锁的名字来源于他的两个发明者Mellor-Crummey和Scott，他俩在1991年创造MCS发表在期刊上面。就算是FIFO自旋锁也存在一个比较大的问题，就是slock值是共享的。如果在单核系统里面没有任何的问题，但是在SMP和NUMA多核系统里面，多核共同修改一个share属性的slock值就会因为cache颠簸的问题，导致性能下降。这是由于MESI缓存一致性协议参与多核之间的cache一致性问题引入的。锁在多核使用的过程中，使得snooping control unit多次参与CPU数据和cache的多次失效。MCS锁的引入就是解决这个问题，原理是在本地CPU上面各自建立锁的副本(PerCPU变量[^3])，只对本地操作，再进行多核的同步。这样就可以避免snooping control unit参与cache的一致性维护。这个锁被称为**Queued SpinLock, QSpinLock**。

这里暂时不展开了。后面有时间用到之后，再去理解如何设计的！

# 5. 信号量和互斥锁

我们在用户空间知道信号量和互斥锁有着相似的特性，但是也有一些差别。它们在等待锁的释放的时候都是进入睡眠等待的，它们是支持调度的；而区别是，信号量是同步的机制，是存在同步深度的（“洗手间理论”，信号量相当于一个N人的洗手间），而互斥量是一个布尔型的互斥锁。在功能上，信号量的一级深度可以实现和互斥锁一模一样的功能。但是需要注意，**信号量内部实现数据结构比互斥锁复杂**，注定是比互斥锁低效的，**速度快是互斥锁的广泛使用的原因**，并且互斥锁的一些特性也用到了信号量身上，比如“乐观自旋”（Optimistic spinning）。

**总得来说，互斥锁比信号量高效的多，体现在：**
* 互斥锁最先实现自旋等待机制；
* 互斥锁在睡眠之前尝试获取锁；
* 互斥锁实现了MCS锁来避免因为多个CPU竞争而导致CPU的cache line颠簸问题；

spinlock和（信号量+互斥锁）的应用场景我们可以拿这个例子举例：**最典型的消费者和生产者线程**。例如，我们去餐厅吃饭，可能没有位置了，但看着人数也不多，就整个人坐在餐厅门口等着有位置空闲，这个场景可以用spinlock解释，一种让我们整个人都等在那里的，短时间能够忍受的；而互斥锁和信号量更像是网络预订，我们去4S店预订汽车，发现汽车没有了，需要联系厂家制造，你会和4S店的店员讲，我先排队，等着到货的时候给我打电话，然后你离开4S店正常生活吃饭，等着有一天店员给你打电话之后你再去买车。这两个生活场景可以解释spinlock和mutex（sem）的应用场景。没有人会在4s店门口拿个板凳等着车到了是吧。因此**，spinlock用于那种短时的值的你去等的场景，而mutex和信号量常常配合完成变量等通知再去取资源**。

## 5.1 信号量
在Linux定义的信号量的数据结构如下：
```C
struct semaphore {
	raw_spinlock_t		lock;
	unsigned int		count;
	struct list_head	wait_list;
};
```

* LOCK是自旋锁变量，用于对信号量数据结构的count和wait_list提供保护；
* COUNT：表示允许进入临界区的执行路径个数；
* WAIT_LIST：表示管理所有该信号上睡眠的进程，没有成功获取锁的进程会睡眠在这个链表上；等待队列在# [0x23_LinuxKernel_内核活动（三）中断体系结构（中断下文）](https://github.com/carloscn/blog/issues/70#top) 等待队列与完成变量中有介绍，请参考。

操作信号量的接口：
* `down`
* `down_interruptible`
* `down_trylock`
* `down_timeout`
相应的还有`up_{ , interruptible, trylock, timeout}`函数。

## 5.2 互斥锁
在Linux定义的互斥锁的数据结构如下：
```C
struct mutex {
	/* 1: unlocked, 0: locked, negative: locked, possible waiters */
	atomic_t		count;
	spinlock_t		wait_lock;
	struct list_head	wait_list;
#if defined(CONFIG_DEBUG_MUTEXES) || defined(CONFIG_MUTEX_SPIN_ON_OWNER)
	struct task_struct	*owner;
#endif
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
	struct optimistic_spin_queue osq; /* Spinner MCS lock */
#endif
#ifdef CONFIG_DEBUG_MUTEXES
	void			*magic;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	struct lockdep_map	dep_map;
#endif
};
```

* count： 原子计数，1表示锁是open状态；0表示被锁被持有着；
* wait_lock：自旋锁，用于保护wait_list睡眠等待队列；
* wait_list：用于管理互斥锁上睡眠的所有进程，没有成功获取锁会被加入到这个链表中睡眠；
* owner: 打开 `CONFIG_MUTEX_SPIN_ON_OWNER`选项才会有owner，用于指示锁持有着的PCB；
* osq：用于实现MCS锁。

因为互斥锁的简洁性和高效性，所以使用互斥锁要更严格，需要注意以下条件：
* 同一时刻只有一个线程可以持有互斥锁；
* 只有锁持者可以解锁。不能在一个进程中互相持有互斥锁，而在另一个进程中释放他；
* 不允许递归加解锁；
* 当进程持有互斥锁的时候，不可以退出；
* 互斥锁必须用官方的API来初始化；
* 互斥锁可以睡眠，所以不允许在中断服务函数或者内核延迟机制中使用。

在工程中**spinlock和mutex选择原则**如下：
* 中断上下文中，毫不犹豫的使用spinlock；
* 临界区有睡眠、隐含睡眠的动作及内核的API的时候，应该避免自旋锁；

另外，除了满足不了上面互斥锁的约束条件，否则都优先使用互斥锁。

# 6. RCU
我们这里简要介绍一些RCU的功能，不再详细描述。RCU（Read-Copy Update），是Linux内核的一种同步机制。无论是自旋锁、互斥锁、信号量，其内部都直接或者间接的使用原子操作，而在架构上原子操作的monitor机制会过多的访问内存，导致性能下降。而且信号量还有一个缺点，只允许多个读者存在，不允许读者和写者同时存在。RCU机制的目标就是为了降低同步开销，让读操作没有任何负担，不需要使用原子操作和内存屏障指令。

RCU可以参考文献[^4]，也可以在内核代码/documents/RCU/wahtisRCU.txt路径找到，这里就不做过多的介绍了。


# X. 附录
## 内核抢占
>把内核看作必须满足两种请求的侍者：一种来自中断，另一种来自用户态进程发出的系统调用或异常。前者的优先级更高。侍者提供的服务对应于 CPU 处于内核态时所执行的代码。如果 CPU 在用户态执行，则认为侍者处于空闲状态。
>## 内核抢占
> 如果进程正在执行内核函数时，即在内核态运行，允许发生内核切换，这个内核就是抢占的。
> 
>**Linux 中的抢占类型**
>
>-   **计划性进程切换**：无论在抢占内核还是非抢占内核中，运行在内核态的进程都可以自动放弃 CPU，比如等待资源。
>-   **强制性进程切**换：抢占式内核可响应引起进程切换的异步事件（如唤醒高优先权进程的中断处理程序）。
>-   **所有的进程切换都由 switch_to 完成**。
>
>**内核抢占的好处**  
>
>使内核可抢占的目的是降低一个进程被另一个运行在内核态进程延迟的风险。
>
>**什么时候禁止抢占**  
>
>当被 current_thread_info() 宏所引用的 thread_info 描述符的 preempt_count 字段大于 0 时，禁止内核抢占，存在以下情况：
>1.  内核正在执行中断服务例程。
>2.  可延迟函数被禁止（当内核正在执行软中断或 tasklet 时经常如此）。
>3.  将抢占计数器设置为正数，显示禁用内核抢占。
>所以，只有当内核正在执行异常处理程序（尤其时系统调用），且内核抢占没有被显式禁用，才可抢占内核。
>
>**内核抢占相关的宏、函数**  
>
>preempt_enable() 宏递减抢占计数器，检查 TIF_NEED_RESCHED 是否被设置。
>
>**内核抢占的时机**
> -   结束内核控制路径时（通常是一个中断处理程序）
> -   异常处理程序调用 preempt_enable() 重新允许内核抢占时。
> -   启用可延迟函数时。


# Ref
[^1]:[内核控制路径，内核同步，中断，异常--x86](https://blog.csdn.net/opencpu/article/details/6757590)
[^2]:[深入理解 Linux 内核—内核同步](https://aqzt.com/18600.html)
[^3]:[PERCPU变量实现](https://zhuanlan.zhihu.com/p/260986194)
[^4]:[What is RCU, Fundamentally?](https://lwn.net/Articles/262464/)