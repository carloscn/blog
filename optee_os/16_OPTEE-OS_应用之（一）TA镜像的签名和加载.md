# 16_OPTEE-OS_应用之（一）TA镜像的签名和加载
使用OP-TEE实现特定功能需求则需要开发一个特定的TA，TA调用GP规范定义的接口实现该功能需求。TA镜像文件会被保存在REE侧的文件系统中并以动态TA的方式运行于OP-TEE中，**当用户需要调用该TA的功能时，通过在CA中调用libteec库中的接口，完成创建会话的操作，将REE侧文件系统中的TA镜像文件加载到OP-TEE的用户空间运行**。为防止该TA镜像文件被篡改或被破坏，在加载TA镜像文件的过程中会对该TA镜像文件的合法性进行检查，只有校验通过的TA镜像文件才允许运行于OP-TEE的用户空间。编译TA镜像文件过程中会对TA镜像文件做电子签名操作。本章将详细介绍TA镜像文件的编译、签名，以及加载过程。

# 1. TA文件的编译和签名
TA镜像文件在OP-TEE工程编译过程中生成，也可通过单独调用TA目录下的脚本来进行编译，但前提是OP-TEE工程被完整编译过。编译过程会先生成原始的TA镜像文件，然后使用签名脚本对该文件进行电子签名，并最终生成.ta文件，即最终会被加载到OP-TEE中的TA镜像文件。

## 1.1 TA镜像文件编译
对某个TA源代码目录中的Makefile文件执行make指令可触发编译生成TA镜像文件的操作，该Makefile文件将会包含optee_os/ta/mk/ta_dev_kit.mk文件，该文件中会定义各种目标依赖关系和Object，编译完目标和object后，编译器将会按照optee_os/ta/arch/arm/link.mk文件中的依赖关系将目标和object链接成xxx.ta文件，其中xxx是该TA UUID的值。link.mk中的链接依赖关系如下：

