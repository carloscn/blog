# 04_OPTEE-OS_系统集成之（二）基于QEMU的OPTEE启动过程

OPTEE的整个工程编译出来的结果在out目录：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20221003123730.png)

我们可以看到各个二进制文件都是谁生成的：

* linux & rootfs：
	1. zImage
	2. Image
	3. rootfs
* ATF:
	1. bl1.bin
	2. bl31.bin
	3. bl2.bin
* OPTEE:
	1. bl32.bin (tee-header_v2.bin)
	2. bl32_extra2.bin (tee-pageable_v2.bin)
	3. bl32_extra1.bin (tee-pager_v2.bin)

对于这些启动文件的打包，每一个芯片厂都采用了不同的方式：

# 1. bios阶段

## QEMU方式

我们从OPTEE-OS的`run-only`启动QEMU的过程就能看到，QEMU运行OPTEE需要哪些依赖。

```makefile
run-only:
	ln -sf $(ROOT)/out-br/images/rootfs.cpio.gz $(BINARIES_PATH)/
	$(call check-terminal)
	$(call run-help)
	$(call launch-terminal,54320,"Normal World")
	$(call launch-terminal,54321,"Secure World")
	$(call wait-for-ports,54320,54321)
	cd $(BINARIES_PATH) && $(QEMU_BUILD)/aarch64-softmmu/qemu-system-aarch64 \
		-nographic \
		-serial tcp:localhost:54320 -serial tcp:localhost:54321 \
		-smp $(QEMU_SMP) \
		-s -S -machine virt,secure=on,mte=$(QEMU_MTE),gic-version=$(QEMU_GIC_VERSION),virtualization=$(QEMU_VIRT) \
		-cpu $(QEMU_CPU) \
		-d unimp -semihosting-config enable=on,target=native \
		-m $(QEMU_MEM) \
		-bios bl1.bin		\
		-initrd rootfs.cpio.gz \
		-kernel Image -no-acpi \
		-append 'console=ttyAMA0,38400 keep_bootcon root=/dev/vda2 $(QEMU_KERNEL_BOOTARGS)' \
		$(QEMU_XEN) \
		$(QEMU_EXTRA_ARGS)
```

我们可以根据ATF的启动流程分析qemu先加载bl1.bin，也就是根据-bios指定的参数，启动过如图：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221003125326.png)

接着就开始启动Linux内核：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221003125558.png)

接着Linux内核挂载rootfs。

## NXP方式
例如nxp使用mkimage的方式把所有的tee.bin, blx.bin, linux image等等这些都包装成一个文件：
```Makefile
mkimage: u-boot tfa ddr-firmware
	ln -sf $(ROOT)/optee_os/out/arm/core/tee-raw.bin \
		$(MKIMAGE_PATH)/iMX8M/tee.bin
	ln -sf $(ROOT)/trusted-firmware-a/build/imx8mq/release/bl31.bin \
		$(MKIMAGE_PATH)/iMX8M/
	ln -sf $(LPDDR_BIN_PATH)/lpddr4_pmu_train_*.bin $(MKIMAGE_PATH)/iMX8M/
	ln -sf $(U-BOOT_PATH)/u-boot-nodtb.bin $(MKIMAGE_PATH)/iMX8M/
	ln -sf $(U-BOOT_PATH)/spl/u-boot-spl.bin $(MKIMAGE_PATH)/iMX8M/
	ln -sf $(U-BOOT_PATH)/arch/arm/dts/imx8mq-evk.dtb \
		$(MKIMAGE_PATH)/iMX8M/fsl-imx8mq-evk.dtb
	ln -sf $(U-BOOT_PATH)/tools/mkimage $(MKIMAGE_PATH)/iMX8M/mkimage_uboot
	$(MAKE) -C $(MKIMAGE_PATH) SOC=iMX8M flash_spl_uboot
```

还有在ARMv7的一些架构里面，也是把这些二进制文件都打包成一个集合bios.bin，然后在入口函数中，对这些二进制文件进行解析和重定位及跳转。这部分处理是非常灵活的。

## ARMv7方式

使用QEMU运行OP-TEE时首先加载的是编译 生成的bios.bin镜像文件，而bios.bin镜像文件的入口函数是在`bios_qemu_tz_arm/bios/entry.S`文件中定义的，该文件的入口函数为`_start`，该文件的主要内容如下：

https://github.com/jenswi-linaro/bios_qemu_tz_arm

