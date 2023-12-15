
# 1. 背景

由于私钥和对称秘钥具有非常高的机密性，不允许泄露；公钥和证书则具有完整性要求，禁止未授权更改，因此，如何存储具有机密性的数据和完整性保护要求的数据，是一个严肃且重要的话题。在携带有硬件HSM、密码机或者是硬件HSE等硬件安全模块的设备上，由硬件安全模块提供对敏感数据的机密性保护和完整性保护。但是，对于一些不具备硬件安全模块的设备，同样需要对机密数据和完整性数据进行保护。幸运的是，TEE提供了安全存储的能力，能够为敏感数据提供机密性和完整性保护能力。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312150942519.png)

# 2. TEE的安全基础-- TrustZone

TrustZone提供了一些基础的安全能力，基于这些安全能力，上层可以构建不同的安全机制，比如说TEE、Secure Boot等。其中，将内存分为安全内存和非安全内存是TrustZone的一项重要特性，其主要目的包括：

1. **隔离敏感数据：** TrustZone允许将敏感数据存储在安全内存中，而将非敏感数据存储在非安全内存中。这样，敏感数据在处理器的安全区域内受到保护，不容易被恶意软件或非授权的应用程序访问和泄露。
2. **执行安全代码：** 安全内存还用于存储安全性相关的代码，如加密和认证算法，以及安全启动代码。这些代码在非安全内存中不可见，从而提供了一种方式来执行和保护安全关键操作。
3. **隔离不受信任的应用：** 非安全内存用于存储不受信任的应用程序和操作系统组件。TrustZone通过硬件隔离确保这些应用程序和组件无法直接访问安全内存中的敏感数据或安全代码。
4. **防止侧信道攻击：** 通过将安全内存隔离，TrustZone有助于防止侧信道攻击，如缓存侧信道攻击和时序侧信道攻击，因为非安全的应用程序无法直接观察或干扰安全内存中的操作。
5. **提供硬件支持：** TrustZone利用ARM处理器的硬件支持，确保内存分区的强制执行。这是硬件级别的隔离，不容易被绕过。

具体参考，对于M核心： https://github.com/carloscn/blog/issues/197 （这里只是写trustzone的意义，更多的用的是A核）

## 2.1 TrustZone的安全世界和非安全世界的切换

在介绍TrustZone的世界切换前，先介绍下ARMv8的异常模型。ARMv8权限的分配划分为四个异常等级（Exception Level，EL）,分别是EL0、EL1、EL2和EL3。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312150946541.png)

异常等级只有在以下几种情况下才会变更：
- 产生异常
- 从异常返回
- 处理器复位
- 在debug状态
- 从debug状态返回

在ARMv8中，TrustZone把运行世界划分为了安全和非安全两个状态，结合ARMv8的异常等级来看的话，可以用下图来描述：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312150947673.png)

在EL0、EL1和EL2处理器可能是安全状态，也可能是非安全状态，状态的确定由SCR_EL3这个bit决定，我们可以用NS表示非安全状态(Non-Secure)，而安全状态则由S表示，比如：

- NS.EL1：非安全状态的异常等级1
- S.EL1：安全状态的异常等级

SCR_EL3.NS是一个bit，这个bit只能由EL3来变更，因此，只要保证从非安全世界的EL1或者EL2进入EL3的方式安全，也就能保证安全世界是安全的，而不是非法进入的或者是非法篡改的。为简化介绍，我们这里不考虑hypervisor的机制，只看非安全世界只有一个Rich OS的情况。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312150947030.png)

从非安全世界进入安全世界的步骤可以拆分为三个步骤：

1. APP通过产生异常来进入更高等级的异常等级，产生的异常一般是FIQ或者是调用SMC产生异常，从而从EL1进入EL3
2. 进入EL3后，由运行在EL3的Firmware/Secure Monitor变更SCR_EL3.NS，从1到0
3. 从异常返回时，由于SCR_EL3.NS为0，因此返回的是安全世界的EL1

### #2.1.1 寄存器管理

