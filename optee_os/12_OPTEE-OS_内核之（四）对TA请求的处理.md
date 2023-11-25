# 12_OPTEE-OS_内核之（四）对TA请求的处理
当在REE侧执行CA时，OP-TEE中的tee_entry_std函数会根据CA调用libteec库文件中不同的接口而采取不同的处理方式，这些操作包括打开会话、关闭会话、调用TA中的命令、取消对TA中命令的调用。**OP-TEE中存在动态和静态两种TA**，本章将详细介绍OP-TEE对这两种TA操作的具体实现。

# 1. session
会话是CA调用TA中具体命令的基础，如果CA与TA之间没有建立会话，CA就无法调用TA中的任何命令。在CA中通过调用libteec库文件中的**TEEC_OpenSession**函数来建议CA与特定TA之间的会话，该函数执行时会调用OP-TEE驱动中的**optee_open_session**函数发送标准安全监控模式调用（std smc）请求，通知OP-TEE开始执行创建会话的操作。该标准安全监控模式调用会被Monitor模式（ARMv7）或者EL3阶段（ARMv8）处理后转发给OP-TEE，OP-TEE调用**entry_open_session**函数来完成创建会话的操作。在OP-TEE中一次完整的创建会话操作的流程如图所示。

综上所述，**REE(TEEC)-> Driver(open_session) -> OPTEE(entry_open_session) ->ARM(switch to EL3)**

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221006150152.png)

OP-TEE支持**动态TA和静态TA**。
* 静态TA镜像将与OP-TEE镜像编译在同一个镜像文件中，因此静态TA镜像会存放在OP-TEE镜像的特定区段中，静态TA在OP-TEE启动时会被加载到属性为`MEM_AREA_TA_RAM`的安全内存中。
* 动态TA则是将TA镜像文件保存到文件系统中，**在创建会话时OP-TEE通过发送RPC请求将动态TA镜像加载到OP-TEE的安全内存中**。

创建会话在OP-TEE中的操作是根据TA对应的UUID值找到对应的TA镜像，读取TA镜像文件的头部数据，并将相关数据保存到`tee_ta_ctx`结构体变量中，然后将填充好的`tee_ta_ctx`结构体变量保存到`tee_ctxes`链表中，以便后期CA执行调用TA命令操作时可通过查找`tee_ctxes`链表来获取对应的会话。最后根据会话的内容进入到指定的TA，根据需要被调用的命令的ID执行特定的操作。

## 1.1 静态TA的创建session过程
静态TA是与OP-TEE OS镜像编译在一起的，在OP-TEE的启动阶段，静态TA镜像的内容会被加载到OP-TEE的安全内存中，且在启动过程中会调用verify_pseudo_tas_conformance函数对所有的静态TA镜像的内容进行检查。

调用创建会话操作后，OP-TEE首先会在已经被创建的会话链表中查找是否有匹配的会话存在。如果找到则将该会话的ID直接返回给REE侧，如果没有找到则会根据UUID去静态TA的段中进行查找，然后将找到的静态TA的相关信息填充到tee_ta_ctx结构体变量中，再将该变量添加到全局的tee_ctxes链表中，并调用静态TA的enter_open_session函数执行创建会话操作。静态TA创建会话的操作全过程如图13-2所示。

静态查找TA的优先级： 已创建的TA session -> 静态TA段。

