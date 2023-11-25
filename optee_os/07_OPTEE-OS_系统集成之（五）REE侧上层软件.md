# 07_OPTEE-OS_系统集成之（五）REE侧上层软件
OP-TEE在REE侧的上层软件包括libteec库和tee_supplicant。
* libteec库提供CA程序运行时的基本接口；
* tee_supplicant处理来自TEE侧的RPC请求。

libteec库和tee_supplicant属于REE侧用户空间的功能，属于OP-TEE架构中的重要组成部分。

# 1. OP-TEE软件架构
OP-TEE的软件分为REE侧部分和TEE侧部分，分别包括：
* CA (用户空间)
* REE侧接口库（libteec）
* 常驻进程 （tee_supplicant）
* OP-TEE驱动
* OP-TEE OS、 
* TA等部分。

使用OP-TEE来实现特定的安全功能需要开发者根据实际需求开发特定的CA和TA程序并集成到OP-TEE中。CA端负责在REE侧实现该新功能在用户空间的对外接口，TA端的代码则是在OP-TEE OS的用户空间负责实现具体的安全功能，例如使用何种算法组合来对数据进行安全处理、对处理后的数据的安全保存、解密加密数据等功能。在OP-TEE中划分了安全世界和正常世界，两个世界通过session进行通信，通过COMMAND确定执行对象，通过shared memory来进行大量的数据交换。如图所示[^1]：

ARM在硬件上将处理器分为两种模式（trustzone结构）如图所示：
<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221004190156.png" width="50%" /></div>

对应的OPTEE在软件栈上需要协同trustzone：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221004185120.png" width="100%" /></div>

借助OP-TEE来实现特定安全需求时，一次完整的功能调用一般是起源于CA，TA实现具体功能并返回结果数据给CA。整个过程需要经过OP-TEE的客户端接口、OP-TEE在Linux内核端的驱动、 Monitor模式/EL3下安全监控模式调用（smc）的处理、OP-TEE OS的线程处理、OP-TEE中的TA程序运行、OP-TEE端底层库或者硬件资源支持等几个阶段。当TA执行完具体请求之后会按照原路径将数据返回给CA。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221004185005.png" width="100%" /></div>

不同厂商对具体API的具体实现不一样，但是其功能和对外接口都是遵循GP（Global Platform）的规范来进行封装。例如，海思和Mstar在实现CA端的API的方案不相同，海思在添加TA和CA时，在驱动层和TEE侧都会对调用TEE服务的进程或者线程做权限检查，建立类似白名单机制，在海思的TEE中添加TA和CA时必须注意将调用CA端接口的进程注册到TEE中。

由于当前所有厂商的TEE方案都会遵循GP标准，OP-TEE也遵循GP规范，本书中涉及的API的实现以OP-TEE中的源代码为准。

# 2. REE侧libteec库

CA使用libteec库中提供的接口来实现对TEE侧TA中具体命令的调用。libteec库是OP-TEE提供给用户在Linux用户空间使用的接口的实现，对于该部分每家芯片厂商可能不一样，但对外的接口都遵循GP规范中CA的接口进行定义。本章将以OP-TEE的实现方法为例进行介绍。

libteec库的所有源代码存放在 optee_client/libteec目录下，OP-TEE提供给Linux端使用的接口源代码的实现存放在`optee_client/libteec/src/tee_client_api.c`文件中。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221004190713.png" width="33%" /></div>

libteec库提供给上层用户使用的API一共有10个，都按照GP标准进行定义，使用这10个API能够满足用户在Linux用户空间的需求，在系统中这部分会被编译成libteec库，保存在REE侧的文件系统中以备上层使用。上述10个函数的功能和实现说明如下：