在从非安全世界切换到非安全世界时，还涉及了寄存器的管理。对于通用寄存器以及大多数系统寄存器，不区分安全世界，只有一份，因此必须由软件去管理这些寄存器的备份和恢复，这不是由硬件去保证的。图中的EL3中的Secure Monitor就是负责在安全世界和非安全世界切换时来备份和恢复这些寄存器，他的动作主要有：

- 从非安全世界进入安全世界时
    - 变更SCR_EL3.NS为0
    - 保存非安全世界的寄存器状态
    - 恢复安全世界的寄存器状态
- 从安全世界进入非安全世界时，正好相反
    - 变更SCR_EL3.NS为1
    - 保存安全世界的寄存器状态
    - 恢复非安全世界的寄存器状态

有一小部分的寄存器由安全状态存储，也就是说系统中将存储两份这些寄存器的值，CPU自动使用属于当前状态的寄存器值。比如说ICC_BPR1_EL1，一个用于控制中断抢占的GIC寄存器。当系统寄存器被存储时，可以通过用(S)和(NS)来标识使用哪个寄存器，例如：ICC_BPR1_EL1(S) 和 ICC_BPR1_EL1(NS) 。


### 2.1.2 内存管理

对于安全世界和非安全世界，系统提供了两种不同的内存映射原理：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312150949940.png)

在获取虚拟内存地址时，根据地址的不同，分别从不同的内存映射机制进行地址映射：

- NS.EL1: 0x7000 -- 映射到非安全世界的0x7000虚拟地址
- S.EL1: 0x7000 -- 映射到安全世界的0x7000虚拟地址

也就是说安全世界和非安全世界有着两套独立的虚拟地址空间，相同的地址并不能跨世界访问，而只能访问当前状态下的虚拟地址。虚拟地址隔离还不够，物理地址也要隔离。在TrustZone的架构中，物理地址同样提供了两套物理地址空间：安全世界的和非安全的。当在非安全世界时，虚地址总是会映射到非安全世界的物理地址空间，这也就保证了处于非安全世界的软件只能访问非安全世界的资源，绝不可能访问安全世界的资源。而当处于安全世界时，软件既可以访问安全世界的物理地址空间，也可以访问非安全世界的物理地址空间，这时候的NS bit的作用就是判断虚拟地址是映射到非安全世界的物理地址空间还是安全世界的物理地址空间，用两张图来表示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312150949837.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312150949407.png)

通过严格管理非安全世界进入安全世界的接口，以及对内存进行安全世界和非安全世界的访问进行管控，安全世界的代码杜绝了被非安全世界篡改的可能性，数据也只能被安全世界或者合法进入安全世界后更改。

# 3. Secure Storage In TEE

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312150950760.png)

在TrustZone的架构中，芯片上的软件系统被分为安全世界和非安全世界，其中安全世界也被称为TEE(Trusted Execution Environment)，非安全世界也被成为REE(Rich Execution Environment)。

TEE的Secure Storage有两种实现方法：一种是基于REE的普通文件系统来实现，另一种则是基于eMMC的RPMB(Replay Protected Memory Block)分区来实现。在OP-TEE的规范中，是可以同时使用REE的文件系统和eMMC的RPMB分区作为secure storage的物理存储介质的。如果使用REE的文件系统作为secure storage的存储介质，那么可以将TEE对该文件系统的访问简要概括为以下模型：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312150950218.png)

当TEE侧的TA需要写入Secure Storage的数据时，TA通过GP的Trusted Storage API去访问TEE的文件操作接口，然后TEE文件系统会使用密钥去加密要写入的数据，然后通过REE的文件操作接口去对REE的文件系统进行写入。读取过程则是一个相反的过程。

GP Trusted Storage要满足以下需求：
- Trusted storage可以由非安全资源实现，只要应用适当的密码学保护措施，密码学措施的强度应该至少和TEE的代码和数据具有相同强度
- Trusted storage必须只允许被授权的TA访问和修改
- TA有能力隐藏用于加密数据的秘钥材料
- 每个TA都有能力访问自己的存储空间，同时也能满足隔绝于其他TA的访问的需求

TEE的secure storage也应遵守以上要求。

## 3.1 TEE的secure storage文件存储在哪里