![](https://raw.githubusercontent.com/carloscn/images/main/typoradfdsfjkdsfhjishvnbgsdf.svg)

### 1.1.1 根据UUID找到对应静态TA
如果CA调用的是静态TA，OP-TEE会到存放静态TA的ta_head区域通过遍历的方式，对比内存中静态TA区域中TA的UUID与需要调用的TA的UUID值是否相等找到需要被调用的静态TA，这些操作是通过调用tee_ta_init_pseudo_ta_session函数来实现的。查找到需要被调用的静态TA后，该函数会将pseudo_ta_ops变量的地址赋值到TA的上下文中，该函数的内容如下：
```C

/*-----------------------------------------------------------------------------
 * Initialises a session based on the UUID or ptr to the ta
 * Returns ptr to the session (ta_session) and a TEE_Result
 *---------------------------------------------------------------------------*/
TEE_Result tee_ta_init_pseudo_ta_session(const TEE_UUID *uuid,
			struct tee_ta_session *s)
{
	struct pseudo_ta_ctx *stc = NULL;
	struct tee_ta_ctx *ctx;
	const struct pseudo_ta_head *ta;

	DMSG("Lookup pseudo TA %pUl", (void *)uuid);

	ta = SCATTERED_ARRAY_BEGIN(pseudo_tas, struct pseudo_ta_head);
	while (true) {
		if (ta >= SCATTERED_ARRAY_END(pseudo_tas,
					      struct pseudo_ta_head))
			return TEE_ERROR_ITEM_NOT_FOUND;
		if (memcmp(&ta->uuid, uuid, sizeof(TEE_UUID)) == 0)
			break;
		ta++;
	}

	/* Load a new TA and create a session */
	DMSG("Open %s", ta->name);
	stc = calloc(1, sizeof(struct pseudo_ta_ctx));
	if (!stc)
		return TEE_ERROR_OUT_OF_MEMORY;
	ctx = &stc->ctx;

	ctx->ref_count = 1;
	ctx->flags = ta->flags;
	stc->pseudo_ta = ta;
	ctx->ts_ctx.uuid = ta->uuid;
	ctx->ts_ctx.ops = &pseudo_ta_ops;

	mutex_lock(&tee_ta_mutex);
	s->ts_sess.ctx = &ctx->ts_ctx;
	TAILQ_INSERT_TAIL(&tee_ctxes, ctx, link);
	mutex_unlock(&tee_ta_mutex);

	DMSG("%s : %pUl", stc->pseudo_ta->name, (void *)&ctx->ts_ctx.uuid);

	return TEE_SUCCESS;
}
```
`__start_ta_head_section`到`__stop_ta_head_section`区域之间保存的是所有静态TA的ta_head数据。在编译各静态TA时，通过使用pseudo_ta_register宏将各静态TA的ta_head数据保存到ta_head_section段中，该段的起始地址是`__start_ta_head_section`，结束地址是`__stop_ta_head_section`。

### 1.1.2 pseudo_ta_ops变量
OP-TEE对所有静态TA的操作接口都保存在pseudo_ta_ops变量中，该变量的enter_open_session成员的值为pseudo_ta_enter_open_session，该函数指针会**检查具体TA中的相关函数是否有效**并执行相应的操作，该函数的内容。

### 1.1.3 pseudo_ta_register宏
在编译过程中，该宏将静态TA的ta_head数据保存到ta_head_section段中，该宏的定义如下：
```C
#define pseudo_ta_register(...) static const struct pseudo_ta_head __hea            __used __section("ta_head_section") = { __VA_ARGS__ }
```

一个静态TA的ta_head数据中保存了该静态TA的UUID、name、flags，以及初始化该TA操作接口的函数指针。以提供网络socket服务的静态TA为例。在该示例中定义了网络socket服务静态TA提供的创建会话、关闭会话、调用命令的操作实现。

## 1.2 动态TA的创建session过程
动态的TA镜像存放在REE侧的文件系统中。CA在执行动态TA的创建会话操作时，OP-TEE会根据UUID值借助RPC机制让tee_supplicant将动态TA镜像加载到OP-TEE的内存中，并获取加载到内存中的动态TA的相关信息，将这些信息填充到tee_ta_ctx结构体变量中，然后再将该变量添加到全局的tee_ctxes链表中，以便后续CA端调用该TA中的命令操作时，可直接根据会话ID值从链表中找到对应的会话。加载动态TA镜像到OP-TEE中是通过调用tee_ta_init_user_ta_session函数来实现的，该函数会调用ta_load函数发送RPC请求从REE侧的文件系统中加载动态TA镜像到OP-TEE中。在将TA镜像的内容写入到OP-TEE的内存中之前，OP-TEE会对该TA镜像中的内容进行电子验签，以确保该TA镜像的合法性。

动态TA运行在OP-TEE的用户空间，创建会话操作最终会切换到用户态，调用到具体动态TA的创建会话接口函数TA_OpenSessionEntryPoint。

### 1.2.1 动态TA加载
动态TA的镜像文件被保存在REE侧的文件系统中。在第一次执行创建会话操作时首先需要将REE侧的动态TA镜像文件加载到OP-TEE中，并初始化该TA的运行上下文。这些操作是通过调用tee_ta_init_user_ta_session函数来实现的，该函数内容和注释如下：
```C
TEE_Result tee_ta_init_user_ta_session(const TEE_UUID *uuid,
				       struct tee_ta_session *s)
{
	TEE_Result res = TEE_SUCCESS;
	struct user_ta_ctx *utc = NULL;

	utc = calloc(1, sizeof(struct user_ta_ctx));
	if (!utc)
		return TEE_ERROR_OUT_OF_MEMORY;

	utc->uctx.is_initializing = true;
	TAILQ_INIT(&utc->open_sessions);
	TAILQ_INIT(&utc->cryp_states);
	TAILQ_INIT(&utc->objects);
	TAILQ_INIT(&utc->storage_enums);
	condvar_init(&utc->ta_ctx.busy_cv);
	utc->ta_ctx.ref_count = 1;

	utc->uctx.ts_ctx = &utc->ta_ctx.ts_ctx;

	/*
	 * Set context TA operation structure. It is required by generic
	 * implementation to identify userland TA versus pseudo TA contexts.
	 */
	set_ta_ctx_ops(&utc->ta_ctx);

	utc->ta_ctx.ts_ctx.uuid = *uuid;
	res = vm_info_init(&utc->uctx);
	if (res)
		goto out;

#ifdef CFG_TA_PAUTH
	crypto_rng_read(&utc->uctx.keys, sizeof(utc->uctx.keys));
#endif

	mutex_lock(&tee_ta_mutex);
	s->ts_sess.ctx = &utc->ta_ctx.ts_ctx;
	s->ts_sess.handle_svc = s->ts_sess.ctx->ops->handle_svc;
	/*
	 * Another thread trying to load this same TA may need to wait
	 * until this context is fully initialized. This is needed to
	 * handle single instance TAs.
	 */
	TAILQ_INSERT_TAIL(&tee_ctxes, &utc->ta_ctx, link);
	mutex_unlock(&tee_ta_mutex);

	/*
	 * We must not hold tee_ta_mutex while allocating page tables as
	 * that may otherwise lead to a deadlock.
	 */
	ts_push_current_session(&s->ts_sess);

	res = ldelf_load_ldelf(&utc->uctx);
	if (!res)
		res = ldelf_init_with_ldelf(&s->ts_sess, &utc->uctx);

	ts_pop_current_session();

	mutex_lock(&tee_ta_mutex);

	if (!res) {
		utc->uctx.is_initializing = false;
	} else {
		s->ts_sess.ctx = NULL;
		TAILQ_REMOVE(&tee_ctxes, &utc->ta_ctx, link);
	}

	/* The state has changed for the context, notify eventual waiters. */
	condvar_broadcast(&tee_ta_init_cv);

	mutex_unlock(&tee_ta_mutex);

out:
	if (res) {
		condvar_destroy(&utc->ta_ctx.busy_cv);
		pgt_flush_ctx(&utc->ta_ctx.ts_ctx);
		free_utc(utc);
	}

	return res;
}
```

当动态TA被加载到OP-TEE中后，OP-TEE会调用user_ta_ops中的enter_open_session成员变量所指向的函数进一步处理CA创建会话的请求。

### 1.2.2 OP-TEE内核空间对创建会话的处理
在创建CA与动态TA的会话过程中，tee_ta_open_session函数会调用ctx->ops>enter_open_session接口，其对应的是user_ta_ops变量中的enter_open_session成员所指向的函数，该成员的值指向user_ta_enter_open_session函数。user_ta_enter_open_session通过调用user_ta_enter函数来处理创建会话的操作，建立内存映射后会进入OP-TEE的用户空间去执行。

### 1.2.3 切换到用户空间的实现
调用thread_enter_user_mode函数会进入到OPTEE的用户空间。调用该函数时会指定切换到用户空间后的起始运行函数的地址。在OP-TEE中该值被设置成ta_head->entry.ptr64，entry.prt64在user_ta_header.c文件中被赋值为__utee_entry，即当切换到用户空间后，系统将会执行__utee_entry函数，那如何实现从OP-TEE的内核态切换到OP-TEE的用户态呢？OP-TEE中是通过调用__thread_enter_user_mode函数来实现的。切换到用户态后，系统将执行pc指针指向的函数，该函数会被赋值成entry.prt64的值，在用户空间调用__utee_entry继续执行。

### 1.2.4 用户空间的entry_open_session函数
当系统切换到OP-TEE的用户态后，会进入__utee_entry函数执行，该函数中会根据命令ID调用用户空间中的entry_open_session函数来执行特定TA中的创建会话接口的内容。

待执行完具体的动态TA中的TA_OpenSessionEntryPoint函数后，动态TA的创建会话操作也就完成，在CA端可使用返回的会话ID通过调用命令的接口来调用该动态TA中的具体命令去执行特定的操作。

### 1.2.5 用户空间返回内核空间

待entry_open_session执行完并返回到__utee_entry后，__utee_entry函数将会调用utee_return函数，通过系统调用的方式返回到OPTEE的内核空间。该返回操作最终会调用syscall_sys_return函数来实现。从用户空间切换回到内核空间的过程如图所示。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221006155122.png" width="60%" /></div>


thread_unwind_user_mode函数主要是将寄存器的状态恢复到切换到用户空间之前的状态，该函数的内容如下：

```C
/*
 * void thread_unwind_user_mode(uint32_t ret, uint32_t exit_status0,
 * 		uint32_t exit_status1);
 * See description in thread.h
 */
FUNC thread_unwind_user_mode , :
	/* Store the exit status */
	load_xregs sp, THREAD_USER_MODE_REC_CTX_REGS_PTR, 3, 5
	str	w1, [x4]
	str	w2, [x5]
	/* Save x19..x30 */
	store_xregs x3, THREAD_CTX_REGS_X19, 19, 30
	/* Restore x19..x30 */
	load_xregs sp, THREAD_USER_MODE_REC_X19, 19, 30
	add	sp, sp, #THREAD_USER_MODE_REC_SIZE
	/* Return from the call of thread_enter_user_mode() */
	ret
END_FUNC thread_unwind_user_mode
```

# 2. TA_InvokeCommand命令操作在OPTEE的实现

REE侧的CA执行创建会话操作成功后，CA就可使用获取到的会话ID和命令ID调用`TEEC_InvokeCommand`接口来让TA执行特定的命令。在CA中调用`TEEC_InvokeCommand`接口时，该函数会将会话ID、命令ID，以及需要传递给TA的参数信息通过ioctl的系统调用发送到OP-TEE的驱动中，OP-TEE驱动会调用`optee_invoke_func`函数将需要传递给TA的参数信息保存在共享内存中，并触发安全监控模式调用（smc）切换到Monitor模式（ARMv7）或EL3（ARMv8）中进行安全世界状态的处理。调用TA命令触发的安全监控模式调用最终会被作为标准安全监控模式调用进行解析，并建立一个专门的线程进入`thread_std_smc_entry`函数去执行，线程运行到`tee_entry_std`函数时会对安全监控模式调用（smc）进行判定，并进入调用TA命令的分支。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221006160241.png" width="80%" /></div>

根据会话ID获取到已经创建的会话内容后，OP-TEE会调用tee_ta_invoke_command函数开始对调用TA命令的操作请求进行处理，该函数的内容如下：

```C
TEE_Result tee_ta_invoke_command(TEE_ErrorOrigin *err,
				 struct tee_ta_session *sess,
				 const TEE_Identity *clnt_id,
				 uint32_t cancel_req_to, uint32_t cmd,
				 struct tee_ta_param *param)
{
	struct tee_ta_ctx *ta_ctx = NULL;
	struct ts_ctx *ts_ctx = NULL;
	TEE_Result res = TEE_SUCCESS;

	if (check_client(sess, clnt_id) != TEE_SUCCESS)
		return TEE_ERROR_BAD_PARAMETERS; /* intentional generic error */

	if (!check_params(sess, param))
		return TEE_ERROR_BAD_PARAMETERS;

	ts_ctx = sess->ts_sess.ctx;
	if (!ts_ctx) {
		/* The context has been already destroyed */
		*err = TEE_ORIGIN_TEE;
		return TEE_ERROR_TARGET_DEAD;
	}

	ta_ctx = ts_to_ta_ctx(ts_ctx);
	if (ta_ctx->panicked) {
		DMSG("Panicked !");
		destroy_ta_ctx_from_session(sess);
		*err = TEE_ORIGIN_TEE;
		return TEE_ERROR_TARGET_DEAD;
	}

	tee_ta_set_busy(ta_ctx);

	sess->param = param;
	set_invoke_timeout(sess, cancel_req_to);
	res = ts_ctx->ops->enter_invoke_cmd(&sess->ts_sess, cmd);

	sess->param = NULL;
	tee_ta_clear_busy(ta_ctx);

	if (ta_ctx->panicked) {
		destroy_ta_ctx_from_session(sess);
		*err = TEE_ORIGIN_TEE;
		return TEE_ERROR_TARGET_DEAD;
	}

	*err = sess->err_origin;

	/* Short buffer is not an effective error case */
	if (res != TEE_SUCCESS && res != TEE_ERROR_SHORT_BUFFER)
		DMSG("Error: %x of %d", res, *err);

	return res;
}
```

## 2.1 静态TA调用
静态的TA运行于OP-TEE的内核空间。TEE通过从CA传来的会话ID获取需要被调用的静态TA的上下文，然后从上下文中获取该静态TA提供的invoke_command_entry_point接口。invoke_command_entry_point对应的函数会根据不同的命令ID执行相应的操作，并将执行结果返回给CA。静态TA的调用命令操作的整个操作过程如图所示。

灰色是entry_invoke_command的通用操作：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221006164248.png)