```C
TEEC_Result TEEC_InitializeContext(const char *name, TEEC_Context *context);

void TEEC_FinalizeContext(TEEC_Context *context);

TEEC_Result TEEC_OpenSession(TEEC_Context *context,
			     TEEC_Session *session,
			     const TEEC_UUID *destination,
			     uint32_t connectionMethod,
			     const void *connectionData,
			     TEEC_Operation *operation,
			     uint32_t *returnOrigin);
			     
void TEEC_CloseSession(TEEC_Session *session);

TEEC_Result TEEC_InvokeCommand(TEEC_Session *session,
			       uint32_t commandID,
			       TEEC_Operation *operation,
			       uint32_t *returnOrigin);

TEEC_Result TEEC_RegisterSharedMemoryFileDescriptor(TEEC_Context *ctx, TEEC_SharedMemory *shm,int fd);

TEEC_Result TEEC_RegisterSharedMemory(TEEC_Context *context,
				      TEEC_SharedMemory *sharedMem);
				      
TEEC_Result TEEC_AllocateSharedMemory(TEEC_Context *context,
				      TEEC_SharedMemory *sharedMem);
				      
void TEEC_RequestCancellation(TEEC_Operation *operation);
```

## 2.1 APIs
### 2.1.1 TEEC_InitializeContext
【定义】：`TEEC_Result TEEC_InitializeContext(const char *name, TEEC_Context *context);`

【解释】：初始化一个TEEC_Context变量，该变量用于CA和TEE之间建立联系。其中参数name用来定义TEE的身份，如果该参数为NULL，则CA将会选择默认的TEE方案来建立联系。该API必须是CA调用的第一个libteec库的API，且该API不会触发TA的执行。

### 2.1.2 TEEC_FinalizeContext
【定义】：`void TEEC_FinalizeContext(TEEC_Context *context);`

【解释】：释放一个已经被初始化的类型为TEEC_Context的变量，关闭CA与TEE之间的连接。在调用该函数之前必须确保打开的session已经被关闭。

### 2.1.3 TEEC_OpenSession

【定义】：
```C
TEEC_Result TEEC_OpenSession(TEEC_Context *context,
			     TEEC_Session *session,
			     const TEEC_UUID *destination,
			     uint32_t connectionMethod,
			     const void *connectionData,
			     TEEC_Operation *operation,
			     uint32_t *returnOrigin);
```
【解释】：打开一个CA与对应TA之间的一个session，该session用于**CA与对应TA之间的联系**，CA需要连接的TA是由UUID指定的。session具有不同的打开和连接方式，根据不同的打开和连接方式CA可以在执行打开session时传递数据给TA，以便TA对打开操作进行权限检查。各种打开方式说明如下：

|打开方式|说明|
|-----|-----|
|TEEC_LOGIN_PUBLIC | 不需要提供，即 connectionData的值必须为NULL。| 
|TEEC_LOGIN_USER | 提示用户链接， connectionData的值必须为NULL。| 
|TEEC_LOGIN_GROUP | CA以组的方式打开session。connectionData的值必须指向一个类型为 uint32_t的数据，其包含某一组的特定信息。在TA 端将会对connectionData的数据进行检查，判定CA 是否真属于该组。 |
|TEEC_LOGIN_APPLICATION | 以application的方式连接，connectionData的值必须为NULL。| 
|TEEC_LOGIN_USER_APPLICATION|以用户程序的方式连接，connectionData的值必须为 NULL。|
|TEEC_LOGIN_GROUP_APPLICATION|以组应用程序的方式连接，其中connectionData需要指向 一个uint32_t类型的变量。在TA端将会对 connectionData的数据进行权限检查，查看连接是否 合法。|

【传递参数】：

|参数 | 说明 |
|-----|------|
|context|指向一个类型为TEEC_Context的变量，该变量用于CA与TA之间的连接和通信，调用TEEC_InitializeContext函数进行初始化；|
|session | 存放session内存的变量； |
|destination | 指向存放需要连接TA的UUID的值的变量；|
|connectionMethod | CA与TA的连接方式，详细可参考函数描述中的说明； |
|connectionData | 指向需要在打开session时传递给TA的数据； |
|operation | 指向TEEC_Operation结构体的变量，变量中包含了一系列用于与TA进行交互使用的buffer或者其他变量。如果在打开session时CA和 TA不需要交互数据，则可以将该变量指向NULL；|
|returnOrigin | 用于存放从TA端返回的结果的变量。如果不需要返回值，则可以将该变量指向NULL。|

相应地，TEEC_CloseSession和opensession成对出现。

opensession内部使用了ioctl的接口，还有一些内存共享的实现。