默认情况下，OP-TEE将数据存储在/data/tee下，为了区分不同的TA的存储空间，OP-TEE为每个TA分配了UUID，因此可以在/data/tee下再以TA的UUID为文件夹，建立不同TA的存储空间。同一个TA，可能也会建立不同的Object，因此可以为Object分配ID，然后再在同一个TA下进行不同的Object的管理。

在TEE的最细粒度的文件夹(也就是Object）下，有一个meta文件和其他一些块文件。meta文件用于存储当前文件的一些TEE信息，TEE就是使用这些信息来管理TEE的文件的，而块文件则是真正的数据。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312150950876.png)

## 3.2 Key manager

前面提到，TEE在写数据时需要将数据加密后再写入REE的文件系统中，而读取时需要将从REE文件系统读到的数据进行解密才能由TEE访问。Key manager在里面就负责了数据的加解密和秘钥的管理。在TEE中，Key manager主要管理三种密钥：Secure Storage Key(SSK)、TA Storage Key(TSK)和File Encryption Key(FEK)。TEE如何做到每个设备、每个TA、甚至每个文件都能独立安全存储，关键就在于这三个秘钥的生成和管理。

Note，一般情况下，HUK托管给SoC的加解密引擎，这部分可以参考： https://github.com/carloscn/blog/issues/206

### 3.2.1 Secure Storage Key

SSK是设备唯一的秘钥，它在TEE启动过程中生成，并存储在安全内存中，因此能确保它的安全性。SSK用于生成TSK。SSK的生成可以用下面的公式来表达：

> SSK = HMACSHA256 (HUK, Chip ID || “static string”)

获取HUK(Hardware Unique Key)和Chip ID的功能和硬件平台执行强相关。那么，SSK是如何保证机密性的呢？由SSK的派生算法可以看出，SSK的机密性取决于HUK(Hardware Unique Key)，因为chip Id和static string都是固定不变的。因此HUK就成了整个Secure Storage的机密性的根。在TEE的实现中，HUK的获取一般都只实现了一个桩，实际实现由用户自定义，这是因为不同芯片的HUK实现机制不一样，OPTEE关于如何获取HUK的定义在`core/include/kernel/tee_common_otp.h`中。

关于HUK的最佳实现是HUK无法被软件读取，甚至是安全侧的软件。对于这个要求有不同的实现方案，可以采用密码学加速器甚至是协处理器来完成。在Layerscape的实现中，NXP采用的是Cryptographic Accelerator and Assurance Module(CAAM)。CAAM是一个SOC内嵌的硬件模块，他不仅支持安全RAM和公钥密码学硬件加速（PKHA, Public-Key Hardware Accelerator），还在出厂预存了芯片唯一的不可变更、不可读取的256-bit的随机值，这个随机值存储在FUSE的OTPMK（One Time Programmable Master Key）区域，由于是出厂就被blow，因此无法被篡改，并且只能被CAAM获取，一般用于加密其他的非对称秘钥，因此也被称为Black Key。

从Layerscape的TEE实现中，我们也能看到，TEE基于CAAM实现了HUK的获取，因此能确保SSK的机密性：

```C
TEE_Result tee_otp_get_hw_unique_key(struct tee_hw_unique_key *hwkey)
{
	COMPILE_TIME_ASSERT(sizeof(hwkey->data) <= sizeof(stored_key));

	if (!mkvb_retrieved)
		return TEE_ERROR_SECURITY;

	memcpy(&hwkey->data, &stored_key, sizeof(hwkey->data));
	return TEE_SUCCESS;
}
```

其中，stored_key在CAAM的初始化过程中获取：

