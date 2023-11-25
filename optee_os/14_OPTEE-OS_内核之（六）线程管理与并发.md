# 14_OPTEE-OS_内核之（六）线程管理与并发
OP-TEE中使用线程的方式来管理当前系统中需要运行的任务。当TA被调用时，OP-TEE都会使用一个线程空间来运行执行流程，待调用完成后，该线程的状态将会被重置，以备后续被再次调用。本章将详细介绍OP-TEE中线程管理的相关内容。

# 1. OP-TEE中的线程
OP-TEE中的每一个线程作为一个任务的运行载体。OP-TEE中定义了一个线程的数组，线程数组中的每一个元素都表示一个单独的线程空间。该数组定义在optee_os/core/arch/arm/kernel/thread.c文件中，其内容如下：

`struct thread_ctx threads[CFG_NUM_THREADS]; `

该定义并不是动态分配，而是通过静态定义，因此OP-TEE中并没有线程的创建一说，可通过修改CFG_NUM_THREADS来控制OP-TEE中支持的线程的最大个数。当CA端触发了安全监控模式调用（smc）时，OP-TEE会从该数组中找寻到可用的线程元素作为一个任务。如果REE侧触发的安全监控模式调用（smc）是由RPC引起的，OP-TEE会直接使用参数中的线程ID值找到对应的线程上下文，然后执行恢复操作继续执行该线程，该线程ID的值是OP-TEE发起RPC请求时的线程ID。

**多核**

由于OP-TEE支持多核处理安全监控模式调用（smc）（即CPU中的每一个核都可以用来处理安全监控模式调用），故在OP-TEE中还存在另外一个数组变量：

`static struct thread_core_local thread_core_local[CFG_TEE_CORE_NB_CORE]`

OP-TEE的线程数组是共用，即CPU中的所有核共用线程数组。thread_core_local数组中的每一个元素表示一个核的相关信息，元素中的tmp_stack_va_end用于指定每个ARM核的栈空间，curr_thread用于表示当前核使用的是哪个线程空间。

当CA触发安全监控模式调用（smc）来调用TA中的命令时，OP-TEE会使用一个线程来完成对该安全监控模式调用（smc）的处理。而如果CA调用的是动态的TA，则该线程最终需要切到用户空间去执行，而在进入到用户空间之前会重新设定该线程的栈空间地址。

# 2. 线程状态切换
OP-TEE中的每个线程都具有三种状态，OPTEE通过判定每个线程的状态来决定该线程是否可用。OP-TEE中线程的三种状态及含义如表所示。

| 线程状态 | 表示状态值           | 含义                                  |
| ---------- | ---------------------- | ------------------------------------- |
| Free态     | THREAD_STATE_FREE      | 标记线程处于空闲状态可以被重复使用    |
| Suspend态  | THREAD_STATE_SUSPENDED | 标记线程处于suspend状态，不可以被free |
| Active态   | THREAD_STATE_ACTIVE    | 标记线程处于运行状态                  |

OP-TEE使用枚举变量thread_state来表示当前线程的状态，枚举中的值就是表中的“表示状态的值”一栏中的内容。线程状态之间的切换关系如图所示：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221006174104.png" width="55%" /></div>

线程状态的切换是通过设定线程的status成员变量来实现，在OP-TEE对状态的切换操作进行了封装，切换是使用汇编来实现的。

**Free态->Active态**
OP-TEE启动时所有的线程都处于Free态（可用状态）。当CA调用TA时就会从线程数组中找到一个可用的线程空间用于运行该调用的任务。通过调用thread_alloc_and_run函数可将Free态的线程设置成Active态。

**Active -> Suspend**
OP-TEE启动时所有的线程都处于Free态（可用状态）。当CA调用TA时就会从线程数组中找到一个可用的线程空间用于运行该调用的任务。通过调用thread_alloc_and_run函数可将Free态的线程设置成Active态。

**Suspend -> Active**
调用thread_resume函数可将挂起的线程切换到运行状态，该函数在OP-TEE接收RPC请求返回的数据时被调用。该函数的实现在前面章节中已有介绍，在此就不再赘述。其原理就是恢复该线程在挂起之前所有寄存器的值。PC的值作为线程恢复时程序运行的入口地址。

**Active -> Free**
当线程处理完所有操作后就需要将线程重置，释放掉分配的系统资源，并将该线程重新设置成Free态以便被其他任务使用。这些操作是通过调用thread_state_free函数来实现的。