https://github.com/OP-TEE/optee_client/blob/master/libteec/src/tee_client_api.c#L594

### 2.1.4 TEEC_InvokeCommand
【定义】：
```C
TEEC_Result TEEC_InvokeCommand(TEEC_Session *session,
			       uint32_t commandID,
			       TEEC_Operation *operation,
			       uint32_t *returnOrigin);
```

【解释】：通过cmd_id和打开的session来通知session对应的TA执行cmd_id指定的操作。

### 2.1.5 TEEC_RequestCancellation
【原型】：`void TEEC_RequestCancellation(TEEC_Operation *operation);`
【解释】：取消某个CA与TA之间的操作，该接口只能由除执行TEEC_OpenSession和TEEC_InvokeCommand的线程之外的其他线程进行调用，而TA端或者TEE OS可以选择并不响应该请求。只有当operation中的 started域被设置成0之后，该操作方可有效。

### 2.1.6 TEEC_RegisterShareMemory
```C
TEEC_Result TEEC_RegisterSharedMemory(TEEC_Context *context,
				      TEEC_SharedMemory *sharedMem);
				      
```
注册一块在CA端的内存作为CA与TA之间的共享内存。shareMemory结构体中的三个成员如下： 
- buffer：指向作为共享内存的起始地址； 
- size：共享内存的大小； 
- flags：表示CA与TA之间的数据流方向。

### 2.1.7 TEEC_RegisterShareMemoryFileDescriptor
```C
TEEC_Result TEEC_RegisterSharedMemoryFileDescriptor(TEEC_Context *ctx, TEEC_SharedMemory *shm,int fd
```
注册一个在CA与TA之间的共享文件，在CA端会将文件的描述符fd传递给OP-TEE，其内容被存放到shm中。

### 2.1.8 TEEC_AllocateSharedMemory
```C			      
TEEC_Result TEEC_AllocateSharedMemory(TEEC_Context *context,
				      TEEC_SharedMemory *sharedMem);
```
分配一块共享内存，共享内存是由OP-TEE分配的，OP-TEE分配了共享内存之后将会返回该内存块的fd给CA，CA将fd映射到系统内存，然后将地址保存到shm中。

除此之外还有对应的TEEC_ReleaseSharedMemory来释放分配的内存。

## 2.2 CA调用libteec库接口流程
CA在使用libteec库中的接口来实现调用TA的操作时，一般过程是需要先建立context，然后建立与需要调用的TA之间的session，再通过执行invoke操作向TA发送command ID来实现具体的操作需求，待TA中command ID的内容执行完成之后，如果后续也不需要再次调用TA时，可以通过close session和final context来释放资源，完全关闭该CA与TA之间的联系。一次完整的操作过程如图所示。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221004193623.png" width="60%" /></div>

- （1）调用CA接口初始化context；
- （2）打开session（此时CA已经向TA发送请求）
- （3）初始化TEEC_PARAM_TYPES 初始化参数和缓存使用invokeCommand向TA发送命令；
- （4）关闭session；
- （5）释放资源，返回结果给REE。 


# 3. REE守护进程tee_supplicant
tee_supplicant是常驻在Linux内核中的一个进程，主要作用是使OP-TEE能够通过tee_supplicant来访问REE侧的资源。例如加载存放在文件系统中的TA镜像到TEE中，对REE侧数据库的操作，对EMMC中RPMB分区的操作，提供socket通信等。 其源代码在optee_client/tee-supplicant目录中。编译之后会生成一个名为tee_supplicant的可执行文件， 该可执行文件在REE启动时会作为一个后台守护程序被自动启动。

## 3.1 tee_supplicant编译生成和自启动
tee_supplicant会在编译optee-client目标时被编译生成一个可执行文件。tee_supplicant可执行文件在Linux启动时会被作为后台程序启动。启动的动 作存放在build/init.d.optee文件中，其内容如下：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221004194921.png" width="70%" /></div>

在编译时，init.d.optee文件将会被打包到根文件系统中并以optee名字存放在/etc/init.d目录中，而 且会被链接到`/etc/rc.d/S09_optee`文件。这些操作是在编译生成rootfs时进行的，详细情况可查看build/common.mk文件中filelist-tee-common目标的内容。系统启动tee_supplicant的过程如图所示：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221004195243.png" width="60%" /></div>