```C
enum caam_status caam_blob_mkvb_init(vaddr_t baseaddr)
{
	struct caam_jobctx jobctx = { };
	enum caam_status res = CAAM_NO_ERROR;
	struct caambuf buf = { };
	uint32_t *desc = NULL;

	assert(!mkvb_retrieved);

	res = caam_calloc_align_buf(&buf, MKVB_SIZE);
	if (res != CAAM_NO_ERROR)
		goto out;

	desc = caam_calloc_desc(8);
	if (!desc) {
		res = CAAM_OUT_MEMORY;
		goto out_buf;
	}

	caam_desc_init(desc);
	caam_desc_add_word(desc, DESC_HEADER(0));
	caam_desc_add_word(desc, SEQ_OUT_PTR(32));
	caam_desc_add_ptr(desc, buf.paddr);
	caam_desc_add_word(desc, BLOB_MSTR_KEY);
	BLOB_DUMPDESC(desc);

	cache_operation(TEE_CACHEFLUSH, buf.data, buf.length);

	jobctx.desc = desc;
	res = caam_jr_enqueue(&jobctx, NULL);

	if (res != CAAM_NO_ERROR) {
		BLOB_TRACE("JR return code: %#"PRIx32, res);
		BLOB_TRACE("MKVB failed: Job status %#"PRIx32, jobctx.status);
	} else {
		cache_operation(TEE_CACHEINVALIDATE, buf.data, MKVB_SIZE);
		BLOB_DUMPBUF("MKVB", buf.data, buf.length);
		memcpy(&stored_key, buf.data, buf.length);
		mkvb_retrieved = true;
	}

out_buf:
	caam_free_desc(&desc);
	caam_free_buf(&buf);
out:
	caam_hal_ctrl_inc_priblob(baseaddr);

	return res;
}
```

### #3.2.2 Trusted Applicaation Storagge Kay

TSK是一个TA唯一的密钥，也就是说每个TA都有一个自己唯一的TSK，TSK的目的是用来保护FEK，这也就确保了每个TA都只能访问自己的数据，而无法跨域访问其他TA的数据。TSK的生成依赖于SSK和TA的识别码UUID：

> TSK = HMACSHA256 (SSK, TA_UUID)

TSK在runtime过程中并不存储，而是使用过程中直接按照上述式子计算，因此有些文档介绍中，会直接忽略TSK的存在，而是直接描述成采用SSK加密FEK。

```C
	uint8_t tsk[TEE_FS_KM_TSK_SIZE];
	uint8_t dst_key[size];

	if (!in_key || !out_key)
		return TEE_ERROR_BAD_PARAMETERS;

	if (size != TEE_FS_KM_FEK_SIZE)
		return TEE_ERROR_BAD_PARAMETERS;

	if (tee_fs_ssk.is_init == 0)
		return TEE_ERROR_GENERIC;

	if (uuid) {
		res = do_hmac(tsk, sizeof(tsk), tee_fs_ssk.key,
			      TEE_FS_KM_SSK_SIZE, uuid, sizeof(*uuid));
		if (res != TEE_SUCCESS)
			return res;
```

### 3.2.3 File Encryption Key

当一个新的TEE文件被创建时，Key manager就会立即生成一个FEK，FEK的生成是直接采用PRNG(Pesudo Random Number Generator)生成的。这个FEK会被存储在文件对应的meta文件中，以后就用来对该文件进行加解密，TEE的文件信息也会使用这个FEK加密存储在meta文件中。

**Key manager机制不仅确保了数据对非安全世界而言的机密性，同时也确保了数据对于TEE世界中非数据主体TA的机密性，因此可以证明TEE对于机密数据的保护是满足机密性要求的**。

## 3.3 数据保护

如果在Secure Storage中的加密数据被篡改，这通常意味着数据的完整性受到了破坏。在这种情况下，有几种潜在的应对措施和后果：
1. **完整性检查失败**：加密数据通常会伴随着完整性检查机制（如数字签名或哈希校验和）。如果数据被篡改，这些检查通常会失败。这意味着在尝试访问或解密这些数据时，系统会识别到完整性问题。
2. **访问拒绝或错误报告**：当检测到数据完整性问题时，系统可能会拒绝访问被篡改的数据，并可能报告一个错误。这是一种安全措施，旨在防止损坏或被篡改的数据造成更大的安全问题。

Meta数据加密数据流如下所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312150954755.png)

FEK由TA唯一的TSK进行AES加密后，存储在meta文件的头部，同时，会被解密并结合meta IV以及meta数据进行AES加密，生成加密后的meta数据和一个tag。对于未授权的访问，由于FEK无法解密，因此就无法访问加密后的meta数据。

