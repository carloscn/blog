# 0x24_LinuxKernel_进程（一）进程的管理（生命周期、进程表示）

在UNIX操作系统下运行的应用程序、服务器以及其他程序都被称为**进程**。每个进程都在CPU的虚拟内存上分配地址空间。每个进程的地址空间都是独立的，因此进程和进程不会意识对方的存在，觉得自己是CPU上唯一运行的进程。从处理器的角度，系统上有几个CPU，就最多能运行几个进程。但Linux是一个多任务的系统，内核为了支持多任务需要在不同的进程之间切换，这样从时间维度的划分造成了多进程同时运行的假象。

内核借助CPU的帮助，负责进程切换，欺骗进程独占了CPU。这就需要在切换进程之前保存进程的所有状态及相关要素，并将进程置于IDLE状态。在进程切换回来之前，这些资源和信息需要恢复到CPU和内存上面。这些资源被成为**进程上下文**。

内核除了要保存进程一些必要信息之外。还需要合理分配每一个进程的时间片，通常重要的进程得到CPU的时间多一点，次要的进程时间少一点。这个时间分配的过程成为**调度**。

本文对于进程的表述的结构如下所示：

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora%E8%BF%9B%E7%A8%8B%E7%AE%A1%E7%90%86%E4%B8%8E%E8%B0%83%E5%BA%A61.svg" width="100%" /> </div>


# 1. 用户空间定义
这部分介绍一些从用户空间视角来观摩进程的概念。

## 1.1 进程树
Linux对进程的表述采用了一种层次结构，每个进程都依赖于一个父进程。内核启动`init`程序作为第一个进程，该进程负责进一步的系统初始化操作，并显示视提示符和登陆界面。因此`init`进程被称为进程树的***根***，**所有的进程都直接或者间接起源自该进程**。如下使用*pstree*程序的输出所示。

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220807124816.png" width="100%" /> </div>

这个树形结构的进程上面和新进程的创建方式密切相关。UNIX操作系统有两种创建新进程的机制分别是`fork()`和`exec()`。

`fork()`可以创建当前进程的一个副本且有独立的PID号码（父进程和子进程只有PID号码不同）。Linux使用了一个中所周的的技术来使`fork`更高效，那就是写时复制（copy on write）操作。

`exec()`将一个新程序加载到当前的内存中执行，旧程序的内存页面被逐出，其内容被替换为新数据，然后开始执行新的程序。

## 1.2 线程

除了重量级进程（有时候也成为UNIX进程），还有一个轻量级*进程*（线程）。本质上，一个进程包含若干个线程，这些线程使用着同样的地址空间，共享同样的数据和资源，因此除了使用同步的方法保护共享的资源之外，没有额外的通信机制了。这也是线程和进程的差别。

Linux使用`clone()`的方法创建线程。其工作方式类似于`fork`，但启用了精准的检查，以确认那些资源与父进程共享、哪些资源为线程独立创建。

## 1.3 用户空间进程编程

linux提供一些关于进程的系统调用，例如`fork()`、`getpid()`、`getppid()`、`wait()`、`waitpid()`，可以创建进程，对父进程和子进程流程的一些控制。

### 1.3.1 fork进程
https://github.com/carloscn/clab/blob/master/linux/test_pid/test_pid.c

```C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>

#define debug_log printf("%s:%s:%d--",__FILE__, __FUNCTION__, __LINE__);printf

int main(int argc, char *argv[])
{
    pid_t pid = fork();

    while(1) {
        if (0 == pid) {
            debug_log("Current PID = %d, PPID = %d\n", getpid(), getppid());
            debug_log("I'm Child process!\n");
            sleep(1);
        } else if (pid > 0) {
            debug_log("-------------------> Current PID = %d, PPID = %d\n", getpid(), getppid());
            debug_log("-------------------> I'm Parent process!\n");
            sleep(2);
        } else {
            debug_log("fork() failed \n");
        }
    }

    return 0;
}

```
两个进程会竞争一个终端输出：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220807180928.png)
但是后台是被fork了两个进程：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220807181014.png)

### 1.3.2 zombie进程
fork进程之后，子进程的处理程序中直接退出，而父进程处理程序中没有任何等待子进程的操作。这时候子进程就会成为*僵尸进程*。僵尸进程会被init 0收养，等待父进程退出之后，僵尸进程会被init 0进程释放掉。
https://github.com/carloscn/clab/blob/master/linux/test_pid/test_pid_zombie.c
```C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>

#define debug_log printf("%s:%s:%d--",__FILE__, __FUNCTION__, __LINE__);printf

int main(int argc, char *argv[])
{
    pid_t pid = fork();
    u_int16_t count = 0;

    while(1) {
        if (0 == pid) {
            debug_log("Current PID = %d, PPID = %d\n", getpid(), getppid());
            debug_log("I'm Child process\n");
            // just exit child process, and parent process don't call waitpid(),
            // which to create zombie process
            sleep(1);
            count ++;
            if (count > 10) {
                debug_log("child will exit.\n");
                exit(0);
            }
        } else if (pid > 0) {
            // debug_log("-------------------> wait for child exiting.\n");
            // WNOHANG is non block mode.
            // WUNTRACED is block mode.
            // mask the next line code, the zombie process will be generated.
            // pid_t t_pid = waitpid(pid, NULL, WUNTRACED);
            debug_log("-------------------> Current PID = %d, PPID = %d\n", getpid(), getppid());
            debug_log("-------------------> I'm Parent process!\n");
            sleep(2);
        } else {
            debug_log("fork() failed \n");
        }
    }
    return 0;
}

```
子进程退出之后，它成为僵尸进程。
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220807181556.png)

父进程使用`waitpid`可以观测到子进程是否已经退出，以及时回收进程资源。

# 2. 进程的管理和调度

## 2.1 进程优先级

进程优先级粗暴分为实时进程和非实时进程：
* **硬实时进程：**有严格的时间限制/**Linux不支持硬实时**/一些linux旁支版本RTLinux支持硬实时，主要原因是调度器没有做进内核中，内核作为一个独立的进程处理一些不仅要的任务。
* Linux的任务优先满足吞吐量，所以弱化了进程调度。但是这些年人们也在降低内核延迟上面做了很多研究，比如提出可抢占机制、实时互斥内核锁还有完全公平调度器。
* **软实时进程**：类似与写CD这种工作种类的进程，就算是写进程被暂时中断也不会造成宕机之类的风险操作。
* **普通进程**： 没有具体的时间约束限制，但是会分配优先级来区分重要性。

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220807183447.png" width="50%" /> </div>

进程优先级模型转换为时间片长度，优先级越高的占用的时间片越多。这种方案被称为**抢占式多任务处理(preemptive multitasking)**，即各个进程都被分配到一定的时间片用来执行。时间到了之后，内核会从当前进程收回控制权，让不同的进程运行。抢占的时候所有的CPU寄存器的内容和页表都会保存起来，因此其结果并不会因为进程的切换而丢失。

## 2.2 进程的生命周期

### 2.2.1 状态机

一个进程是有状态机可以表示的，我们可以分为：**运行(running)、等待(waiting)、休眠(sleeping)和终止(stopped) **四个状态，如图所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220810092859.png)

状态机的**切换是调度器**来完成，切换条件可以解释为：

**① Running -> Sleeping**：如果进程必须等待事件，则状态变为“运行”直接切换到“睡眠”，但这个过程是不可逆的。
**② Running -> Waiting**: 调度器决定从进程收回资源的时候，过程状态从R变为W。
**③ Sleeping -> Waiting**：正如①所说，Sleeping没有办法直接变回Running，必须有一个中间状态waiting。
**④ Waiting -> Running**: CPU此时把资源分配给其他进程。在调度器授予CPU时间之前，进程一直保持waiting的状态。在分配CPU之后，其状态变为Running。

