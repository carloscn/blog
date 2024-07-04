
在ls104x上，pkcs11是通过NXP开发的libsecure_obj来加载的。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202406130920688.png)

完成openssl engine pkcs11的支持，需要配置两个地方：
* OpenSC提供了libp11，这个开源库提供了一个higher-level层级的接口用于访问PKCS11的object。OpenSSL直接和这一层进行交互；
* pkcs11 engine plugin，是NXP提供的插件，是libp11调用的一层，由NXP开发；可以在NXP的sdk找到；

# OpenSC的libp11库交叉编译

交叉编译 libp11的库需要提前准备好，交叉编译的openssl的输出。

```bash
	rm -rf libp11
	git clone https://github.com/OpenSC/libp11
	cd libp11 && ./bootstrap && \
		./configure --host=aarch64-linux-gnu --prefix=$(PWD)/libp11/libp11-aarch64/ --with-enginesdir=$(PWD)/libp11/libp11-aarch64/ \
				OPENSSL_LIBS="-ldl -lpthread -lcrypto -L${OPENSSL_PATH}/lib" OPENSSL_CFLAGS="-g -O2 -I ${OPENSSL_PATH}/include" \
				CC=${OPENWRT_DIR}/staging_dir/toolchain-aarch64_generic_gcc-11.2.0_glibc/bin/aarch64-openwrt-linux-gnu-gcc \
				CXX=${OPENWRT_DIR}/staging_dir/toolchain-aarch64_generic_gcc-11.2.0_glibc/bin/aarch64-openwrt-linux-gnu-g++ \
				LD=${OPENWRT_DIR}/staging_dir/toolchain-aarch64_generic_gcc-11.2.0_glibc/bin/aarch64-openwrt-linux-gnu-ld
	cd libp11 && make -j8 && make install
```


![](https://raw.githubusercontent.com/carloscn/images/main/typora202406130926586.png)

编译完成之后，把这些so文件放入以下： （假设sec_rootfs是你的rootfs根目录）

```bash
mkdir -p sec_rootfs/usr/share/lib
cp -rfv ${PWD}/libp11/libp11-aarch64/lib/* sec_rootfs/usr/share/lib/
cp -rfv ${PWD}/libp11/libp11-aarch64/*.so sec_rootfs/usr/lib/engines-1.1/pkcs11.so
```

![](https://raw.githubusercontent.com/carloscn/images/main/typora202406130928046.png)

# 目标硬件OpenSSL配置

`vim /etc/ssl/openssl.cnf`

```
[openssl_init] 
engines=engine_section 

[engine_section] 
pkcs11 = pkcs11_section 

[pkcs11_section] 
engine_id = pkcs11 
dynamic_path = /usr/lib/engines-1.1/libpkcs11.so
MODULE_PATH = /usr/lib/libpkcs11.so
init = 0
```

使用`openssl engine pkcs11 -t` 测试：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202406131713172.png)

之后我们就可以正常使用engine pkcs11了。

# 2. 签名工具

https://github.com/carloscn/ls104x-bsp/tree/master/sign_cert

## 2.1 启动tee-supplicant

由于NXP的实现的HSM基于TEE环境，因此需要启动`tee-supplicant`

## 2.2 生成私钥在HSM环境

`sobj_app -C -f rsa_key_2048.pem -k rsa -o pair -s 2048 -l "tcu_key" -i 0`

查看私钥url：

`p11tool --provider /usr/lib/libpkcs11.so --list-privkeys`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202406141718373.png)

在这里可以copy URL作为tcu sign工具入参。

## 2.3 初始化生成rootca和公钥

`./tcu_sign_tool.elf --init -u "pkcs11:model=PKCS11-OP-TEE;manufacturer=NXP;serial=1;token=tcu_key;id=%00%00%00%00;object=tcu_key;type=private"`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202406141719556.png)

## 2.4 假设生成某个csr

为了测试签csr的功能，我们这里假设手机生成的csr，所以我们需要使用openssl生成私钥和csr。

`openssl genpkey -algorithm RSA -out xtaxi_app_private.pem -pkeyopt rsa_keygen_bits:2048`

`openssl req -new -key xtaxi_app_private.pem -out xtaxi_app.csr`

## 2.5 签csr

`./tcu_sign_tool.elf --sign -u "pkcs11:model=PKCS11-OP-TEE;manufacturer=NXP;serial=1;token=tcu_key;id=%00%00%00%00;object=tcu_key;type=private" -f xtaxi_app.csr -o xtaxi_app_cert.pem`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202406141719490.png)

## 2.6 验证签发的证书

`./tcu_sign_tool.elf --verify -u "pkcs11:model=PKCS11-OP-TEE;manufacturer=NXP;serial=1;token=tcu_key;id=%00%00%00%00;object=tcu_key;type=private" -f xtaxi_app_cert.pem`

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202406141719935.png)

## 2.7 清除rootca和公钥

`./tcu_sign_tool.elf --clean`