在对静态的TA执行创建会话操作时会将该TA的运行上下文中的ctx->ops变量赋值成pseudo_ta_ops，故调用ss->ctx->ops>enter_invoke_cmd就会调用pseudo_ta_enter_invoke_cmd函数。

## 2.2 动态TA调用
动态TA运行在OP-TEE的用户空间，OP-TEE通过从CA传来的会话ID找到对应动态TA，并获取该动态TA的运行上下文，然后调用sess->ctx->ops成员中的调用命令的方法，即user_ta_enter_invoke_cmd函数。这是因为在创建CA与该动态TA的会话时，sess->ctx->ops被赋值成user_ta_ops，该变量中的enter_invoke_cmd成员指向的就是user_ta_enter_invoke_cmd函数。user_ta_enter_invoke_cmd函数会执行运行空间的切换操作，关于如何从OP-TEE的内核空间进入OPTEE的用户空间，可参阅13.1.2节“用户空间的entry_open_session函数”部分。动态TA的调用命令的整个操作过程如图所示。

当系统运行于OP-TEE的用户空间后，会调用用户空间的entry_invoke_command函数执行调用命令的操作。用户空间的entry_invoke_command函数定义在optee_os/lib/libutee/arch/arm/user_ta_entry.c文件中。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221006164939.png)