不在于周期范围内的进程:“**僵尸进程（zombie）**”：子进程被KILL信号（SIGTERM和SIGKILL）杀死，父进程没有调用wait4()。正常流程是，子进程被KILL，父进程调用wait4系统调用，通知内核子进程已死。处于僵尸进程状态进程：
  * 还会在进程表中（ps/top能刷出进程）
  * 占用很少的资源
  * 重启后才能刷掉该进程
  
### 2.2.2 抢占式多任务处理
* 进程执行分为内核态和用户态，最大区别在于，内存地址区域访问划分不同。
* 进内核态方法一：如果用户态进程进入内核态：访问共享数据，文件系统空间，必须通过**系统调用**，才能进入到内核态。
* 进内核态方法二：中断触发进入内核态。
* 中断触发和系统调用可以使用户态进入到内核态，但用户态是主动调用的，中断是外部触发的。
  * **用户态 < 核心态 < 中断**
* 进程抢占层次：
  * 普通进程总会被抢占
  * 进程处于内核态（或者普通进程处于系统调用期间）无法被任何进程抢占，但中断可以抢占内核态，因此要求**中断不能占用太长时间**。
  * **中断可以暂停用户态进程，甚至和内核进程都可以被暂定**。
  * **内核抢占（kernel preemption）被加入到内核配置中**，允许进程在紧急状态下抢占用户进程或者内核进程。优点：减少等待时间，让进程更“平滑”的执行；缺点：增加内核复杂程度。

## 2.3 进程的表示

Linux有一套自己对于进程管理的方式，还有调度器，调度器可以理解为真正去执行和指挥进程如何运行的东西，而管理方式可以说是调度器去管理进程的一个笔记计划表（task_struct）在include/sched.h中，这里有如何管理进程，进程命名，编号法等等。

### 2.3.1 task结构体
-Linux 通常把process當做是**task**，**PCB (processing control block)** 通常也稱為 **struct tast_struct** [^6]