```makefile
link-script$(sm) = $(ta-dev-kit-dir$(sm))/src/ta.ld.S
link-script-pp$(sm) = $(link-out-dir$(sm))/ta.lds
link-script-dep$(sm) = $(link-out-dir$(sm))/.ta.ld.d

SIGN_ENC ?= $(PYTHON3) $(ta-dev-kit-dir$(sm))/scripts/sign_encrypt.py
TA_SIGN_KEY ?= $(ta-dev-kit-dir$(sm))/keys/default_ta.pem

ifeq ($(CFG_ENCRYPT_TA),y)
# Default TA encryption key is a dummy key derived from default
# hardware unique key (an array of 16 zero bytes) to demonstrate
# usage of REE-FS TAs encryption feature.
#
# Note that a user of this TA encryption feature needs to provide
# encryption key and its handling corresponding to their security
# requirements.
TA_ENC_KEY ?= 'b64d239b1f3c7d3b06506229cd8ff7c8af2bb4db2168621ac62c84948468c4f4'
endif

all: $(link-out-dir$(sm))/$(user-ta-uuid).dmp \
	$(link-out-dir$(sm))/$(user-ta-uuid).stripped.elf \
	$(link-out-dir$(sm))/$(user-ta-uuid).ta
cleanfiles += $(link-out-dir$(sm))/$(user-ta-uuid).elf
cleanfiles += $(link-out-dir$(sm))/$(user-ta-uuid).dmp
cleanfiles += $(link-out-dir$(sm))/$(user-ta-uuid).map
cleanfiles += $(link-out-dir$(sm))/$(user-ta-uuid).stripped.elf
cleanfiles += $(link-out-dir$(sm))/$(user-ta-uuid).ta
cleanfiles += $(link-script-pp$(sm)) $(link-script-dep$(sm))

link-ldflags  = -e__ta_entry -pie
link-ldflags += -T $(link-script-pp$(sm))
link-ldflags += -Map=$(link-out-dir$(sm))/$(user-ta-uuid).map
link-ldflags += --sort-section=alignment
link-ldflags += -z max-page-size=4096 # OP-TEE always uses 4K alignment
ifeq ($(sm)-$(CFG_TA_BTI),ta_arm64-y)
link-ldflags += $(call ld-option,-z force-bti) --fatal-warnings
endif
link-ldflags += --as-needed # Do not add dependency on unused shlib
link-ldflags += $(link-ldflags$(sm))

$(link-out-dir$(sm))/dyn_list:
	@$(cmd-echo-silent) '  GEN     $@'
	$(q)mkdir -p $(dir $@)
	$(q)echo "{" >$@
	$(q)echo "__elf_phdr_info;" >>$@
ifeq ($(CFG_FTRACE_SUPPORT),y)
	$(q)echo "__ftrace_info;" >>$@
endif
	$(q)echo "trace_ext_prefix;" >>$@
	$(q)echo "trace_level;" >>$@
	$(q)echo "};" >>$@
link-ldflags += --dynamic-list $(link-out-dir$(sm))/dyn_list
dynlistdep = $(link-out-dir$(sm))/dyn_list
cleanfiles += $(link-out-dir$(sm))/dyn_list

link-ldadd  = $(user-ta-ldadd) $(addprefix -L,$(libdirs))
link-ldadd += --start-group
link-ldadd += $(addprefix -l,$(libnames))
ifneq (,$(filter %.cpp,$(srcs)))
link-ldflags += --eh-frame-hdr
link-ldadd += $(libstdc++$(sm)) $(libgcc_eh$(sm))
endif
link-ldadd += --end-group

link-ldadd-after-libgcc += $(addprefix -l,$(libnames-after-libgcc))

ldargs-$(user-ta-uuid).elf := $(link-ldflags) $(objs) $(link-ldadd) \
				$(libgcc$(sm)) $(link-ldadd-after-libgcc)

link-script-cppflags-$(sm) := \
	$(filter-out $(CPPFLAGS_REMOVE) $(cppflags-remove), \
		$(nostdinc$(sm)) $(CPPFLAGS) \
		$(addprefix -I,$(incdirs$(sm)) $(link-out-dir$(sm))) \
		$(cppflags$(sm)))

-include $(link-script-dep$(sm))

link-script-pp-makefiles$(sm) = $(filter-out %.d %.cmd,$(MAKEFILE_LIST))

define gen-link-t
$(link-script-pp$(sm)): $(link-script$(sm)) $(conf-file) $(link-script-pp-makefiles$(sm))
	@$(cmd-echo-silent) '  CPP     $$@'
	$(q)mkdir -p $$(dir $$@)
	$(q)$(CPP$(sm)) -P -MT $$@ -MD -MF $(link-script-dep$(sm)) \
		$(link-script-cppflags-$(sm)) $$< -o $$@

$(link-out-dir$(sm))/$(user-ta-uuid).elf: $(objs) $(libdeps) \
					  $(libdeps-after-libgcc) \
					  $(link-script-pp$(sm)) \
					  $(dynlistdep) \
					  $(additional-link-deps)
	@$(cmd-echo-silent) '  LD      $$@'
	$(q)$(LD$(sm)) $(ldargs-$(user-ta-uuid).elf) -o $$@

$(link-out-dir$(sm))/$(user-ta-uuid).dmp: \
			$(link-out-dir$(sm))/$(user-ta-uuid).elf
	@$(cmd-echo-silent) '  OBJDUMP $$@'
	$(q)$(OBJDUMP$(sm)) -l -x -d $$< > $$@

$(link-out-dir$(sm))/$(user-ta-uuid).stripped.elf: \
			$(link-out-dir$(sm))/$(user-ta-uuid).elf
	@$(cmd-echo-silent) '  OBJCOPY $$@'
	$(q)$(OBJCOPY$(sm)) --strip-unneeded $$< $$@

cmd-echo$(user-ta-uuid) := SIGN   #
ifeq ($(CFG_ENCRYPT_TA),y)
crypt-args$(user-ta-uuid) := --enc-key $(TA_ENC_KEY)
cmd-echo$(user-ta-uuid) := SIGNENC
endif
$(link-out-dir$(sm))/$(user-ta-uuid).ta: \
			$(link-out-dir$(sm))/$(user-ta-uuid).stripped.elf \
			$(TA_SIGN_KEY) \
			$(lastword $(SIGN_ENC))
	@$(cmd-echo-silent) '  $$(cmd-echo$(user-ta-uuid)) $$@'
	$(q)$(SIGN_ENC) --key $(TA_SIGN_KEY) $$(crypt-args$(user-ta-uuid)) \
		--uuid $(user-ta-uuid) --ta-version $(user-ta-version) \
		--in $$< --out $$@
endef

$(eval $(call gen-link-t))

additional-link-deps :=

```