在最后的设备端的REE上，有这个文件，内部启动supplicant：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typoratelegram-cloud-document-5-6188192002518025850.jpg" width="80%" /></div>

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typoratelegram-cloud-document-5-6188192002518025851.jpg" width="80%" /></div>

## 3.2 tee_supplicant入口函数
tee_supplicant作为Linux中的一个守护进程，起到处理RPC请求的服务器端的作用，通过类似于C/S的方式，为OP-TEE提供对REE侧资源进行操作 的实现。该可执行文件的入口函数存放在 `optee_client/tee-supplicant/src/tee_supplicant.c`文件中。其入口函数内容如下：

https://github.com/OP-TEE/optee_client/blob/master/tee-supplicant/src/tee_supplicant.c#L790

## 3.3 tee_supplicant存放RPC请求的结构体
在tee_supplicant中用于接收和发送请求的数据都存放在类型为tee_rpc_invoke的结构体变量中。该结构体内容如下：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221004200056.png" width="67%" /></div>


tee_rpc_invoke结构体中的数据展开之后的组成如图所示：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221004200241.png" width="67%" /></div>

## 3.4 tee_supplicant的监听程序
tee_supplicant启动后最终会进入一个无限循环，调用process_one_request函数来监控、接收、 处理、回复OP-TEE的请求。如图所示为tee_supplicant处理RPC请求过程：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221004201036.png" width="67%" /></div>

参考： https://github.com/OP-TEE/optee_client/blob/master/tee-supplicant/src/tee_supplicant.c#L613

tee_supplicant获取TA的RPC请求。tee_supplicant通过read_request接收来自TA端的请求。该函数会阻塞tee驱动层面，其内容如下：
```C
static bool read_request(int fd, union tee_rpc_invoke *request)
{
	struct tee_ioctl_buf_data data;

	memset(&data, 0, sizeof(data));

	data.buf_ptr = (uintptr_t)request;
	data.buf_len = sizeof(*request);
	if (ioctl(fd, TEE_IOC_SUPPL_RECV, &data)) {
		EMSG("TEE_IOC_SUPPL_RECV: %s", strerror(errno));
		return false;
	}
	return true;
}
```
在OP-TEE驱动中ioctl的TEE_IOC_SUPPL_RECV操作将会阻塞，直到接收 到来自TA的请求。

当tee_supplicant解析出RPC请求的功能ID，并根据该ID找到对应的处理函数，完成TEE请求操作后，tee_supplicant通过调用write_response函数将处 理结果和数据返回给TA。该函数的内容和解释如下：

```C
static bool write_response(int fd, union tee_rpc_invoke *request)
{
	struct tee_ioctl_buf_data data;

	memset(&data, 0, sizeof(data));

	data.buf_ptr = (uintptr_t)&request->send;
	data.buf_len = sizeof(struct tee_iocl_supp_send_arg) +
		       sizeof(struct tee_ioctl_param) *
				(__u64)request->send.num_params;
	if (ioctl(fd, TEE_IOC_SUPPL_SEND, &data)) {
		EMSG("TEE_IOC_SUPPL_SEND: %s", strerror(errno));
		return false;
	}
	return true;
}
```

## 3.5 RPC的请求处理
当解析完来自TA的RPC请求，获取到具体参数后，在process_one_request函数中会根据请求的功能ID来决定具体执行什么操作。这些操作包括：
- 从文件系统中读取TA的镜像保存在共享内存 中；
- 对文件系统中的节点进行读/写/打开/关闭/移 除等操作； 
- 执行RPMB（EMMC中的RPMB分区）相关操 作；
- 分配共享内存； 
- 释放共享内存； 
- 处理gprof请求； 
- 执行网络socket请求。

# 4. RPC请求
tee_supplicant获取到远程过程调用（Remote Procedure Call，RPC）请求后会解析出功能ID，然后根据该ID值来命中tee_supplicant提供的具体操作。当请求处理完成后会将处理结果和数据发送给OP-TEE驱动，OP-TEE驱动最终会触发安全监控模式调用（smc）将数据传递给OP-TEE。