https://elixir.bootlin.com/linux/v3.19.8/source/include/linux/sched.h#L1274
```C
struct task_struct {
	volatile long state;	/* -1 unrunnable, 0 runnable, >0 stopped */
	void *stack;
	atomic_t usage;
	unsigned int flags;	/* per process flags, defined below */
	unsigned int ptrace;

#ifdef CONFIG_SMP
	struct llist_node wake_entry;
	int on_cpu;
	struct task_struct *last_wakee;
	unsigned long wakee_flips;
	unsigned long wakee_flip_decay_ts;

	int wake_cpu;
#endif
	int on_rq;

	int prio, static_prio, normal_prio;
	unsigned int rt_priority;
	const struct sched_class *sched_class;
	struct sched_entity se;
	struct sched_rt_entity rt;
#ifdef CONFIG_CGROUP_SCHED
	struct task_group *sched_task_group;
#endif
	struct sched_dl_entity dl;

#ifdef CONFIG_PREEMPT_NOTIFIERS
	/* list of struct preempt_notifier: */
	struct hlist_head preempt_notifiers;
#endif

#ifdef CONFIG_BLK_DEV_IO_TRACE
	unsigned int btrace_seq;
#endif

	unsigned int policy;
	int nr_cpus_allowed;
	cpumask_t cpus_allowed;

#ifdef CONFIG_PREEMPT_RCU
	int rcu_read_lock_nesting;
	union rcu_special rcu_read_unlock_special;
	struct list_head rcu_node_entry;
#endif /* #ifdef CONFIG_PREEMPT_RCU */
#ifdef CONFIG_PREEMPT_RCU
	struct rcu_node *rcu_blocked_node;
#endif /* #ifdef CONFIG_PREEMPT_RCU */
#ifdef CONFIG_TASKS_RCU
	unsigned long rcu_tasks_nvcsw;
	bool rcu_tasks_holdout;
	struct list_head rcu_tasks_holdout_list;
	int rcu_tasks_idle_cpu;
#endif /* #ifdef CONFIG_TASKS_RCU */

#if defined(CONFIG_SCHEDSTATS) || defined(CONFIG_TASK_DELAY_ACCT)
	struct sched_info sched_info;
#endif

	struct list_head tasks;
#ifdef CONFIG_SMP
	struct plist_node pushable_tasks;
	struct rb_node pushable_dl_tasks;
#endif

	struct mm_struct *mm, *active_mm;
#ifdef CONFIG_COMPAT_BRK
	unsigned brk_randomized:1;
#endif
	/* per-thread vma caching */
	u32 vmacache_seqnum;
	struct vm_area_struct *vmacache[VMACACHE_SIZE];
#if defined(SPLIT_RSS_COUNTING)
	struct task_rss_stat	rss_stat;
#endif
/* task state */
	int exit_state;
	int exit_code, exit_signal;
	int pdeath_signal;  /*  The signal sent when the parent dies  */
	unsigned int jobctl;	/* JOBCTL_*, siglock protected */

	/* Used for emulating ABI behavior of previous Linux versions */
	unsigned int personality;

	unsigned in_execve:1;	/* Tell the LSMs that the process is doing an
				 * execve */
	unsigned in_iowait:1;

	/* Revert to default priority/policy when forking */
	unsigned sched_reset_on_fork:1;
	unsigned sched_contributes_to_load:1;

#ifdef CONFIG_MEMCG_KMEM
	unsigned memcg_kmem_skip_account:1;
#endif

	unsigned long atomic_flags; /* Flags needing atomic access. */

	pid_t pid;
	pid_t tgid;

#ifdef CONFIG_CC_STACKPROTECTOR
	/* Canary value for the -fstack-protector gcc feature */
	unsigned long stack_canary;
#endif
	/*
	 * pointers to (original) parent process, youngest child, younger sibling,
	 * older sibling, respectively.  (p->father can be replaced with
	 * p->real_parent->pid)
	 */
	struct task_struct __rcu *real_parent; /* real parent process */
	struct task_struct __rcu *parent; /* recipient of SIGCHLD, wait4() reports */
	/*
	 * children/sibling forms the list of my natural children
	 */
	struct list_head children;	/* list of my children */
	struct list_head sibling;	/* linkage in my parent's children list */
	struct task_struct *group_leader;	/* threadgroup leader */

	/*
	 * ptraced is the list of tasks this task is using ptrace on.
	 * This includes both natural children and PTRACE_ATTACH targets.
	 * p->ptrace_entry is p's link on the p->parent->ptraced list.
	 */
	struct list_head ptraced;
	struct list_head ptrace_entry;

	/* PID/PID hash table linkage. */
	struct pid_link pids[PIDTYPE_MAX];
	struct list_head thread_group;
	struct list_head thread_node;

	struct completion *vfork_done;		/* for vfork() */
	int __user *set_child_tid;		/* CLONE_CHILD_SETTID */
	int __user *clear_child_tid;		/* CLONE_CHILD_CLEARTID */

	cputime_t utime, stime, utimescaled, stimescaled;
	cputime_t gtime;
#ifndef CONFIG_VIRT_CPU_ACCOUNTING_NATIVE
	struct cputime prev_cputime;
#endif
#ifdef CONFIG_VIRT_CPU_ACCOUNTING_GEN
	seqlock_t vtime_seqlock;
	unsigned long long vtime_snap;
	enum {
		VTIME_SLEEPING = 0,
		VTIME_USER,
		VTIME_SYS,
	} vtime_snap_whence;
#endif
	unsigned long nvcsw, nivcsw; /* context switch counts */
	u64 start_time;		/* monotonic time in nsec */
	u64 real_start_time;	/* boot based time in nsec */
/* mm fault and swap info: this can arguably be seen as either mm-specific or thread-specific */
	unsigned long min_flt, maj_flt;

	struct task_cputime cputime_expires;
	struct list_head cpu_timers[3];

/* process credentials */
	const struct cred __rcu *real_cred; /* objective and real subjective task
					 * credentials (COW) */
	const struct cred __rcu *cred;	/* effective (overridable) subjective task
					 * credentials (COW) */
	char comm[TASK_COMM_LEN]; /* executable name excluding path
				     - access with [gs]et_task_comm (which lock
				       it with task_lock())
				     - initialized normally by setup_new_exec */
/* file system info */
	int link_count, total_link_count;
#ifdef CONFIG_SYSVIPC
/* ipc stuff */
	struct sysv_sem sysvsem;
	struct sysv_shm sysvshm;
#endif
#ifdef CONFIG_DETECT_HUNG_TASK
/* hung task detection */
	unsigned long last_switch_count;
#endif
/* CPU-specific state of this task */
	struct thread_struct thread;
/* filesystem information */
	struct fs_struct *fs;
/* open file information */
	struct files_struct *files;
/* namespaces */
	struct nsproxy *nsproxy;
/* signal handlers */
	struct signal_struct *signal;
	struct sighand_struct *sighand;

	sigset_t blocked, real_blocked;
	sigset_t saved_sigmask;	/* restored if set_restore_sigmask() was used */
	struct sigpending pending;

	unsigned long sas_ss_sp;
	size_t sas_ss_size;
	int (*notifier)(void *priv);
	void *notifier_data;
	sigset_t *notifier_mask;
	struct callback_head *task_works;

	struct audit_context *audit_context;
#ifdef CONFIG_AUDITSYSCALL
	kuid_t loginuid;
	unsigned int sessionid;
#endif
	struct seccomp seccomp;

/* Thread group tracking */
   	u32 parent_exec_id;
   	u32 self_exec_id;
/* Protection of (de-)allocation: mm, files, fs, tty, keyrings, mems_allowed,
 * mempolicy */
	spinlock_t alloc_lock;

	/* Protection of the PI data structures: */
	raw_spinlock_t pi_lock;

#ifdef CONFIG_RT_MUTEXES
	/* PI waiters blocked on a rt_mutex held by this task */
	struct rb_root pi_waiters;
	struct rb_node *pi_waiters_leftmost;
	/* Deadlock detection and priority inheritance handling */
	struct rt_mutex_waiter *pi_blocked_on;
#endif

#ifdef CONFIG_DEBUG_MUTEXES
	/* mutex deadlock detection */
	struct mutex_waiter *blocked_on;
#endif
#ifdef CONFIG_TRACE_IRQFLAGS
	unsigned int irq_events;
	unsigned long hardirq_enable_ip;
	unsigned long hardirq_disable_ip;
	unsigned int hardirq_enable_event;
	unsigned int hardirq_disable_event;
	int hardirqs_enabled;
	int hardirq_context;
	unsigned long softirq_disable_ip;
	unsigned long softirq_enable_ip;
	unsigned int softirq_disable_event;
	unsigned int softirq_enable_event;
	int softirqs_enabled;
	int softirq_context;
#endif
#ifdef CONFIG_LOCKDEP
# define MAX_LOCK_DEPTH 48UL
	u64 curr_chain_key;
	int lockdep_depth;
	unsigned int lockdep_recursion;
	struct held_lock held_locks[MAX_LOCK_DEPTH];
	gfp_t lockdep_reclaim_gfp;
#endif

/* journalling filesystem info */
	void *journal_info;

/* stacked block device info */
	struct bio_list *bio_list;

#ifdef CONFIG_BLOCK
/* stack plugging */
	struct blk_plug *plug;
#endif

/* VM state */
	struct reclaim_state *reclaim_state;

	struct backing_dev_info *backing_dev_info;

	struct io_context *io_context;

	unsigned long ptrace_message;
	siginfo_t *last_siginfo; /* For ptrace use.  */
	struct task_io_accounting ioac;
#if defined(CONFIG_TASK_XACCT)
	u64 acct_rss_mem1;	/* accumulated rss usage */
	u64 acct_vm_mem1;	/* accumulated virtual memory usage */
	cputime_t acct_timexpd;	/* stime + utime since last update */
#endif
#ifdef CONFIG_CPUSETS
	nodemask_t mems_allowed;	/* Protected by alloc_lock */
	seqcount_t mems_allowed_seq;	/* Seqence no to catch updates */
	int cpuset_mem_spread_rotor;
	int cpuset_slab_spread_rotor;
#endif
#ifdef CONFIG_CGROUPS
	/* Control Group info protected by css_set_lock */
	struct css_set __rcu *cgroups;
	/* cg_list protected by css_set_lock and tsk->alloc_lock */
	struct list_head cg_list;
#endif
#ifdef CONFIG_FUTEX
	struct robust_list_head __user *robust_list;
#ifdef CONFIG_COMPAT
	struct compat_robust_list_head __user *compat_robust_list;
#endif
	struct list_head pi_state_list;
	struct futex_pi_state *pi_state_cache;
#endif
#ifdef CONFIG_PERF_EVENTS
	struct perf_event_context *perf_event_ctxp[perf_nr_task_contexts];
	struct mutex perf_event_mutex;
	struct list_head perf_event_list;
#endif
#ifdef CONFIG_DEBUG_PREEMPT
	unsigned long preempt_disable_ip;
#endif
#ifdef CONFIG_NUMA
	struct mempolicy *mempolicy;	/* Protected by alloc_lock */
	short il_next;
	short pref_node_fork;
#endif
#ifdef CONFIG_NUMA_BALANCING
	int numa_scan_seq;
	unsigned int numa_scan_period;
	unsigned int numa_scan_period_max;
	int numa_preferred_nid;
	unsigned long numa_migrate_retry;
	u64 node_stamp;			/* migration stamp  */
	u64 last_task_numa_placement;
	u64 last_sum_exec_runtime;
	struct callback_head numa_work;

	struct list_head numa_entry;
	struct numa_group *numa_group;

	/*
	 * numa_faults is an array split into four regions:
	 * faults_memory, faults_cpu, faults_memory_buffer, faults_cpu_buffer
	 * in this precise order.
	 *
	 * faults_memory: Exponential decaying average of faults on a per-node
	 * basis. Scheduling placement decisions are made based on these
	 * counts. The values remain static for the duration of a PTE scan.
	 * faults_cpu: Track the nodes the process was running on when a NUMA
	 * hinting fault was incurred.
	 * faults_memory_buffer and faults_cpu_buffer: Record faults per node
	 * during the current scan window. When the scan completes, the counts
	 * in faults_memory and faults_cpu decay and these values are copied.
	 */
	unsigned long *numa_faults;
	unsigned long total_numa_faults;

	/*
	 * numa_faults_locality tracks if faults recorded during the last
	 * scan window were remote/local. The task scan period is adapted
	 * based on the locality of the faults with different weights
	 * depending on whether they were shared or private faults
	 */
	unsigned long numa_faults_locality[2];

	unsigned long numa_pages_migrated;
#endif /* CONFIG_NUMA_BALANCING */

	struct rcu_head rcu;

	/*
	 * cache last used pipe for splice
	 */
	struct pipe_inode_info *splice_pipe;

	struct page_frag task_frag;

#ifdef	CONFIG_TASK_DELAY_ACCT
	struct task_delay_info *delays;
#endif
#ifdef CONFIG_FAULT_INJECTION
	int make_it_fail;
#endif
	/*
	 * when (nr_dirtied >= nr_dirtied_pause), it's time to call
	 * balance_dirty_pages() for some dirty throttling pause
	 */
	int nr_dirtied;
	int nr_dirtied_pause;
	unsigned long dirty_paused_when; /* start of a write-and-pause period */

#ifdef CONFIG_LATENCYTOP
	int latency_record_count;
	struct latency_record latency_record[LT_SAVECOUNT];
#endif
	/*
	 * time slack values; these are used to round up poll() and
	 * select() etc timeout values. These are in nanoseconds.
	 */
	unsigned long timer_slack_ns;
	unsigned long default_timer_slack_ns;

#ifdef CONFIG_FUNCTION_GRAPH_TRACER
	/* Index of current stored address in ret_stack */
	int curr_ret_stack;
	/* Stack of return addresses for return function tracing */
	struct ftrace_ret_stack	*ret_stack;
	/* time stamp for last schedule */
	unsigned long long ftrace_timestamp;
	/*
	 * Number of functions that haven't been traced
	 * because of depth overrun.
	 */
	atomic_t trace_overrun;
	/* Pause for the tracing */
	atomic_t tracing_graph_pause;
#endif
#ifdef CONFIG_TRACING
	/* state flags for use by tracers */
	unsigned long trace;
	/* bitmask and counter of trace recursion */
	unsigned long trace_recursion;
#endif /* CONFIG_TRACING */
#ifdef CONFIG_MEMCG
	struct memcg_oom_info {
		struct mem_cgroup *memcg;
		gfp_t gfp_mask;
		int order;
		unsigned int may_oom:1;
	} memcg_oom;
#endif
#ifdef CONFIG_UPROBES
	struct uprobe_task *utask;
#endif
#if defined(CONFIG_BCACHE) || defined(CONFIG_BCACHE_MODULE)
	unsigned int	sequential_io;
	unsigned int	sequential_io_avg;
#endif
#ifdef CONFIG_DEBUG_ATOMIC_SLEEP
	unsigned long	task_state_change;
#endif
};

```