# 3. 线程运行时的资源
线程在运行过程中需要很多资源的支持，其中最重要的资源就是栈空间。当调用动态TA时，线程会切换到用户空间运行，OP-TEE为每个线程指定了内核空间栈，即OP-TEE中的所有线程都具有独立的内核栈，如果线程需要进入到用户空间，也会具有独立的用户空间栈。

## 3.1 线程数据结构
OP-TEE使用thread_ctx结构体变量来表示每个线程的基本信息，该结构体的定义如下：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221006175540.png)

线程执行挂起时会将cpsr、spsr、pc以及其他寄存器的值保存到线程的regs变量中，以备在恢复线程时直接通过regs中的数据恢复到挂起之前的状态。stack_va_end是线程在内核态的栈底地址，当线程切换到用户空间时需要重新设置栈空间。

## 3.2 OPTEE分配内核栈
如果OP-TEE不支持PAGER，则会建立三个栈空间，这三个栈空间的作用和说明如表所列：

| 栈名         | 用途                                 |
| ------------ | ------------------------------------ |
| stack_tmp    | ARMv7中Monitor模式程序运行时的栈空间 |
| stack_abt    | OP-TEE产生异常时的栈空间             |
| stack_thread | OP-TEE中thread运行时的内核栈空间     |

这三个栈使用DECLARE_STACK来进行定义，在编译时会被保存到.nozi_stack段中，其定义在thread.c文件中，内容如下：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221006175849.png)

每个线程都具有独立的内核栈空间，该栈空间是从nozi_stack中划分出来的。OP-TEE启动时会调用init_thread_stacks函数为OP-TEE支持的每个线程指定内核栈空间，并将该栈的地址赋值给线程结构体中的stack_va_end成员，其内容如下：
```C
bool thread_init_stack(uint32_t thread_id, vaddr_t sp)
{
	if (thread_id >= CFG_NUM_THREADS)
		return false;
	threads[thread_id].stack_va_end = sp;
	return true;
}

#ifdef CFG_WITH_PAGER
static void init_thread_stacks(void)
{
	size_t n = 0;

	/*
	 * Allocate virtual memory for thread stacks.
	 */
	for (n = 0; n < CFG_NUM_THREADS; n++) {
		tee_mm_entry_t *mm = NULL;
		vaddr_t sp = 0;
		size_t num_pages = 0;
		struct fobj *fobj = NULL;

		/* Find vmem for thread stack and its protection gap */
		mm = tee_mm_alloc(&tee_mm_vcore,
				  SMALL_PAGE_SIZE + STACK_THREAD_SIZE);
		assert(mm);

		/* Claim eventual physical page */
		tee_pager_add_pages(tee_mm_get_smem(mm), tee_mm_get_size(mm),
				    true);

		num_pages = tee_mm_get_bytes(mm) / SMALL_PAGE_SIZE - 1;
		fobj = fobj_locked_paged_alloc(num_pages);

		/* Add the region to the pager */
		tee_pager_add_core_region(tee_mm_get_smem(mm) + SMALL_PAGE_SIZE,
					  PAGED_REGION_TYPE_LOCK, fobj);
		fobj_put(fobj);

		/* init effective stack */
		sp = tee_mm_get_smem(mm) + tee_mm_get_bytes(mm);
		asan_tag_access((void *)tee_mm_get_smem(mm), (void *)sp);
		if (!thread_init_stack(n, sp))
			panic("init stack failed");
	}
}
#else
static void init_thread_stacks(void)
{
	size_t n;

	/* Assign the thread stacks */
	for (n = 0; n < CFG_NUM_THREADS; n++) {
		if (!thread_init_stack(n, GET_STACK_BOTTOM(stack_thread, n)))
			panic("thread_init_stack failed");
	}
}
#endif /*CFG_WITH_PAGER*/
```

## 3.3 线程运行于用户空间资源
CA触发安全监控模式调用（smc）时，OPTEE都会使用一个线程来完成具体操作。如果调用的是动态TA，则该线程最终会切换到OP-TEE的用户空间，调用具体TA的接口来完成处理。OP-TEE使用user_ta_ctx结构体变量保存该调用在用户空间的所有信息，其中就包括了线程在用户空间运行时的栈信息。

只有当CA调用的是动态TA时才会创建该结构体变量。创建该变量时，entry_func会被初始化成该TA镜像的ta_head段中的entry.ptr64的值，该值在编译生成TA镜像时被设定成__utee_entry。

**用户空间中使用的栈空间是从tee_mm_sec_ddr内存池中分配出来的**，该内存池属于MEM_AREA_TA_RAM内存区域，该区域是由OPTEE分配，用于运行TA镜像。