Block数据的加密流程也类似，只是此时的FEK来自于meta文件的头部：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312150957776.png)

此外，TEE对于数据的保护还引入了hash树。对于一个secure storage的文件，hash树负责处理数据的加密和解密，同时记录数据的hash值。TEE中hash树采用二叉树的方式实现，树中每个节点保护了该节点的两个子节点和一个数据块，保护方式就是节点的hash值。其中，meta数据存储在二叉树的头结点。需要注意的是，所有的数据都会有两份，版本分别为0和1，这是为了确保原子更新，原子更新的内容不在这展开。

树节点结构：


```
struct htree_node {
	size_t id;
	bool dirty;
	bool block_updated;
	struct tee_fs_htree_node_image node;
	struct htree_node *parent;
	struct htree_node *child[2];
};
```

其中，`tee_fs_htree_node_image`中包含了hash信息：

```
struct tee_fs_htree_node_image {
        uint8_t hash[TEE_FS_HTREE_HASH_SIZE];
        uint8_t iv[TEE_FS_HTREE_IV_SIZE];
        uint8_t tag[TEE_FS_HTREE_TAG_SIZE];
        uint16_t flags;
};
```

hash树在文件中的形式：

```
 * +----------------------------+
 * | htree_image.0		|
 * | htree_image.1		|
 * +----------------------------+
 * | htree_node_image.1.0	|
 * | htree_node_image.1.1	|
 * +----------------------------+
 * | htree_node_image.2.0	|
 * | htree_node_image.2.1	|
 * +----------------------------+
 * | htree_node_image.3.0	|
 * | htree_node_image.3.1	|
 * +----------------------------+
 * | htree_node_image.4.0	|
 * | htree_node_image.4.1	|
 * +----------------------------+
```


### 3.3.1 创建哈希树

在要创建一个文件时，首先会创建文件的哈希树，TEE内部是通过调用`tee_fs_htree_open`来创建和打开哈希树的。这个可以通过ree为tee提供的文件operations结构来证明：

```
static const struct tee_fs_dirfile_operations ree_dirf_ops = {
	.open = ree_fs_open_primitive,
	.close = ree_fs_close_primitive,
	.read = ree_fs_read_primitive,
	.write = ree_fs_write_primitive,
	.commit_writes = ree_dirf_commit_writes,
};
```

其中，`ree_fs_open_primitive`最后会调用`tee_fs_htree_open`，因此无论是创建文件还是打开文件，都会走入哈希树的open环节。

哈希树根节点结构体：

```
struct tee_fs_htree {
	struct htree_node root;
	struct tee_fs_htree_image head;
	uint8_t fek[TEE_FS_HTREE_FEK_SIZE];
	struct tee_fs_htree_imeta imeta;
	bool dirty;
	const TEE_UUID *uuid;
	const struct tee_fs_htree_storage *stor;
	void *stor_aux;
};
```

对于创建动作，TEE首先生成一个随机数（通过`crypto_rng_read`从随机数池中读取），这个随机数就是上述提到的FEK，然后根据当前的TA UUID和SSK，使用hmac算法生成加密FEK的密钥TSK，然后用TSK加密FEK后存储加密后的FEK到哈希树节点的head.enc_fek中。这个步骤的详细实现在`tee_fs_fek_crypt`中。

在FEK生成后，就要对哈希树的根节点进行初始化。初始化的过程主要是计算根节点的哈希，计算的内容包括：root.node.iv、root.node.tag、root.node.flags和 imeta.meta，如果node.child[2]有内容，则会将这两个节点的哈希值归并计算根节点的哈希，但是在创建过程，这两个节点指针为空。计算得到的哈希值会存储到root.node.hash中。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312150958379.png)

### 3.3.2 哈希树读取