OP-TEE调用用户空间的entry_invoke_command函数时，创建的线程就已进入到TA镜像的上下文中运行了，当调用TA_InvokeCommandEntryPoint函数时就会去执行TA镜像中定义的TA_InvokeCommandEntryPoint函数，该函数具体会执行什么操作就由具体的TA决定，一般是根据命令id执行定义好的操作。

# 3. 关闭会话
CA端通过调用libteec库文件中关闭会话的接口通知OP-TEE执行关闭会话的操作。该操作的作用是让OP-TEE释放建立的会话的相关资源，并将该会话从全局的会话链表中删除。OP-TEE实现关闭会话操作的流程如图所示。

实现关闭会话的操作过程中，在将会话从全局会话队列tee_open_session链表删除之前，需要先执行TA中的关闭会话接口中的操作，这是因为TA可能在创建会话时分配了一些资源，如果在释放这些资源之前就将该会话从链表中移除，这些分配的资源将无法被释放，这样会造成内存泄漏或者其他问题。tee_ta_close_session函数是执行关闭会话的主要函数，大多数资源的释放操作都是在该函数中完成的。

## 3.1 静态TA关闭操作
关闭与静态TA的会话操作时，OP-TEE会调用ctx->ops->enter_close_session来执行具体TA的关闭会话操作。其调用的是pseudo_ta_enter_close_session函数，该函数会调用具体静态TA提供的close_session_entry_ponit和destroy_entry_point指定的接口来释放掉该TA占用的系统资源。

## 3.2 动态TA关闭操作
关闭动态TA的会话操作时，OP-TEE会调用ctx->ops->enter_close_session来执行具体TA的关闭会话操作，其调用的是user_ta_enter_close_session函数，该函数的执行过程中会切换到OP-TEE的用户空间，调用具体动态TA中的TA_CloseSessionEntryPoint函数，完成动态TA在用户空间资源的释放，其整体的调用流程类似于动态TA的创建会话操作。


# 4. 总结
本章介绍了CA端调用libteec库文件中的接口执行创建会话、关闭会话、调用命令操作时在OPTEE中的具体实现和流程。静态TA运行于OP-TEE的内核空间，动态TA运行于OP-TEE的用户空间。两种TA在OP-TEE中实现上述操作各有不同，静态TA的所有操作都在内核空间中完成，动态TA的所有操作则需要分别在内核空间和用户空间中完成。关于如何从OP-TEE的内核空间切换到OP-TEE的用户空间运行在本章中也进行了详细的介绍。