TA镜像文件中的调试信息。在原始TA镜像文件的头部有一个ta_head段，该段中存放该TA的基本信息以及被调用到的入口地址，该段的内容将会在加载TA镜像到OP-TEE时和调用TA执行特定命令时被使用到。存放在该段中的内容定义在optee_os/ta/arch/arm/user_ta_header.c文件中：
```C
const struct ta_head ta_head __section(".ta_head") = {
	/* UUID, unique to each TA */
	.uuid = TA_UUID,
	/*
	 * According to GP Internal API, TA_FRAMEWORK_STACK_SIZE corresponds to
	 * the stack size used by the TA code itself and does not include stack
	 * space possibly used by the Trusted Core Framework.
	 * Hence, stack_size which is the size of the stack to use,
	 * must be enlarged
	 */
	.stack_size = TA_STACK_SIZE + TA_FRAMEWORK_STACK_SIZE,
	.flags = TA_FLAGS,
	/*
	 * The TA entry doesn't go via this field any longer, to be able to
	 * reliably check that an old TA isn't loaded set this field to a
	 * fixed value.
	 */
	.depr_entry = UINT64_MAX,
};
```

## 1.2 对TA镜像文件的签名
生成原始的TA镜像文件后，编译系统会对该镜像文件进行签名生成最终的xxx.ta文件，该文件会被保存在REE侧的文件系统中。对原始TA镜像文件的签名操作是使用optee_os/scripts/sign.py文件来实现，使用的私钥是optee_os/keys目录下的RSA2048密钥（default_ta.pem）。当该TA需要被正式发布时，应该使用OEM厂商自有的私钥替换掉该密钥。sign.py文件的内容如下：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007193904.png)

# 2. TA镜像加载
当CA第一次调用libteec库中的创建会话操作时，如果被调用的TA是动态TA，则会触发OP-TEE加载该动态TA镜像文件的操作。在加载过程中，OP-TEE会发送PRC请求通知tee_supplicant从文件系统中将UUID对应的TA镜像文件传递到OP-TEE，OP-TEE会对接收到的数据进行验证操作，如果验证通过则将相关段中的内容保存到OP-TEE用户空间分配的TA内存中。加载TA镜像的整体流程如图所示。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007194228.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007194702.png)

## 2.1 REE获取TA文件
OP-TEE通过调用rpc_load函数发送PRC请求，将TA镜像文件的内容从REE侧加载到OP-TEE的共享内存中。该函数会触发两次RPC请求，第一次RPC请求用于获取TA镜像文件的大小，第二次RPC请求是将TA镜像文件加载到OP-TEE的共享内存中。触发第二次RPC请求之前，OP-TEE会在用户空间先分配与TA镜像文件的大小相等的共享内存区域，该区域用于存放TA镜像文件的内容。

对TA镜像文件内容的合法性检查，将TA加载到OP-TEE用户空间TA的内存操作都是在共享内存中完成的。

## 2.2 加载TA镜像的RPC请求
加载TA过程中，ta_open函数会调用rpc_load函数，该函数会调用thread_rpc_cmd来发送OPTEE_MSG_RPC_CMD_LOAD_TA的RPC请求，rpc_load函数会组合该类请求的相关数据结构变量，然后通过调用thread_rpc函数向REE发送RPC请求。

在整个TA的加载过程中会发送两次RPC请求，第一次是用于获取TA镜像文件的大小，第二次RPC请求是通知tee_supplicant将TA镜像文件的内容加载到OP-TEE提供的共享内存中。

## 2.3 RPC请求发送
RPC请求的发送是通过触发安全监控模式调用（smc）来实现的，在触发安全监控模式调用（smc）之前会将当前的线程挂起，并保存该线程的运行上下文。

当REE处理完RPC请求后，会发送标准安全监控模式调用（std smc）重新进入到OP-TEE中，OPTEE根据返回的安全监控模式调用（smc）的类型判定当前的安全监控模式调用（smc）是RPC的返回还是普通的安全监控模式调用（smc）。如果该安全监控模式调用（smc）是返回RPC请求的处理结果，则会进入到thread_resume_from_rpc分支恢复之前被挂起的线程。在thread_rpc函数中已经指定了恢复该线程之后程序执行的入口函数—— thread_rpc_return，到此一次完整的RPC请求也就被处理完毕。

## 2.4 读取TA内容到共享内存
rpc_load函数发起第二次RPC请求时才会将TA镜像文件的内容读取到OP-TEE提供的共享内存中，共享内存的分配是在rpc_load函数中调用thread_rpc_alloc_payload函数来实现的。分配的共享内存的地址将会被保存到ta_handle变量的nw_ta成员中，读取到的TA镜像文件的内容将会被加载到OP-TEE用户空间TA运行的内存中。