成员非常多，弄清楚费劲，可以分类：
* 状态和执行信息，如待决信号、进程的二进制格式种类、进程PID、父进程地址、优先级和CPU时间。
* 分配的虚拟内存信息。
* 身份凭据，如用户ID、组ID和权限等。
* 使用的文件（包括程序代码的二进制文件）
* 线程信息记录，CPU的运行时间数据。
* 与其他进程通信有关的信息。
* 该进程所用的信号处理，用于响应到来的信号。

### 2.3.2 进程状态机

```C
<sched.h>
stcuct task_struct {
	volatile long state;
};
```

* TASK_RUNNING: 处于可运行状态，未必处于CPU cover时间也有可能是等待调度器的状态。
* TASK_INTERRUPTIBLE: 针对某时间或者其他资源的睡眠进程设置的。
* TASK_UNINTERRUPTIBLE：用于内核指示而停用的睡眠进程。不能由外部唤醒，只能由内核亲自唤醒。
* TASK_STOPPED：进程特意停止运行，例如，由调度器暂停。
* TASK_TRACED：本来不是进程状态，用于停止的进程（ptrace机制）与常规停止进程区分开。
* EXIT_ZOMBIE：僵尸状态
* EXIT_DEAD：wait系统调用已发出，解除僵尸状态。

### 2.3.3 进程资源限制

```C
<sched.h>
<resource.h>
struct rlimit {
    unsigned long rlim_cur;
    unsigned long rlim_max;
}
struct task_struct {
	struct rlimite limit;
};
```

* rlim_cur: 进程当前的资源限制，成为软限制(soft limit)
* rlim_max:该限制的最大容许值， 硬限制（hard limit）

用户进程通过setrlimit()系统调用来增减当前限制，最大值不能超过rlim_max；getrlimits()用于检查当前限制。那么设定的资源是什么呢？

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220810114549.png" width="100%" /> </div>

`cat /proc/self/limits`来查看当前系统设定进程的资源限制。

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220810114812.png" width="100%" /> </div>

Linux系统启动的时候会设定好当前资源限制的属性，在`include/asm-generic/resource.h`中定义进程的资源限制，在linux启动的时候通过init进程完成配置。

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220810114630.png" width="100%" /> </div>

在init_task.h中挂载该数组。

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220810115026.png" width="100%" /> </div>

### 2.3.4 进程类型

进程是由（二进制代码应用程序）、（单线程）、分配给应用程序的资源（内存、文件）。新进程是使用（fork）或者（exec）系统调用产生的。

* fork生成当前进程的副本，称为子进程，复制为两份一样的独立的进程，资源是分开的，资源都是copy的两份，不再系统上做任何关联。
* exec从一个可执行二进制加载另一个应用程序，来替代当前运行的进程。exec不是创建新的进程，首先使用fork复制一份旧的程序，然后调用exec在系统上创建一个应用程序。
* 旧版本的clone调用，原理和fork一致，可以共享父进程的一些资源，**但属于线程范畴**。

### 2.3.5 命名空间

**命名空间是解决进程之间权限访问问题的一种设计机制**。Linux的全局管理特性，比如PID、UID和系统调用uname返回系统的信息都是全局调用。因此就导致资源和重用的问题，在虚拟化中亟需解决。把全局资源通过命名空间抽象出来，划分不同的命名空间对应不同的资源分配，可实现虚拟化环境。

父命名空间生成多个子命名空间，子命名空间有映射关系到父命名空间。

#### a) 命名空间的数据结构

```c
#include <nsproxy.h>
struct nsproxy {
    atomic_t count;
    struct uts_namespace *uts_ns; //uts: 内核名称、版本、底层体系结构
	struct ipc_namespace *ipc_ns; //ipc: 进程间通信有关的信息
	struct mnt_namespace *mnt_ns; //已装在的文件系统的视图 stcuct mnt_namespace
	struct pid_namespace *pid_ns; //有关进程ID的信息 struct pid_namespace
	struct user_namespace *user_ns; // 保存用于限制每个用户资源使用的信息
	struct net *net_ns;// 包含所有网络相关的命名空间参数
};
```

创建新进程的使用使用fork可以建立一个新的命名空间，所以在fork的时候需要标记创建新的命名空间的类别。

```C
#include <sched.h> 
#define CLONE_NEWUTS 0x04000000 /* 创建新的utsname组 */ 
#define CLONE_NEWIPC 0x08000000 /* 创建新的IPC命名空间 */ 
#define CLONE_NEWUSER 0x10000000 /* 创建新的用户命名空间 */ 
#define CLONE_NEWPID 0x20000000 /* 创建新的PID命名空间 */ 
#define CLONE_NEWNET 0x40000000 /* 创建新的网络命名空间 */
```

在task_struct里面有有关于命名空间的定义：

```C
struct task_struct {
    //...
	struct nsproxy *nsproxy;
    //...
};
```

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220810115540.png)

struct  task_struct里面挂的是stcut nsproxy的指针，只要挂上不同命名空间的指针，就算是赋予不同的命名空间了。

NOTE: 命名空间的支持必须在编译的时候启动 General setup -> Namespaces support，而且必须逐一指定需要支持的命名空间。如果内核编译的时候没有指定命名空间的支持，默认的命名空间的作用则类似于不启用命名空间，所有的属性相当于全局的。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220810115601.png)

#### b) UTS命名空间和用户命名空间

```C
<kernel/nsproxy.c> 
struct nsproxy init_nsproxy = INIT_NSPROXY(init_nsproxy); 

<init_task.h> 
#define INIT_NSPROXY(nsproxy) { \ 
    .pid_ns = &init_pid_ns, \ 
    .count = ATOMIC_INIT(1), \ 
    .uts_ns = &init_uts_ns, \ 
    .mnt_ns = NULL, \ 
    INIT_NET_NS(net_ns) \ 
    INIT_IPC_NS(ipc_ns) \ 
    .user_ns = &init_user_ns, \ 
}
```

##### b.1) UTS命名空间

UTS命名空间是Linux内核Namespace（命名空间）的一个子系统，主要用来完成对容器HOSTNAME和domain的隔离，同时保存内核名称、版本、以及底层体系结构类型等信息。