```assembly
#include <platform_config.h>

#include <asm.S>
#include <arm32.h>
#include <arm32_macros.S>

.section .text.boot
FUNC _start , :
	b	reset
	b	.	/* Undef */
	b	.	/* Syscall */
	b	.	/* Prefetch abort */
	b	.	/* Data abort */
	b	.	/* Reserved */
	b	.	/* IRQ */
	b	.	/* FIQ */
END_FUNC _start

/*
 * Binary is linked against BIOS_RAM_START, but starts to execute from
 * address 0. The branching etc before relocation works because the
 * assembly code is only using relative addressing.
 */
LOCAL_FUNC reset , :
	read_sctlr r0
	orr	r0, r0, #SCTLR_A
	write_sctlr r0

	/* Setup vector */
	adr	r0, _start
	write_vbar r0

	/* Relocate bios to RAM */
	mov	r0, #0
	ldr	r1, =__text_start
	ldr	r2, =__data_end
	sub	r2, r2, r1
	bl	copy_blob

	/* Jump to new location in RAM */
	ldr	ip, =new_loc
	bx	ip
new_loc:

	/* Setup vector again, now to the new location */
	adr	r0, _start
	write_vbar r0

	/* Zero bss */
	ldr	r0, =__bss_start
	ldr	r1, =__bss_end
	sub	r1, r1, r0
	bl	zero_mem

	/* Setup stack */
	ldr	ip, =main_stack_top;
	ldr	sp, [ip]

	push	{r0, r1, r2}
	mov	r0, sp
	ldr	ip, =main_init_sec
	blx	ip
	pop	{r0, r1, r2}
	mov	ip, r0	/* entry address */
	mov	r0, r1	/* argument (address of pagable part if != 0) */
	blx	ip

	/*
	 * Setup stack again as we're in non-secure mode now and have
	 * new registers.
	 */
	ldr	ip, =main_stack_top;
	ldr	sp, [ip]

	ldr	ip, =main_init_ns
	bx	ip
END_FUNC reset

LOCAL_FUNC copy_blob , :
	ldrb	r4, [r0], #1
	strb	r4, [r1], #1
	subs	r2, r2, #1
	bne	copy_blob
	bx	lr
END_FUNC copy_blob

LOCAL_FUNC zero_mem , :
	cmp	r1, #0
	bxeq	lr
	mov	r4, #0
	strb	r4, [r0], #1
	sub	r1, r1, #1
	b	zero_mem
END_FUNC zero_mem
```

main_init_sec函数用来将Linux内核镜像、OP- TEE OS镜像、rootfs镜像文件加载到RAM的对应位 置，并且解析出OP-TEE OS的入口地址、Linux内核的加载地址、rootfs在RAM中的地址和其他相关 信息。main_init_sec函数执行完成后会返回OP-TEE OS的入口地址以及设备树（device tree，DT）的地 址，然后在汇编代码中通过调用blx指令进入OP- TEE OS的启动。

OP-TEE启动完成后会重新进入entry.S文件中继续执行，最终执行main_init_ns函数来启动Linux 内核，在`main_init_sec`函数中会设定Linux内核的入口函数地址、DT的相关信息，`main_init_ns`函数会使用这些信息来开始Linux内核的加载。

上述两个函数都定义在`bios_qemu_tz_arm/bios/main.c`文件中。将各种镜像 文件复制到RAM的操作都是通过解析bios.bin镜像的对应section来实现的，通过寻找特定的section来确定各镜像文件在bios.bin文件中的位置。

启动过程中entry.S文件通过汇编调用main_init_sec函数将optee-os镜像、Linux镜像和rootfs加载到RAM中，并定位DT的地址信息，以备Linux和OP-TEE启动使用，这些操作是由main_init_sec函数进行的，该函数定义在 bios_qemu_tz_arm/bios/main.c文件中，其内容如下：