# 3. TA合法性验证
当TA镜像文件被加载到共享内存后，OP-TEE会对获取到的数据进行合法性检查。检查TA镜像文件中的哈希（hash）值、magic值、flag值等是否一致，并对镜像文件中的电子签名部分做验证。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007195147.png)

## 3.1 RSA公钥产生和获取
编译整个工程时会生成一个ta_pub_key.c文件，该文件中存放的是RSA公钥，用于验证TA镜像文件合法性。该文件是在编译gensrcs-y目标中的ta_pub_key成员时生成的，该部分的内容定义在optee_os/core/sub.mk文件中。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007195334.png)

编译ta_pub_key目标时会调用recipe-ta_pub_key命令来生成ta_pub_key.c文件，该文件被保存在optee_os/out/arm/core/目录中。recipe-ta_pub_key命令调用pem_to_pub_c.py文件解析optee_os/keys目录中的RSA密钥来获取RSA公钥，并将该公钥保存到ta_pub_key.c文件中。

## 3.2 合法性检查
对TA镜像文件内容合法性的检查是通过调用check_shdr函数来实现的，该函数除了会对TA镜像文件中的签名信息进行验签操作外，还会校验TA镜像文件的shdr部分。

校验TA镜像签名时使用的RSA公钥是由ta_public_key.c文件中的ta_pub_key_exponent和ta_pub_key_modulus变量的值组成。

# 4. 加载TA到OP-TEE用户空间
待共享内存中的TA镜像文件校验通过后，OPTEE就会将共享内存中的TA的内容复制到OP-TEE用户空间的TA内存区域，并初始化该TA运行于用户空间时的上下文。这些操作通过调用load_elf函数来实现。整个TA镜像文件加载到OP-TEE用户空间的过程如图所示。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007195720.png)

TA镜像文件的TA原始文件是ELF格式，在加载前需要先解析该ELF格式文件，获取该ELF文件中哪些段在运行时是必需的、需要保存在什么位置，从而决定用户空间中该TA运行时需要的内存大小和堆栈空间大小。解析完后再将ELF格式的必要段的内容复制到为该TA分配的OP-TEE用户空间内存中。

# 5. TA运行上下文的初始化
TA镜像的内容从共享内存复制到OP-TEE用户空间内存区域后会返回到ta_load函数继续执行，执行初始化该TA运行上下文的操作，并将该上下文添加到OP-TEE的TA运行上下文队列中。
```C
static TEE_Result ta_load(struct tee_tadb_ta_read *ta)
{
	struct thread_param params[2] = { };
	TEE_Result res;
	const size_t sz = ta->entry.prop.custom_size + ta->entry.prop.bin_size;

	if (ta->ta_mobj)
		return TEE_SUCCESS;

	ta->ta_mobj = thread_rpc_alloc_payload(sz);
	if (!ta->ta_mobj)
		return TEE_ERROR_OUT_OF_MEMORY;

	ta->ta_buf = mobj_get_va(ta->ta_mobj, 0, sz);
	assert(ta->ta_buf);

	params[0] = THREAD_PARAM_VALUE(IN, OPTEE_RPC_FS_READ, ta->fd, 0);
	params[1] = THREAD_PARAM_MEMREF(OUT, ta->ta_mobj, 0, sz);

	res = thread_rpc_cmd(OPTEE_RPC_CMD_FS, ARRAY_SIZE(params), params);
	if (res) {
		thread_rpc_free_payload(ta->ta_mobj);
		ta->ta_mobj = NULL;
	}
	return res;
}
```

待ta_load执行完后，加载TA镜像到OP-TEE的操作也就全部完成。在CA中执行的创建会话操作会得到该TA的会话ID，用于REE侧的CA对该TA执行调用命令的操作。

# 6. 总结
本章节主要介绍OP-TEE在执行创建会话操作时加载动态TA的全过程，OP-TEE通过发送RPC请求通知REE侧的tee_supplicant将文件系统中的TA镜像文件加载到OP-TEE分配的共享内存中，然后对共享内存中的数据进行合法性检查，并将必要段的内容复制到分配的OP-TEE用户空间。本章节同时也介绍了对TA镜像文件进行合法性检查时使用的密钥的生成以及TA镜像文件的签名和验签过程。