## 4.1 加载TA镜像
请求加载TA镜像的功能ID为`RPC_CMD_LOAD_TA`。执行该功能时， tee_supplicant会到文件系统中将TA镜像的内容读取到共享内存中。该操作是通过调用load_ta函数来实现的，该函数定义在tee_supplicant.c文件中，在 REE侧加载TA镜像文件的整体流程如图所示。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221004202642.png" width="67%" /></div>

```C
static uint32_t load_ta(size_t num_params, struct tee_ioctl_param *params)
{
	int ta_found = 0;
	size_t size = 0;
	struct param_value *val_cmd = NULL;
	TEEC_UUID uuid;
	TEEC_SharedMemory shm_ta;

	memset(&uuid, 0, sizeof(uuid));
	memset(&shm_ta, 0, sizeof(shm_ta));

	if (num_params != 2 || get_value(num_params, params, 0, &val_cmd) ||
	    get_param(num_params, params, 1, &shm_ta))
		return TEEC_ERROR_BAD_PARAMETERS;

	uuid_from_octets(&uuid, (void *)val_cmd);

	size = shm_ta.size;
	ta_found = TEECI_LoadSecureModule(supplicant_params.ta_dir, &uuid, shm_ta.buffer, &size);
	if (ta_found != TA_BINARY_FOUND) {
		EMSG("  TA not found");
		return TEEC_ERROR_ITEM_NOT_FOUND;
	}

	MEMREF_SIZE(params + 1) = size;

	/*
	 * If a buffer wasn't provided, just tell which size it should be.
	 * If it was provided but isn't big enough, report an error.
	 */
	if (shm_ta.buffer && size > shm_ta.size)
		return TEEC_ERROR_SHORT_BUFFER;

	return TEEC_SUCCESS;
}
```

当load_ta执行完成并正确读取了TA镜像文件的信息之后，最终会将读取到的数据通过调用write_response函数，将数据发送给OP-TEE驱动，由驱动来完成将数据发送给OP-TEE的操作。OP-TEE会对接收到的TA镜像的合法性进行校验，主要是验证TA镜像文件的电子签名是否合法。

## 4.2 操作REE侧文件系统

当功能ID为RPC_CMD_FS时，tee_supplicant会根据TA请求调用tee_supp_fs_process函数来完成对 文件系统的具体操作，包括常规的文件和目录的打开、关闭、读取、写入、重命名、删除等。其内容如下：

https://github.com/OP-TEE/optee_client/blob/master/tee-supplicant/src/tee_supp_fs.c

```C
TEEC_Result tee_supp_fs_process(size_t num_params,
				struct tee_ioctl_param *params)
{
	if (!num_params || !tee_supp_param_is_value(params))
		return TEEC_ERROR_BAD_PARAMETERS;

	if (strlen(tee_fs_root) == 0) {
		if (tee_supp_fs_init() != 0) {
			EMSG("error tee_supp_fs_init: failed to create %s/",
				tee_fs_root);
			memset(tee_fs_root, 0, sizeof(tee_fs_root));
			return TEEC_ERROR_STORAGE_NOT_AVAILABLE;
		}
	}

	switch (params->a) {
	case OPTEE_MRF_OPEN:
		return ree_fs_new_open(num_params, params);
	case OPTEE_MRF_CREATE:
		return ree_fs_new_create(num_params, params);
	case OPTEE_MRF_CLOSE:
		return ree_fs_new_close(num_params, params);
	case OPTEE_MRF_READ:
		return ree_fs_new_read(num_params, params);
	case OPTEE_MRF_WRITE:
		return ree_fs_new_write(num_params, params);
	case OPTEE_MRF_TRUNCATE:
		return ree_fs_new_truncate(num_params, params);
	case OPTEE_MRF_REMOVE:
		return ree_fs_new_remove(num_params, params);
	case OPTEE_MRF_RENAME:
		return ree_fs_new_rename(num_params, params);
	case OPTEE_MRF_OPENDIR:
		return ree_fs_new_opendir(num_params, params);
	case OPTEE_MRF_CLOSEDIR:
		return ree_fs_new_closedir(num_params, params);
	case OPTEE_MRF_READDIR:
		return ree_fs_new_readdir(num_params, params);
	default:
		return TEEC_ERROR_BAD_PARAMETERS;
	}
}
```

