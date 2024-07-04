
OpenSSL offload 指的是将 OpenSSL 加密和解密任务从 CPU 转移到专用硬件上执行的过程。OpenSSL 是一个广泛使用的开源加密库，支持多种加密算法，用于保护网络通信的安全。在处理大量加密解密操作时，这些任务可能会消耗大量的 CPU 资源，影响系统的其他功能或增加服务器的负载。

通过使用加密卸载技术，例如使用支持硬件加速的专用加密卡或集成加速硬件的处理器，可以显著提高加密和解密操作的效率。这种方法可以减少 CPU 的负担，提高整体系统性能，尤其是在需要处理高强度加密操作的高性能计算环境或大规模数据中心中。

硬件加速器可以是专用的安全处理器、网络卡（提供 SSL/TLS 加速功能）、或者是集成了加密加速功能的 CPU。这些硬件通常设计有专门的算法（如 AES、RSA、SHA）加速引擎，能够在不影响主 CPU 性能的情况下，快速处理加密和解密任务。

OpenSSL offload 通过硬件来实现安全算法的加速，对于那些需要处理大量安全通信，同时希望优化资源使用和提高性能的场景特别有用。

本文以NXP的Layerscape为例子，来描述OpenSSL的offload的功能及NXP如何在Linux中和自己CAAM Driver进行结合。

#  OpenSSL软件栈

OpenSSL可以通过其引擎接口将加密操作委托给各种硬件设备执行。建立在这个接口之上的是cryptodev引擎，它用于将加密操作卸载到由操作系统内核控制的硬件设备上。Cryptodev引擎最初为OpenBSD开发，后来通过多个驱动程序（如OCF和cryptodev-linux）将相同的API移植到了GNU/Linux操作系统上。

Cryptodev-linux是一个Linux内核驱动程序，通过设备文件/dev/crypto将内部加密API暴露给用户空间。用户空间应用通过ioctl系统调用请求Linux内核代表它们执行加密操作。Linux内核支持多种加密算法，并在CPU上运行软件实现。硬件加速器的驱动程序安装时具有更高的优先级，并且无需进一步配置即可覆盖软件实现。

从任何应用程序的角度来看，算法的最快实现是透明使用的。这种行为也传递给了cryptodev接口，该接口不知道算法是在CPU上运行还是在硬件加速器上运行。因此，确保在运行应用程序之前有可用的硬件内核驱动程序是应用操作员的责任。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202403191436558.png)

# NXP的方案

The following layers can be observed in NXP's solution for OpenSSL hardware offloading:
* 用户层：OpenSSL (user space): implements the SSL protocol
* 用户层驱动映射：cryptodev-engine (user space): implements the OpenSSL ENGINE interface; talks to cryptodev-linux (/dev/crypto) through ioctls, offloading cryptographic operations in the kernel
* **cryptodev-linux (kernel space)**: Linux module that translates ioctl requests from cryptodev-engine into calls to Linux Crypto API
* **AF_ALG** is a netlink-based in the kernel asynchronous interfac
* **Linux Crypto API (kernel space):** Linux kernel crypto abstraction layer
* **CAAM driver (kernel space)**: Linux device driver for the CAAM crypto engine

## 编译cryptodev-engine

在内核环境中这样的编译：

`git clone git@github.com:cryptodev-linux/cryptodev-linux.git`

```bash
pushd cryptodev-linux

CROSS_COMPILE=${CROSS_COMPILE} make -C ${OPENWRT_LINUX_DIR} M=${PWD} modules

popd
```

如果是openWRT环境的话，有一点点不同。因为没有办法去直接编译内核。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202403191554036.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202403191554416.png)

请先编译内核：

`make target/linux/compile V=s -j32`

可以使用openwrt单独编译：

`make package/kernel/cryptodev-linux/compile V=s -j32`

现在已经有了：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202403201433623.png)

```log
root@imx8qxpmek:~# modprobe cryptodev
[17044.896494] cryptodev: driver 1.10 loaded.
root@imx8qxpmek:~# openssl engine
(devcrypto) /dev/crypto engine
(dynamic) Dynamic engine loading support
root@imx8qxpmek:~#
```

现在还有点问题：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202403201434717.png)

```log
root@OpenWrt:/# openssl engine
(dynamic) Dynamic engine loading support
(devcrypto) /dev/crypto engine
281473283568672:error:2506406A:DSO support routines:dlfcn_bind_func:could not bind to the requested symbol name:crypto/dso/dso_dlfcn.c:188:symname(EVP_PKEY_get_base_id): /usr/lib/engines-1.1/devcrypto.so: undefined symbol: EVP_PKEY_get_base_id
281473283568672:error:2506C06A:DSO support routines:DSO_bind_func:could not bind to the requested symbol name:crypto/dso/dso_lib.c:186:
```

总的来说：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202403281443506.png)

测试：`openssl speed -engine devcrypto -multi 8 -elapsed -evp aes-128-cbc`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202403281444966.png)