用**户空间使用的堆空间是在user_ta_header.c文件中定义的ta_heap数组变量，其大小由TA_DATA_SIZE宏决定**。该宏定义在每个TA的user_ta_header.h文件中，ta_heap会被编译到TA镜像文件的BSS段中，加载TA镜像到OP-TEE的过程中会使用malloc_add_pool函数将ta_heap作为该TA的堆空间添加到内存池中，在TA中需要使用类似于malloc的函数分配一块内存空间时就会从该内存池中分配所需要的内存。

## 3.4 tee_ta_session结构体

CA调用创建会话操作时会创建一个会话建立CA与特定TA之间的通道。会话是tee_ta_session的结构体变量，创建好的变量会被添加到OP-TEE用于保存当前系统中已经被创建的会话的链表tee_open_sessions中。待下次CA调用时，只要提供会话ID就可从tee_open_session链表中查找到对应的会话实体，进行调用TA命令的操作。添加到该链表中的是tee_ta_session结构体变量。

# 4. 线程运行时资源的使用关系

使用线程来处理CA对TA的调用请求时都会使用到上一节中的所有资源。如果调用的是动态TA，还会使用到该TA的user_ta_ctx的内容，线程的运行与上述资源的关系如图所示。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221006181614.png)

线程切换到用户空间之前，使用thread_ctx实体中的stack_va_end作为该线程运行时的内核栈空间，并且通过UUID从tee_open_session链表中查找到需要被调用的TA的tee_ta_session实体。该实体会保存TA的操作上下文信息，该上下文信息是在CA调用创建会话操作时被分配和初始化的。保存在user_ta_ctx的tee_ta_ctx结构体成员中，通过tee_ta_ctx实体就能够找到在用户空间使用的user_ta_ctx的内容。user_ta_ctx实体中会指定线程在用户空间使用的用户空间栈地址。这些结构体之间的包含关系如图所示。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221006181753.png" width="60%" /></div>

# 5. OP-TEE中线程的调度
## 5.1 调度
OP-TEE支持SMP，即支持多核来处理安全监控模式调用，但任何时候一个核上同时只会有一个线程在运行。CPU中的所有核共享OP-TEE的线程数组，即如果某个线程被一个核挂起了，待需要被恢复继续运行时，任何CPU的核都可以通过线程ID继续运行该线程。

OP-TEE中的线程调度并不像Linux一样采取时间片轮转的方式进行。在OP-TEE中，**一个线程分配到ARM核运行之后，其将独占该核的使用权**。除非线程主动挂起或正常世界状态的中断触发状态切换。待线程执行完后会释放掉调用该线程时分配的资源，并将线程空间重置成Free状态，释放掉占使用的ARM核的控制权限。

如果线程在执行过程中主动执行挂起操作，则线程会保存当前线程的资源，并将线程的状态设置成挂起态，然后通过指定当前ARM核的thread_core_local结构体变量中的curr_thread交出ARM核的控制权限。待线程被挂起后，若有实际需求时，CPU中的任何一个核都可以使用该线程的ID来唤醒该线程继续执行。

## 5.2 死锁
死锁对于任何一个系统来说都是很严重的问题，轻则会导致线程被杀死而无法完成任务，重则可能会引起看门狗超时导致系统重启。这对于任何一个系统来说都是不可接受的。故避免死锁现象对于系统的稳定性来说至关重要。

死锁即一个线程占用了资源A，同时需要获取到资源B之后才会释放资源A，而另一个线程占用了资源B，而且只有获取到资源A之后才会释放资源B，这样导致线程one和线程two都无法正确地获取到资源继续执行。

死锁一般出现在对互斥体或自旋锁的使用过程中，尤其是一个线程需要获取多个互斥体或者自旋锁的情况下。有效地防止死锁现象的做法是，如果一个线程需要使用多个互斥体或者自旋锁来完成一些操作时，其他的线程在使用这些互斥体或者自旋锁时需要按照同样的顺序来获得，即在使用互斥体或者自旋锁时统一按照相同的顺序进行。而且最好互斥体和自旋锁的锁住和解锁动作在同一个函数中完成。这点在编写TA程序使用互斥体或者自旋锁时需要格外注意。

# 6. 总结
本章介绍了OP-TEE中线程的相关信息和状态切换的实现。注意CPU中的所有核共享OP-TEE的线程数组，当执行动态TA时，线程进入到用户空间之后会使用新的堆栈空间，在某种意义上可以理解成各个TA之间是相互隔离的。