tee_supp_fs_process函数主要是对REE侧文件系统进行操作。如果执行的是open、create操作则会返回文件的操作句柄fd值给OP-TEE；如果是write操作则会将需要写的内容写到具体的文件中。

## 4.3 操作RPMB

当功能ID为RPC_CMD_RPMB时，tee_supplicant会根据TA请求调用process_rpmb函数 来完成对EMMC中rmpb分区的操作。EMMC中的RPMB分区，在读写过程中会执行验签和加解密的操作。其内容如下：
https://github.com/OP-TEE/optee_client/blob/master/tee-supplicant/src/rpmb.c

```C
uint32_t rpmb_process_request(void *req, size_t req_size, void *rsp, size_t rsp_size) {
	uint32_t res = 0;
	tee_supp_mutex_lock(&rpmb_mutex);
	res = rpmb_process_request_unlocked(req, req_size, rsp, rsp_size);
	tee_supp_mutex_unlock(&rpmb_mutex);
	return res;
}
```

## 4.4 分配释放共享内存

当功能ID为RPC_CMD_SHM_ALLOC时，tee_supplicant会根据TA请求调用process_alloc函数来分配TA与tee_supplicant之间的共享内存。其内容如下：
```C
static uint32_t process_alloc(struct thread_arg *arg, size_t num_params,
			      struct tee_ioctl_param *params)
{
	struct param_value *val = NULL;
	struct tee_shm *shm = NULL;

	if (num_params != 1 || get_value(num_params, params, 0, &val))
		return TEEC_ERROR_BAD_PARAMETERS;

	if (arg->gen_caps & TEE_GEN_CAP_REG_MEM)
		shm = register_local_shm(arg->fd, val->b);
	else
		shm = alloc_shm(arg->fd, val->b);

	if (!shm)
		return TEEC_ERROR_OUT_OF_MEMORY;

	shm->size = val->b;
	val->c = shm->id;
	push_tshm(shm);

	return TEEC_SUCCESS;
}
```
当功能ID为RPC_CMD_SHM_FREE时，tee_supplicant会根据TA请求调用process_free函数来释放TA与tee_supplicant之间的共享内存。

## 4.5 socket
当功能ID为OPTEE_MSG_RPC_CMD_SOCKET时， tee_supplicant会根据TA请求调用tee_socket_process 函数来完成网络套接字（socket）的相关操作，包括网络套接字的建立、发送、接收和ioctl操作。其内容如下：

https://github.com/OP-TEE/optee_client/blob/master/tee-supplicant/src/tee_socket.c

```C
TEEC_Result tee_socket_process(size_t num_params,
			       struct tee_ioctl_param *params)
{
	if (!num_params || !tee_supp_param_is_value(params))
		return TEEC_ERROR_BAD_PARAMETERS;

	switch (params->a) {
	case OPTEE_MRC_SOCKET_OPEN:
		return tee_socket_open(num_params, params);
	case OPTEE_MRC_SOCKET_CLOSE:
		return tee_socket_close(num_params, params);
	case OPTEE_MRC_SOCKET_CLOSE_ALL:
		return tee_socket_close_all(num_params, params);
	case OPTEE_MRC_SOCKET_SEND:
		return tee_socket_send(num_params, params);
	case OPTEE_MRC_SOCKET_RECV:
		return tee_socket_recv(num_params, params);
	case OPTEE_MRC_SOCKET_IOCTL:
		return tee_socket_ioctl(num_params, params);
	default:
		return TEEC_ERROR_BAD_PARAMETERS;
	}
}
```

supplicant给TA提供服务，TA可以通过这些功能远程对REE测一些资源进行操作。

# Ref
[^1]:[Demystifying ARM TrustZone TEE Client API using OP-TEE](https://github.com/carloscn/doclib/blob/master/paper/software/trustzone/Demystifying%20ARM%20TrustZone%20TEE%20Client%20API%20using%20OP-TEE.pdf)