```C
void main_init_sec(struct sec_entry_arg *arg) 
{ 
    void *fdt; 
    int r; 
    //定义OP-TEE OS 镜像文件存放的起始地址 
    const uint8_t *sblob_start = &__linker_secure_blob_start; 
    //定义OP-TEE OS 镜像文件存放的末端地址 
    const uint8_t *sblob_end = &__linker_secure_blob_end; 
    struct optee_header hdr; //存放OP-TEE OS image头的信息 
    size_t pg_part_size; //OP-TEE OS image除去初始化头部信息的大小 
    uint32_t pg_part_dst; //OP-TEE OS image除去初始化头部信息后在RAM中的起始地址 
    
    msg_init(); //初始化uart 
    
    /* 加载device tree 信息。在qemu工程中,并没有将device tree信息编译到Bios.bin中,而默认存放在DTB_START地址中 */ 
    fdt = open_fdt(DTB_START, &__linker_nsec_dtb_start, &__linker_nsec_dtb_end); 
    r = fdt_pack(fdt); 
    CHECK(r < 0); 
    
    /* 判定OP-TEE OS image的大小是否大于image header的大小 */ 
    CHECK(((intptr_t)sblob_end - (intptr_t)sblob_start) < 
    (ssize_t)sizeof(hdr)); 
    
    /* 将OP-TEE OS image header信息复制到hdr变量中 */ 
    copy_bios_image("secure header", (uint32_t)&hdr, sblob_start, 
    sblob_start + sizeof(hdr)); 
    
    /* 校验OP-TEE OS image header中的magic和版本信息是否合法 */ 
    CHECK(hdr.magic != OPTEE_MAGIC || hdr.version != OPTEE_VERSION);msg("found secure header\n"); 
    sblob_start += sizeof(hdr); //将sblob_start的值后移到除去image header的位置 
    CHECK(hdr.init_load_addr_hi != 0); //检查OP-TEE OS的初始化加载地址是否为零 
    
    /* 获取OP-TEE OS除去 image header和ini操作部分代码后的大小 */ 
    pg_part_size = sblob_end - sblob_start - hdr.init_size; 
    
    /* 确定存放OP-TEE OS除去image header和init操作部分代码后存放在RAM中的地址 */ 
    pg_part_dst = (size_t)TZ_RES_MEM_START + TZ_RES_MEM_SIZE - pg_part_size; 
    
    /* 将存放OP-TEE OS除去image header和init操作部分后的内容复制到RAM中 */ 
    copy_bios_image("secure paged part", 
    pg_part_dst, sblob_start + hdr.init_size, sblob_end); 
    sblob_end -= pg_part_size; //重新计算sblo_end的地址,剔除page part 
    
    //将pg_part_dst赋值给arg中的paged_part以备跳转执行OP-TEE OS使用 
    arg->paged_part = pg_part_dst; 
    
    //将hdr.init_load_addr_lo赋值给arg中的entry,该地址为op-TEE OS的入口地址 
    arg->entry = hdr.init_load_addr_lo; 
    /* 将OP-TEE OS的实际image复制到起始地址为hdr.init_load_addr_l的RAM地址中 */ 
    copy_bios_image("secure blob", hdr.init_load_addr_lo, sblob_start, sblob_end); 
    
    //复制kernel image、rootfs到RAM,并复制device tree到对应地址,以备被kernel使用 
    copy_ns_images(); 
    
    /* 将device tree的地址赋值给arg->fdt变量,以备OP-TEE OS启动使用 */ 
    arg->fdt = dtb_addr; 
    msg("Initializing secure world\n"); 
} 
```

main_init_sec函数执行后将会返回一个`sec_entry_arg`的变量，该变量包含启动OP-TEE OS的入口地址、DT的地址以及`paged_table`的地址。 `sec_entry_arg`变量将会被entry.S文件用来启动OP- TEE OS，entry.S会将OP-TEE OS的入口地址保存在r0寄存器中，而`paged_table`部分的起始地址会被保 存在r1寄存器中，将r0赋值给ip，最终entry.S文件 通过执行blx ip指令进入OP-TEE OS的入口函数中去 执行OP-TEE OS的启动。当OP-TEE OS启动完成之后，entry.S文件会调用`main_init_ns`函数来启动Linux内核。待Linux内核启动完成之后，整个系统也就启动完成。

entry.S文件通过调用`main_init_ns`函数来完成对 Linux内核的启动，该函数会调用`call_kernel`函数来完成Linux内核的启动。

# 2. OPTEE驱动启动

在OP-TEE工程中，OP-TEE在REE侧的驱动会被编译到Linux内核镜像中，Linux系统在启动的过程中会自动挂载OP-TEE的驱动，驱动挂载过程中会创建`/dev/tee0`和`/dev/teepriv0`设备，其中`/dev/tee0 `设备将会被REE侧的用户空间的库（libteec）使用，`/dev/teepriv0`设备将会被系统中的常驻进程`tee_supplicant`使用，并且在OP-TEE驱动的挂载过程中会建立正常世界状态与安全世界状态之间的共享内存，用于OP-TEE驱动与OP-TEE之间的数据共享，同时还会创建两个链表，分别用于保存来自OP-TEE的RPC请求和发送RPC请求的处理结果给OP-TEE。

# 3. tee_supplicant启动

`tee_supplicant`是Linux系统中的常驻进程，该进程用于接收和处理来自OP-TEE的RPC请求，并将处理结果返回给OP-TEE。**来自OP-TEE的RPC请求主要包括socket操作、REE侧文件系统操作、加载TA镜像文件、数据库操作、共享内存分配和注册操作等**。该进程在Linux系统启动过程中被自动创建，在编译时，该进程的启动信息会被写入到/etc/init.d文件中，而该进程的可执行文件则被保存在文件系统的bin目录下。该进程中会使用一个loop循环接收来自OP-TEE的远程过程调用（Remote Procedure Call，RPC）请求，且每次获取到来自OP-TEE的RPC请求后都会自动创建一个线程，用于接收OP-TEE驱动队列中来自OP-TEE的RPC请求，之所以这么做是因为时刻需要保证在REE侧有一个线程来接收OP-TEE的请求，实现RPC请求的并发处理。