```C
<utsname.h> 
struct uts_namespace { 
    struct kref kref; 
    struct new_utsname name; 
};
// 在proc中可以看到这些值
struct new_utsname { 
    char sysname[65];    //  cat /proc/sys/kernel/ostype
    char nodename[65];   
    char release[65];    //  cat /proc/sys/kernel/osrelease
    char version[65];    //  cat /proc/sys/kernel/version
    char machine[65]; 
    char domainname[65]; // cat /proc/sys/kernel/domainname
};

init/version.c 
struct uts_namespace init_uts_ns = { 
... 
    .name = { 
        .sysname = UTS_SYSNAME, 
        .nodename = UTS_NODENAME, 
        .release = UTS_RELEASE, 
        .version = UTS_VERSION,
        .machine = UTS_MACHINE, 
        .domainname = UTS_DOMAINNAME, 
	}, 
};
```

就一个名字和kref，引用计数器，可以跟踪内核中有多少地方使用了struct uts_namespace的实例。

##### b.2) 用户命名空间

用户命名空间在数据结构管理方面类似于UTS：在要求创建新的用户命名空间时，则生成当前用户命名空间的一份副本，并关联到当前进程的nsproxy实例。但用户命名空间自身的表示要稍微复杂一些：

```C
<user_namespace.h> 
    struct user_namespace { 
    struct kref kref; 
    struct hlist_head uidhash_table[UIDHASH_SZ]; 
    struct user_struct *root_user; 
};

<kernel/user_namespace.c> 
static struct user_namespace *clone_user_ns(struct user_namespace *old_ns) 
{ 
    struct user_namespace *ns; 
    struct user_struct *new_user; 
    ... 
    ns = kmalloc(sizeof(struct user_namespace), GFP_KERNEL); 
    ... 
    ns->root_user = alloc_uid(ns, 0); 
    /* 将current->user替换为新的 */ 
    new_user = alloc_uid(ns, current->uid); 
    switch_uid(new_user); 
    return ns; 
}
```



### 2.3.6 进程ID号
我们耳熟能详的PID（process id）在其命名空间中唯一的标识号码，我们也可以通过PID找到一个进程，对进程进行操作。在进程的领域，并不是只有PID，也有其他很多的ID类型。这些概念延伸至进程组[^1]。

Linux对于PID是要进行管理的，而管理手段对PID进行数据结构的表述，这些数据结构一只脚踏入task_struct中，还有一只脚在命名空间映射，而且对于繁琐的PID的数据结构查找，Linux内核也提供了若干辅助函数用于扫描PID的数据结构。

PID的分配，首要保证的是PID在命名空间上的唯一性，内核提供了这样的方法，`alloc_pidmap`和`free_pidmap`这样的方法可以分配和释放PID。

#### a) 进程组的TGID
>**（1）进程组**
> 
> 也称之为作业，BSD与1980年前后向UNIX中增加的一个新特性，代表一个或多个进程的集合。每个进程都属于一个进程组，在waitpid函数和kill函数的参数中都曾经使用到，操作系统设计的进程组的概念，是为了简化对多个进程的管理。
>  
> **当父进程创建子进程的时候，默认子进程与父进程属于同一个进程组**，进程组ID等于进程组第一个进程ID(组长进程)。所以，组长进程标识：其进程组ID等于其进程ID.
> 
> 组长进程可以创建一个进程组，创建该进程组的进程，然后终止，只要进程组中有一个进程存在，进程组就存在，与组长进程是否终止无关。
> 
>**（2）kill发送给进程组**
> 
> 使用 kill -n -pgid 可以将信号 n 发送到进程组 pgid 中的所有进程。例如命令 kill -9 -4115 表示杀死进程组 4115 中的所有进程。

每个进程除了PID之外还有TGID（进程组的ID），组长(group leader)的PID = TGID。

```C
<sched.h>
struct task_struct {
    ...
    pid_t pid;
    pid_t tgid;
    ...
}
```

在userspace提供`setpgid`，`getpgrp`，`getpgid`的系统调用。

```C
pid_t getpgrp(void); /*获取当前进程的进程组ID*/
pid_t getpgid(pid_t pid); /*改变进程默认所属的进程组，通常可用来加入一个现有的进程组或新进程组。*/
int setpgid(pid_t pid, pid_t pgid); /* 改变进程默认所属的进程组，通常可用来加入一个现有的进程组或新进程组。*/
```

这部分可以做一个实验，https://github.com/carloscn/clab/blob/master/linux/test_pid/test_process_grp.c

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220810131250.png)

#### b) 管理PID
除了`pid`和`tgid`还需要其他成员来管理PID，如下结构体：
```C
<pid.h>
struct pid_namespace {
    ...
    struct task_struct *child_reaper;
    ...
    int level;
    struct pid_namespace *parent;
};

struct upid {
    int nr;
    struct pid_namespace *ns;
    struct hlist_node pid_chain;
};

struct pid {
    atomic_t count;
    /* lists of tasks that use this pid */
    struct hlist_head tasks[PIDTYPE_MAX];
    int level;
    struct upid numbers[1];
};

<pid.h>
enum pid_type
{
    PIDTYPE_PID,
    PIDTYPE_PGID,
    PIDTYPE_SID,
    PIDTYPE_MAX
};
```

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220810131944.png)

#### c) 管理PID函数
```C
//kernel/pid.c
pid_t task_pid_nr_ns(struct task_struct *tsk, struct pid_namespace *ns)
pid_t task_tgid_nr_ns(struct task_struct *tsk, struct pid_namespace *ns)
pid_t task_pgrp_nr_ns(struct task_struct *tsk, struct pid_namespace *ns)
pid_t task_session_nr_ns(struct task_struct *tsk, struct pid_namespace *ns)
```
生成唯一的PID：
```C
//kernel/pid.c
static int alloc_pidmap(struct pid_namespace *pid_ns)
static fastcall void free_pidmap(struct pid_namespace *pid_ns, int pid)
```

#### d) 进程关系
除了ID链接之外，内核还负责管理建立在UNIX进程创建模型之上的“家族关系”。术语如下：
* 如果是进程A分支形成了B，那么A成为B的父进程，而B成为A的子进程。
* 如果A分支形成了B1，B2，B3，...，Bn，这些Bx称为兄弟关系。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220810132536.png)

```C
// <sched.h>
struct task_struct {
	...
	struct list_head children; /* list of my children */
	struct list_head sibling; /* linkage in my parent’s children list */
	...
}
```

* children是链表表头，该链表保存所有进程的子进程；
* sibling用于将兄弟进程彼此联系起来。

`task_struct`之间的互相联系可以表现如图所示[^2]：

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220811092135.png" width="70%" /> </div>

# 3. 进程管理相关的系统调用