在读取哈希树时，会尝试对加密后的FEK进行解密，如果解密失败，那么可以认为哈希树被篡改了，这次访问就会禁止，从而达到不受非法篡改数据影响的目的，这个过程是在`tee_fs_fek_crypt`中完成。解密后的FEK会用来解密imeta。在读取的最后一步，需要校验哈希树，这一步骤在`verify_tree`中完成，校验哈希树的逻辑是采用后续遍历的方式，**逐一校验每个节点的哈希值与计算出来的哈希值是否一致，不一致则直接返回错误，不允许访问非法篡改的数据**。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312150959856.png)

```C
static TEE_Result verify_tree(struct tee_fs_htree *ht)
{
	TEE_Result res;
	void *ctx;

	res = crypto_hash_alloc_ctx(&ctx, TEE_FS_HTREE_HASH_ALG);
	if (res != TEE_SUCCESS)
		return res;

	res = htree_traverse_post_order(ht, verify_node, ctx);
	crypto_hash_free_ctx(ctx);

	return res;
}
```

**通过hash树的机制，TEE确保了Secure storage中的数据的完整性不被篡改，以及篡改后不使用**。

# 3. TEE的安全能力和FUSE的安全能力对比

两者理论上不是可以对比的对象， 因为各自负责的领域不同，FUSE一般用于一些不可变配置以及secure boot的公钥哈希存储等场景，TEE如其名，为系统提供可信的执行环境。但是，在数据的存储上，两者都有共同点：都能提供完整性数据的存储服务。

FUSE提供的完整性数据存储更多的是对于不可变数据的存储，但是由于FUSE的容量限制，一般只存储对应的哈希值，比如：公钥哈希、证书哈希等。但是有一点需要注意，FUSE属于是一次性刷写设备（OTP，One Time Programmable device）,因此对于需要更新的数据，不应存储在FUSE。

TEE的安全存储能力依赖于TrustZone技术，不仅可以提供完整性保护，同时提供了机密性数据的存储能力，相比于FUSE，TEE提供的完整性保护服务同时支持授权的更新行为。并且，**所有支持TrustZone技术的芯片，都能支持secure storage的能力，相比之下，使用FUSE是严格依赖于芯片的FUSE容量，不同芯片的FUSE容量不一致，甚至有些芯片没有FUSE**。

总结而言：

- FUSE可以提供硬件级别的完整性保护，但因硬件性质，不可更新且空间小
    
- TEE不仅可以提供完整性保护，还可以提供机密性保护。这个能力是由硬件机制TrustZone提供的，空间相比FUSE较大

# 4. 车上敏感数据

汽车控制器上涉及的敏感数据主要包括：
- 非对称密钥
- 对称密钥
- 证书
- 隐私数据
    
其中，不同的数据所需要的保护措施不一样：
- 非对称密钥：私钥要求机密性保护，公钥要求完整性保护
- 对称密钥：机密性保护
- 证书：完整性保护
- 隐私数据：机密性保护

由于TCU上没有硬件安全模块，对于机密性保护的数据，只能存储在TEE中，对于完整性保护的数据，可以存储在FUSE中，也可以存储在TEE中。但由于FUSE的容量有限，目前能用于OEM使用的只有二十字节，因此需要慎重考虑存储于FUSE的数据。


# 5. Reference

1. [https://optee.readthedocs.io/en/latest/general/index.html](https://optee.readthedocs.io/en/latest/general/index.html)
2. [https://static.linaro.org/connect/las16/Presentations/Friday/LAS16-504%20-%20Secure%20Storage%20updates%20in%20OP-TEE.pdf](https://static.linaro.org/connect/las16/Presentations/Friday/LAS16-504%20-%20Secure%20Storage%20updates%20in%20OP-TEE.pdf)
3. [https://github.com/OP-TEE/optee_os/blob/master/core/tee/fs_htree.c](https://github.com/OP-TEE/optee_os/blob/master/core/tee/fs_htree.c)
4. [https://www.arm.com/technologies/trustzone-for-cortex-a/tee-reference-documentation](https://www.arm.com/technologies/trustzone-for-cortex-a/tee-reference-documentation)
5. [https://optee.readthedocs.io/en/3.16.0/architecture/porting_guidelines.html#hardware-unique-key](https://optee.readthedocs.io/en/3.16.0/architecture/porting_guidelines.html#hardware-unique-key)