在这部分，我们来讨论`fork`和`exec`的系统调用实现。这部分其实在[# 13_ARMv8_内存管理（一）-内存管理要素](https://github.com/carloscn/blog/issues/53) 有提及，但是是从内存管理的角度来看（内存分页技术在fork上面提供了写时复制功能，为其效率提升做了很大的贡献）。在这一节我们来了解一下`fork`如何实现的。

同时，我们把`fork`的系统调用实现，归纳到[# 0x21_LinuxKernel_内核活动（一）之系统调用](https://github.com/carloscn/blog/issues/69) 的“可用系统调用”章节中去。

## 3.1 进程复制

在Linux系统中，不仅仅只有一个`fork`系统调用，还有其他系统调用，例如`vfork`，`clone`。`fork`进程依赖于写时复制技术的实现。`fork`、`vfork`、`clone`调用的内核函数分别是`sys_fork`，`sys_vfork`，`sys_clone`函数。三个函数最终也都调用了`do_fork`函数，这个函数内部大多数工作都是由`copy_process`复制进程的内核函数完成的。

进程内部至少包含一个线程，因此进程复制中比较重要的处理过程就是对于线程的复制。每个线程都有自己独立的栈空间，因此这部分要很谨慎小心的处理。

进程复制涉及方方面面，比如对于进程一些共享信息的处理（内存）,`shmat`这样的系统调用就强调了在进程复制之前，复制后的行为[^4]。

### 3.1.1 写时复制
关于进程的系统调用分为：
* `fork()`：属于重量级调用，因为其建立了一个完整的进程副本，然后作为子进程去执行。为了减少工作量，linux内核使用了写时复制的的技术。由于写时复制技术的实现，vfork速度方面不再有优势，应避免使用vfork。
* `vfork()`：类似于`fork()`，但是不会创建一个进程副本，父进程和子进程之间共享一份数据。好处就是节约了大量的CPU，而坏处是，父进程或者子进程之间任意一个成员修改数据都能影响到对方。`vfork`的设计用于子进程形成之后立即执行`execve`系统调用[^3]加载新程序的情形。在子进程退出或者加载新程序之前，**内核保证父进程处于阻塞状态**。
* `clone()`：产生线程，可以对父子进程之间的共享、复制进行精准控制。

写时复制(Copy-on-write, COW)技术，防止了父进程被复制之后建立真的数据，避免了使用大量的内存，复制时操作耗费更多的时间。如果复制之后执行exec立即加载程序，那么负面效应更加严重，甚至复制都是多余的，因为exec会把新数据的内存替换到当前进程的内存位置，然后运行。

写时复制技术在底层实现的是在fork的时候，**只复制父进程的页表，而不是复制真正的数据**，而且父子进程不允许修改彼此的物理页（共享内存的物理页除外）， 这种实现很简单，通过标记页表为只读属性，无论父子进程在尝试访问内存的时候，由于访问了只读属性的页面，处理器会向内核报告访问段错误，接着在段错误的处理函数中开始真正的复制数据。

COW机制使得内核可能尽可能的延迟内存的复制，更重要的是，实际上在很多情况下不需要复制，这样节约了大量的时间。

### 3.1.2 执行系统调用
#### do_fork
 **do_fork**：https://elixir.bootlin.com/linux/v3.19.8/source/kernel/fork.c#L1626
 ```C
//kernel/fork.c
long do_fork(unsigned long clone_flags, /* 控制复制过程 */
             unsigned long stack_start, /* 用户状态下的起始地址 */
             struct pt_regs *regs,      /* 指向寄存器集合的指针 */
             unsigned long stack_size,  /* 用户栈大小，不必要，设为0 */
             int __user *parent_tidptr, /* 用户空间中地址的两个指针\ */
             int __user *child_tidptr)  /* 分别指向父子进程的PID */
```

NPTL(Native Poxsix Threads Library)库线程需要实现这个两个参数。[^5]

| 参数          | 描述                                                         |
| :------------ | :----------------------------------------------------------- |
| clone_flags   | 与clone()参数flags相同, 用来控制进程复制过的一些属性信息, 描述你需要从父进程继承那些资源。该标志位的4个字节分为两部分。最低的一个字节为子进程结束时发送给父进程的信号代码，通常为SIGCHLD；剩余的三个字节则是各种clone标志的组合（本文所涉及的标志含义详见下表），也就是若干个标志之间的或运算。通过clone标志可以有选择的对父进程的资源进行复制； |
| stack_start   | 与clone()参数stack_start相同, **子进程用户态堆栈的地址**     |
| regs          | 是一个**指向了寄存器集合的指针**, 其中以原始形式, 保存了调用的参数, 该参数使用的数据类型是特定体系结构的struct pt_regs，其中按照系统调用执行时寄存器在内核栈上的存储顺序, 保存了所有的寄存器, 即指向内核态堆栈通用寄存器值的指针，通用寄存器的值是在从用户态切换到内核态时被保存到内核态堆栈中的(指向pt_regs结构体的指针。当系统发生系统调用，即用户进程从用户态切换到内核态时，该结构体保存通用寄存器中的值，并被存放于内核态的堆栈中) |
| stack_size    | **用户状态下栈的大小**, 该参数通常是不必要的, 总被设置为0    |
| parent_tidptr | 与clone的ptid参数相同, 父进程在用户态下pid的地址，该参数在CLONE_PARENT_SETTID标志被设定时有意义 |
| child_tidptr  | 与clone的ctid参数相同, 子进程在用户太下pid的地址，该参数在CLONE_CHILD_SETTID标志被设定时有意义 |

linux-4.2之后选择引入一个新的`CONFIG_HAVE_COPY_THREAD_TLS`，和一个新的`COPY_THREAD_TLS`接受TLS参数为额外的长整型（系统调用参数大小）的争论。改变sys_clone的TLS参数unsigned long，并传递到copy_thread_tls。新版本的系统中clone的TLS设置标识会通过TLS参数传递, 因此_do_fork替代了老版本的do_fork。
```C
#ifndef CONFIG_HAVE_COPY_THREAD_TLS
/* For compatibility with architectures that call do_fork directly rather than
 * using the syscall entry points below. */
long do_fork(unsigned long clone_flags,
              unsigned long stack_start,
              unsigned long stack_size,
              int __user *parent_tidptr,
              int __user *child_tidptr)
{
        return _do_fork(clone_flags, stack_start, stack_size,
                        parent_tidptr, child_tidptr, 0);
}
#endif
```

#### sys_fork
从设计层次来看，`sys_fork`是架构级定义（在arch/xxx/kernel目录下），调用linux/kernel下实现的do_fork实现。

早期内核2.4.31版本都在自己的架构目录上实现：

| 架构   | 实现                                                         |
| ------ | ------------------------------------------------------------ |
| arm    | [arch/arm/kernel/sys_arm.c, line 239](https://elixir.bootlin.com/linux/2.4.31/source/arch/arm/kernel/sys_arm.c#L239) |
| i386   | [arch/i386/kernel/process.c, line 710](https://elixir.bootlin.com/linux/2.4.31/source/arch/i386/kernel/process.c#L710) |
| x86_64 | [arch/x86_64/kernel/process.c, line 703](https://elixir.bootlin.com/linux/2.4.31/source/arch/x86_64/kernel/process.c#L703) |

```C
asmlinkage long sys_fork(struct pt_regs regs)
{
    return do_fork(SIGCHLD, regs.rsp, &regs, 0);
}
```

新版本例如4.1.15版本的内核把`sys_fork`已经去掉，换成：
```C
#ifdef __ARCH_WANT_SYS_FORK
SYSCALL_DEFINE0(fork)
{
#ifdef CONFIG_MMU
        return _do_fork(SIGCHLD, 0, 0, NULL, NULL, 0);
#else
        /* can not support in nommu mode */
        return -EINVAL;
#endif
}
#endif
```

我们可以看到唯一使用的标志是SIGCHLD。这意味着在子进程终止后将发送信号SIGCHLD信号通知父进程。由于写时复制(COW)技术，最初父子进程的栈地址相同，但是如果操作栈地址闭并写入数据，则COW机制会为每个进程分别创建一个新的栈副本。如果do_fork成功，则新建进程的pid作为系统调用的结果返回，否则返回错误码。

#### sys_vfork
早期内核2.4.31版本都在自己的架构目录上实现：

| 架构   | 实现                                                         |
| ------ | ------------------------------------------------------------ |
| arm    | [arch/arm/kernel/sys_arm.c, line 254](https://elixir.bootlin.com/linux/2.4.31/source/arch/arm/kernel/sys_arm.c#L254) |
| i386   | [arch/i386/kernel/process.c, line 737](https://elixir.bootlin.com/linux/2.4.31/source/arch/i386/kernel/process.c#L737) |
| x86_64 | [arch/x86_64/kernel/process.c, line 725](https://elixir.bootlin.com/linux/2.4.31/source/arch/x86_64/kernel/process.c#L725) |

```C
asmlinkage long sys_vfork(struct pt_regs regs)
{
    return do_fork(CLONE_VFORK | CLONE_VM | SIGCHLD, regs.rsp, &regs, 0);
}
```

同样，新版本例如4.1.15版本的内核把`sys_vfork`已经去掉，换成：
```C
#ifdef __ARCH_WANT_SYS_VFORK
SYSCALL_DEFINE0(vfork)
{
        return _do_fork(CLONE_VFORK | CLONE_VM | SIGCHLD, 0,
                        0, NULL, NULL, 0);
}
#endif
```
可以看到`sys_vfork`的实现与`sys_fork`只是略微不同, 前者使用了额外的标志`CLONE_VFORK | CLONE_VM`。

#### sys_clone
早期内核2.4.31版本都在自己的架构目录上实现：

| 架构   | 实现                                                         |
| ------ | ------------------------------------------------------------ |
| arm    | [arch/arm/kernel/sys_arm.c, line 247](https://elixir.bootlin.com/linux/2.4.31/source/arch/arm/kernel/sys_arm.c#L247) |
| i386   | [arch/i386/kernel/process.c, line 715](https://elixir.bootlin.com/linux/2.4.31/source/arch/i386/kernel/process.c#L715) |
| x86_64 | [arch/x86_64/kernel/process.c, line 708](https://elixir.bootlin.com/linux/2.4.31/source/arch/x86_64/kernel/process.c#L708) |

```C
asmlinkage int sys_clone(struct pt_regs regs)
{
    /* 注释中是i385下增加的代码, 其他体系结构无此定义
    unsigned long clone_flags;
    unsigned long newsp;

    clone_flags = regs.ebx;
    newsp = regs.ecx;*/
    if (!newsp)
        newsp = regs.esp;
    return do_fork(clone_flags, newsp, &regs, 0);
}
```

同样，新版本例如4.1.15版本的内核把`sys_clone`已经去掉，换成：
```C
#ifdef __ARCH_WANT_SYS_CLONE
#ifdef CONFIG_CLONE_BACKWARDS
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
                 int __user *, parent_tidptr,
                 unsigned long, tls,
                 int __user *, child_tidptr)
#elif defined(CONFIG_CLONE_BACKWARDS2)
SYSCALL_DEFINE5(clone, unsigned long, newsp, unsigned long, clone_flags,
                 int __user *, parent_tidptr,
                 int __user *, child_tidptr,
                 unsigned long, tls)
#elif defined(CONFIG_CLONE_BACKWARDS3)
SYSCALL_DEFINE6(clone, unsigned long, clone_flags, unsigned long, newsp,
                int, stack_size,
                int __user *, parent_tidptr,
                int __user *, child_tidptr,
                unsigned long, tls)
#else
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
                 int __user *, parent_tidptr,
                 int __user *, child_tidptr,
                 unsigned long, tls)
#endif
{
        return _do_fork(clone_flags, newsp, 0, parent_tidptr, child_tidptr, tls);
}
#endif
```

我们可以看到sys_clone的标识不再是硬编码的，而是通过各个寄存器参数传递到系统调用，因而我们需要提取这些参数。其次，**clone也不再复制进程的栈，而是可以指定新的栈地址，在生成线程时，可能需要这样做，线程可能与父进程共享地址空间， 但是线程自身的栈可能在另外一个地址空间。** 另外，还指令了用户空间的两个指针(parent_tidptr和child_tidptr)， 用于与线程库通信[^5]。

### 3.1.3 进程的生命周期
下图是一个进程的生命周期和相应的探针点：

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220811124814.png" width="70%" /> </div>

Unix 进程生成分为两个阶段[^2]：
-   父进程调用 `fork()` 系统调用。kernel 创建一个父进程的副本， 包括地址空间(在 `copy-on-write` 模式下)，打开的文件，分配一个新的 PID。 如果 `fork()` 调用成功，这个将返回在两个进程的上下文中，这个有同一个指令指针(PC 指针是一样的) 在子进程中随后的代码通常用来关闭文件，重置信号等。
-   子进程调用 `execve()` 系统调用，这个将使用新的 based 传递给 execve() 来替换掉进程的地址空间。

当调用 `exit()` 系统调用，子进程将结束。 但是，进程也可以会被 `killed`，当 kernel 出现不正确的条件(引发 kernel oops) 或者机器错误。 如果父进程像等待子进程结束，这个可以调用 `wait()` 系统调用(或者 `waitid()`), `wait()` 调用将收到进程的退出码，随后关联的 `task_struct` 将被销毁。 如果父进程不像等待子进程，子进程退出后，这个将被作为僵尸进程。 父进程可能会收到 kernel 发送的 `SIGCHLD` 信号。

### 3.1.4 do_fork的实现

#### do_fork overview
`do_fork`无论是最新版还是老的linux版本，都是内核级的实现，这部分给架构级的sys_fork函数调用。从这里可以看出，这部分已经不是和平台相关的代码了，纯属内核内部的软件逻辑。`kernel/fork.c`

<div align='left'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220811130423.png" width="80%" /> </div>

* `do_fork`以调用`copy_process`开始，后者执行新进程的实际工作（收尾工作）。我们暂时不去理会`copy_process`内部做了什么。
* 确定PID，这部分涉及两种逻辑。有无创建新的命名空间，fork如果创建了新的命名空间，则调用`pid_nr_ns`；如果没有创建命名空间只在局部PID获取即可`task_pid_vnr`。
* 如果进程使用ptrace监控新的进程，创建进程之后还要发送SIGSTOP信号，以便调试器检查其数据。
* 关于调度方面，子进程使用`wake_up_new_task`唤醒。
* 在fork进程的时候为了防止父进程修改数据，父进程需要阻塞，通过完成机制丰富设计。

#### copy_process
`copy_process`是`do_fork`中核心函数，任务是完成父进程的复制功能，这里面必须包含三个系统调用的请求处理`fork\vfork\clone`。

<div align='left'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220811130322.png" width="80%" /> </div>

这个复制过程比较复杂，我们大部分过程略过，具体解析参考《深入Linux内核架构》P55-P61，原版参考P66-P75。

#### thread问题

父进程的PCB实例只有一个成员不同：新进程分配了一个新的内核态栈，`task_struct->stack`。通常栈和`thread_info`保存在一个联合体中，`thread_info`保存了线程所需要的所有特定处理器的底层信息。
```C
<sched.h>
union thread_union {
	struct thread_info thread_info;
	unsigned long stack[THREAD_SIZE/sizeof(long)];
};

// <asm-arch/thread_info.h>
struct thread_info {
	struct task_struct *task; /* main task structure */
	struct exec_domain *exec_domain; /* execution domain */
	unsigned long flags; /* low level flags */
	unsigned long status; /* thread-synchronous flags */
	__u32 cpu; /* current CPU */
	int preempt_count; /* 0 => preemptable, <0 => BUG */
	mm_segment_t addr_limit; /* thread address space */
	struct restart_block restart_block;
}
```
关系可以看：

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220811131513.png" width="80%" /> </div>

在内核的某个特定组件使用了过多的栈空间的时候，内核栈就会溢出到thread_info上，这可能会出现严重的故障。在这种情况下，调用栈回溯的时候就会导致错误的信息出现，因此内核提供了`kstack_end`函数，用于判断给出的地址是否位于栈的有效部分。

Linux 並沒有特定的data structure來標示thread or process，thread與process都使用process的PCB。

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220811131844.png" width="100%" /> </div>


## 3.2 内核线程
在linux系统中， 我们接触最多的莫过于用户空间的任务，像用户线程或用户进程，因为他们太活跃了，也太耀眼了以至于我们感受不到内核线程的存在，但是内核线程却在背后默默地付出着，如**内存回收，脏页回写，处理大量的软中断等**，如果没有内核线程那么linux世界是那么的可怕！[^7]在进入我们真正的主题之前，我们需要知道一下事实：
* 内核线程永远运行于内核态绝不会跑到用户态去执行。
* 由于内核线程运行于内核态，所有它的权限很高，请注意这里说的是权限很高并不意味着它的优先级高，所有他可以直接做到操作页表，维护cache, 读写系统寄存器等操作。
* 内核线性是没有地址空间的概念，准确的来说是没有用户地址空间的概念，使用的是所有进程共享的内核地址空间，但是调度的时候会借用前一个进程的地址空间。
* 内核线程并没有什么特别神秘的地方，他和普通的用户任务一样参与系统调度，也可以被迁移到任何cpu上运行。
* 每个cpu都有自己的idle进程，实质上也是内核线程，但是他们比较特殊，一来是被静态创建，二来他们的优先级最低，cpu上没有其他进程运行的时候idle进程才运行。
* 除了初始化阶段0号内核线程和kthreadd本身，其他所有的内核线程都是被kthreadd内核线程来间接创建。

我们知道linux所有任务的祖先是0号进程，然后0号进程创建了天字第一号的1号init进程，init进程是所有用户任务的祖先，而**内核线程同样也有自己的祖先那就是kthreadd内核线程他的pid是2**，我们通过top命令可以观察到：红色方框都是父进程为2号进程的内核线程，绿色方框为kthreadd，他的父进程为0号进程。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220811132618.png)

这里面有一个知识点**惰性TLB(lazy TLB)**，把task_struct里面的mm_struct设定为空指针将成为惰性TLB进程。假设内核线程之后运行的进程与之前是同一个，在这种情况下，内核并不需要修改用户空间的地址表，地址表转换后备缓冲器（TLB）中的信息仍然有效。只有在内核线程执行的进程是与此前不同的用户层才需要切换，清除TLB数据。

创建内核线程的辅助方法是，`kthread_create`：

```C
kernel/kthread.c
struct task_struct *kthread_create(int (*threadfn)(void *data),
				   void *data, const char namefmt[], ...)
}
```
该函数创建一个新的内核线程，命名为namefmt。最初线程是挺值得，需要使用`wake_up_process`启动它。词汇会调用`kthreadfn`给出线程函数。

另一个备选方案是通过`kthread_run`宏来代替。

使用`ps fax`可以输出方括号的进程为内核线程所属进程，与普通进程区分。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220811133632.png)
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220811133646.png)

## 3.3 启动新程序
Linux提供`execve`系统调用启动新的程序，通过新代码替换现存程序。

### 3.3.1 execve实现

该系统调用的入口节点是`sys_execve`函数，早期内核2.4.31版本都在自己的架构目录上实现：

| 架构   | 实现                                                         |
| ------ | ------------------------------------------------------------ |
| arm    | [arch/arm/kernel/sys_arm.c, line 262](https://elixir.bootlin.com/linux/2.4.31/source/arch/arm/kernel/sys_arm.c#L262) |
| i386   | [arch/i386/kernel/process.c, line 475](https://elixir.bootlin.com/linux/2.4.31/source/arch/i386/kernel/process.c#L475) |
| x86_64 | [arch/x86_64/kernel/process.c, line 678](https://elixir.bootlin.com/linux/2.4.31/source/arch/x86_64/kernel/process.c#L678) |

```C
asmlinkage int sys_execve(char *filenamei, char **argv, char **envp, struct pt_regs *regs)
{
	int error;
	char * filename;

	filename = getname(filenamei);
	error = PTR_ERR(filename);
	if (IS_ERR(filename))
		goto out;
	error = do_execve(filename, argv, envp, regs);
	putname(filename);
out:
	return error;
}
```

<div align='left'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220811143053.png" width="66%" /> </div>

* 首先要打开可执行文件，内核找到相关的inode生成一个文件描述符；
* `bprm_init`处理几个管理型的任务，包括`mm_struct`初始化，`mm_init`用于栈创建；
* `prepare_binprm`：用于提供一些父进程相关的值UID和GID；
* 剩下是处理参数的列表，环境文件名等。Linux支持执行文件的各种不同组织，标准格式是ELF。
* `search_binary_handler`用于`do_execve`结束之后查找一个适当的二进制格式，用于执行特定的文件。
	* 释放原始进程的所有资源。
	* 将应用程序映射到虚拟地址空间中。必须考虑下列段的处理：
		* text段包含程序的可执行代码。`start_code`和`end_code`指定该段在地址空间驻留区域。
		* 预先初始化数据（在编译阶段指定了具体值的变量）位于`start_data`和`end_data`之间。
		* 堆空间用于动态内存分配。`start_brk`和`brk`指定边界。
		* 栈位置`start_stack`定义。几乎所有的寄存机栈都是自动向下增长的。唯一的例外是PA-RISC。对于栈反向增长，体系结构相关部分的实现必须告知内核，可以通过设定`STACK_GROWSUP`完成。
		* 程序的环境变量和参数也需要映射到虚拟空间。`arg_start`和`arg_end`之间，还有`env_end`和`env_start`、

除了ELF格式，还有几种Linux支持的二进制格式，这里列举作为参考：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220811144158.png)

### 3.3.2 解释二进制
在linux内核中，每种二进制都表示下列数据结构的实例：

```C
<binfmts.h>
struct linux_binfmt {
    struct linux_binfmt * next;
    struct module *module;
    int (*load_binary)(struct linux_binprm *, struct pt_regs * regs);
    int (*load_shlib)(struct file *);
    int (*core_dump)(long signr, struct pt_regs * regs, struct file * file);
    unsigned long min_coredump; /* minimal dump size */
};
```

* `load_binary`：用于加载普通程序
* `load_shlib`：用于加载共享库
* `core_dump`：用于在程序错的情况下输出内存转储。内存转储随后可以使用调试器，例如gdb分析，以便解决问题。
* `min_coredump`是生成内存转储时，内存转储文件长度的下界。

注意每种二进制格式首先必须使用`resgiser_binfmt`像内核注册。该函数的目的是向一个链表增加一种新的二进制格式。

### 3.3.3 退出进程

进程必须`exit`系统调用终止。这使得内核有机会将该进程的资源释放回系统。该调用入口点是`sys_exit`函数，需要一个错误码作为其参数。很快退出进程调度委托给`do_exit`。简言之，该函数的实现就是将各个引用计数器-1。

# 4. 参考文献

[^1]:[【进程】_进程组__月雲之霄的博客-CSDN博客__进程组_ ](https://www.baidu.com/link?url=Jg_cyO9HsLv7sAJ5ApHvoZvdAojoguMbuUpSrDFiwZp4WR8GEk8DqQa8OrgNaGc7QBY9GFZaRp74OT5A4LA01K&wd=&eqid=e3d490970001a91b0000000662f3314a)
[^2]:[Process management](https://myaut.github.io/dtrace-stap-book/kernel/proc.html)
[^3]:[execve](https://baike.baidu.com/item/execve/4475693?fr=aladdin)
[^4]:[Linux进程之间的通信-内存共享（System V）](https://github.com/carloscn/blog/issues/16)
[^5]:[Linux下进程的创建过程分析(_do_fork/do_fork详解)--Linux进程的管理与调度（八）](https://blog.csdn.net/gatieme/article/details/51569932)
[^6]:[# 奔跑吧 CH 3.1 進程的誕生](https://hackmd.io/@PIFOPlfSS3W_CehLxS3hBQ/S14tx4MqP)
[^7]:[深入理解Linux内核之内核线程（上）](https://www.eet-china.com/mp/a47856.html)
[^8]:[]()
[^9]:[]()
[^10]:[]()
[^11]:[]()
[^12]:[]()
[^13]:[]()
[^14]:[]()
[^15]:[]()
[^16]:[]()
[^17]:[]()
[^18]:[]()
[^19]:[]()
[^20]:[]()