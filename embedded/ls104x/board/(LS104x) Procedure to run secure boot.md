# 1. Overview
This section describes the steps to run secure boot on NXP Layerscape family SOC-based boards:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312191531672.png)

**The procedure to run the secure boot is shown in the following steps:**

1. Prepare board for secure boot
    1. [Enable POVDD](https://docs.nxp.com/bundle/GUID-487B2E69-BB19-42CB-AC38-7EF18C0FE3AE/page/GUID-0EFF85FB-9070-4D76-A926-B973CA6C8FB5.html#GUID-0EFF85FB-9070-4D76-A926-B973CA6C8FB5)
    2. [Program OTPMK](https://docs.nxp.com/bundle/GUID-487B2E69-BB19-42CB-AC38-7EF18C0FE3AE/page/GUID-F780D1D5-F1B2-478B-86AE-267D74F9C790.html#GUID-F780D1D5-F1B2-478B-86AE-267D74F9C790)
    3. [Program SRK](https://docs.nxp.com/bundle/GUID-487B2E69-BB19-42CB-AC38-7EF18C0FE3AE/page/GUID-D8CC0ABA-108D-4966-8C01-F584B897B9E2.html#GUID-D8CC0ABA-108D-4966-8C01-F584B897B9E2)
2. Build secure boot images for NXP CoT and Arm CoT
    - [using flex-builder](https://docs.nxp.com/bundle/GUID-487B2E69-BB19-42CB-AC38-7EF18C0FE3AE/page/GUID-2AE4E770-FCC5-4376-93CE-D69F3A3BF9DF.html#GUID-2AE4E770-FCC5-4376-93CE-D69F3A3BF9DF), or
    - [manually (using TF-A)](https://docs.nxp.com/bundle/GUID-487B2E69-BB19-42CB-AC38-7EF18C0FE3AE/page/GUID-07074F71-3F5C-484B-9965-93D919936CB0.html#GUID-07074F71-3F5C-484B-9965-93D919936CB0)
3. [Program secure boot images for NXP CoT and Arm CoT](https://docs.nxp.com/bundle/GUID-487B2E69-BB19-42CB-AC38-7EF18C0FE3AE/page/GUID-11BC1A68-D413-4105-9469-0020BAAE1E23.html#GUID-11BC1A68-D413-4105-9469-0020BAAE1E23)
4. [Steps to run chain of trust with confidentiality](https://docs.nxp.com/bundle/GUID-487B2E69-BB19-42CB-AC38-7EF18C0FE3AE/page/GUID-7EE8D47D-42C5-4D37-A94B-A75A73148A30.html#GUID-7EE8D47D-42C5-4D37-A94B-A75A73148A30)

Note, **Red** blocks are dangerous, and can't take it back.

# 2. Enabling Secure Boot

## 2.1 Enable POVDD

The use of the trust architecture feature is dependent on programming fuses in the Security Fuse Processor (SFP). To program SFP fuses, the user is required to **supply 1.8 V to the TA_PROG_SFP pin** Power sequencing. TA_PROG_SFP should only be powered for the duration of the fuse programming cycle, with a per-device limit of six fuse programming cycles. At all other times, TA_PROG_SFP should be connected to GND. The TA_PROG_SFP pin is shown in the following figure:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312191532244.png)

This figure shows the TA_PROG_SFP timing diagram:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312191532427.png)

## 2.2 RCW modifying

The reset configuration word (RCW) resides in non-volatile memories (for example, NOR, QSPI, SDHC). It gives the flexibility to accommodate a large number of configuration parameters to support a high degree of configurability of the SoC. Configuration parameters generally include:
- Frequencies of various blocks including cores/DDR/interconnect.
- IP pin-muxing configurations
- Other SoC configurations

The RCW's provided with the release enable the following features:
- Boot location as NOR flash
- Enables 4 UART without flow control
- Enables I2C1, I2C2, I2C3, I2C4, SDHC, IFC, PCIe, SATA

### How to change RCW?

RCW[SB_EN] = 1 can be set by the following figure:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312191533364.png)

Build the new RCW file:

$ `flex-builder -i clean-firmware`
$ `flex-builder -c firmware -a arm64 -m ls1046ardb -b sd -S 1040`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312191533519.png)

### Cat RCW file

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312191534679.png)

In the [LS1046A Reference Manual](https://www.nxp.com/webapp/Download?colCode=LS1046ARM) - 4.4.6.1 RCW Field Definitions, the offset of the SB_EN is 202 (0xCA) bits.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312191534694.png)

The position of the SB_EN is fixed in the following figure:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312191534034.png)

## 2.3 Blown eFUSE

### 2.3.1 Provisioning Ordering

**SRKH and OTPMK should be carefully written keeping in mind the SFP Block Endianness. If SRKH and OTPMK are written using Core, SRKH and OTPMK need to be swapped. However, if SRKH and OTPMK are written using DAP or SFP, swap is not required. Refer the following table for details.**

|   |   |   |   |   |
|---|---|---|---|---|
|**Console**|**SRKH/OTPMK generated order from CST**|**SRKH/OTPMK write order**|**SRKH/OTPMK Read order**|**Endianness**|
|U-Boot|order_1|**reverse order_1**|**reverse order_1**|Core endianness|
|CCS|order_1|order_1|order_1|SFP endianness (DAP)|

**Assuming following SRKH values are generated:**

```
SRK (Public Key) Hash:
fdc2fed4317f569e1828425ce87b5cfd34beab8fdf792a702dff85e132a29687
               SFP SRKHR0 = fdc2fed4
               SFP SRKHR1 = 317f569e
               SFP SRKHR2 = 1828425c
               SFP SRKHR3 = e87b5cfd
               SFP SRKHR4 = 34beab8f
               SFP SRKHR5 = df792a70
               SFP SRKHR6 = 2dff85e1
               SFP SRKHR7 = 32a29687
```

**Execute the following commands at the CCS console to permanently write SRKH using DAP/SFP:**

```
ccs::write_mem 32 0x1e80254 4 0 0xfdc2fed4
ccs::write_mem 32 0x1e80258 4 0 0x317f569e
ccs::write_mem 32 0x1e8025c 4 0 0x1828425c
ccs::write_mem 32 0x1e80260 4 0 0xe87b5cfd
ccs::write_mem 32 0x1e80264 4 0 0x34beab8f
ccs::write_mem 32 0x1e80268 4 0 0xdf792a70
ccs::write_mem 32 0x1e8026c 4 0 0x2dff85e1
ccs::write_mem 32 0x1e80270 4 0 0x32a29687
```

**Execute the following commands at the U-Boot console to permanently write SRKH using core:**

```
mw.l 0x1e80254 0xd4fec2fd
mw.l 0x1e80258 0x9e567f31
mw.l 0x1e8025c 0x5c422818
mw.l 0x1e80260 0xfd5c7be8
mw.l 0x1e80264 0x8fabbe34
mw.l 0x1e80268 0x702a79df
mw.l 0x1e8026c 0xe185ff2d
mw.l 0x1e80270 0x8796a232
mw.l 0x1e80020 0x2       
```

### 2.3.2 Program OTPMK

After [enabling POVDD](https://docs.nxp.com/bundle/GUID-487B2E69-BB19-42CB-AC38-7EF18C0FE3AE/page/GUID-0EFF85FB-9070-4D76-A926-B973CA6C8FB5.html#GUID-0EFF85FB-9070-4D76-A926-B973CA6C8FB5), follow these steps to program OTPMK at **U-Boot**:

#### Verify

Verify the SNVS register - HPSR to check whether OTPMK is fused already.

```
=> md $SNVS_HPSR_REG
88000900
```

OTPMK_ZERO_BIT (second nibble) is 1, indicating that OTPMK is not fused. The second nibble of 8**8**000900 is 1 indicatesthe OTPMK is not fused.

#### Gen OTPMK

Generate OTPMK by CST ./gen_otpmk_drbg -b 2

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312191535504.png)

#### Fuse OTPMK

**Note, write order !!!**

```
=> mw.l $OTPMKR0 0x125ce2c7
=> mw.l $OTPMKR1 0xdf730728
=> mw.l $OTPMKR2 0x5ef92af6
=> mw.l $OTPMKR3 0xf6dafed7
=> mw.l $OTPMKR4 ...
=> mw.l $OTPMKR5 ...
=> mw.l $OTPMKR6 ...
=> mw.l $OTPMKR7 ...
```

#### Check OTPMK

At the U-Boot prompt, verify that the **SNVS registers** for OTPMK are correctly written.

Check if OPTMK is fused.

```
=> md $SNVS_HPSR_REG
80000900
```

OTPMK_ZERO_BIT (second nibble) is 0, indicating that OTPMK is fused.

Read OTPMK:

```
=> md $OTPMKR0 0x10
01e80234: ffffffff ffffffff ffffffff ffffffff    ................
01e80244: ffffffff ffffffff ffffffff ffffffff    ................
```

**Note:** OTPMK is not visible in plain.

### 2.3.3 Program SRKH mirror registers

After [enabling POVDD](https://docs.nxp.com/bundle/GUID-487B2E69-BB19-42CB-AC38-7EF18C0FE3AE/page/GUID-0EFF85FB-9070-4D76-A926-B973CA6C8FB5.html#GUID-0EFF85FB-9070-4D76-A926-B973CA6C8FB5), follow these steps to program SRKH registers at U-Boot:

#### verify

Check if SRKH is fused.

```
=> md $SRKHR0 0x10 
01e80254: 00000000 00000000 00000000 00000000 ................ 
01e80264: 00000000 00000000 00000000 00000000 ................
```

Zero indicates that SRKH is not fused.

#### Fuse SRKH

The CST tool will generate the SRK by inputting public key.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312191535736.png)

**SRKH should be carefully written considering the SFP block endianness.**

```
=> mw.l $SRKHR0 0x82d05d68
=> mw.l $SRKHR1 <SRKHR_1_32Bit_val> 
=> mw.l $SRKHR2 <SRKHR_2_32Bit_val> 
=> mw.l $SRKHR3 <SRKHR_3_32Bit_val> 
=> mw.l $SRKHR4 <SRKHR_4_32Bit_val> 
=> mw.l $SRKHR5 <SRKHR_5_32Bit_val> 
=> mw.l $SRKHR6 <SRKHR_6_32Bit_val> 
=> mw.l $SRKHR7 <SRKHR_7_32Bit_val>
```

#### Check

Then, the following could be the value of dumping SRKH.

```
=> md $SRKHR0 0x10 
```

SRKH is visible in plain because of the SFP block endianness.

### 2.3.3 Program SFP_INGR register to LOCK FUSE

**_CAUTION:_ Do not proceed to the steps in this topic, until you are sure that OTPMK and SRKH are correctly fused, as explained in the topics above. After the next step, fuses are burnt permanently, which cannot be undone.**

- Write SFP_INGR[INST] with the PROGFB(0x2) instruction to blow the fuses.
    
```
=> mw $SFP_INGR_REG  $SFP_WRITE_DATA_FRM_MIRROR_REG_TO_FUSE
```

- Reset the board.

## 2.4 Gen Flow Overview

The log of the ls1043a signing images is shown in the following file:

And we can cat the progress of the ls1043a signing is shown in the following figure:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312191536854.png)

According to above figure, tools that we should port are shown as follows:
- gen_keys
- srk_hash_gen
- sign

### 2.4.1 gen_keys

The gen_keys.c source code is in the following figure:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312191536351.png)

change it to:
- copy the `"autox_pub_key.pem"` public key to srk.pub.
- copy the mocking `"autox_pri_key.pem"` private key to srk.pri.

Note, please put your generated `autox_pub_key.pem` and `autox_pri_key.pem` keys together with `gen_keys` tool in the same directory.

#### How to gen the key pairs

```
# !/bin/bash

rm -rf autox_pri_key.pem autox_pub_key.pem
openssl genrsa -out autox_pri_key.pem 1024
openssl rsa -in autox_pri_key.pem -out autox_pub_key_pkcs8.pem -pubout
openssl rsa -pubin -in autox_pub_key_pkcs8.pem -RSAPublicKey_out > autox_pub_key.pem
cat autox_pri_key.pem
cat autox_pub_key.pem
```

Note, the NXP cst uses the PKCS#1 format public key.

## 2.5 Image Signing

According to the NXP CST tool, the RSA (PKCS#1 format) is supported by CST. Please refer to the link [https://autoxai.atlassian.net/wiki/spaces/SEC/pages/2006257263/layerscape+NXP-LS1046+secure+boot#4.-Code-Signing-Tool](https://autoxai.atlassian.net/wiki/spaces/SEC/pages/2006257263/layerscape+NXP-LS1046+secure+boot#4.-Code-Signing-Tool) .

One image signing process is shown in the following figure:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312191536272.png)

### 2.5.1 uni_sign

**uni_sign** tool can be used for the mode that the CSF header generation without signature if a private key is not provided.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312191537375.png)

Meanwhile, use the --img_hash argument when executing the uni_sign tool. (following figure)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312191537303.png)

The image’s hash.out in a binary file, the public key's SRK hash in the console, and the CSF header in a binary file are outputted by the uni_sign after ./uni_sign --img_hash input_ppa_secure executed.

### 2.5.2 Sign Server signing the hash

The image’s hash.out in a binary file shall be signed by the signing server or the HSM so that the private key cannot be exported. The HSM signing process should aligned with the C programming:

```c
int crypto_rsa_sign(void *img_hash, uint32_t len, void *rsa_sign,
                    uint32_t *rsa_len, char *key_name)
{
    int ret;
    FILE *fpriv;
    RSA *priv_key;

    /* Open the private Key */
    fpriv = fopen(key_name, "r");
    if (fpriv == NULL) {
        printf("Error in file opening %s:\n", key_name);
        return -1;
    }

    priv_key = PEM_read_RSAPrivateKey(fpriv, NULL, NULL, NULL);
    fclose(fpriv);
    if (priv_key == NULL) {
        printf("Error in key reading %s:\n", key_name);
        return -1;
    }
    /* Sign the Image Hash with Private Key */
    ret = RSA_sign(NID_sha256, img_hash, len,
            rsa_sign, rsa_len,
            priv_key);
    if (ret != 1) {
        printf("Error in Signing\n");
        return -1;
    }

    return 0;
}
```

The API RSA_Sign in the OpenSSL is invoked by the signing process. [https://www.openssl.org/docs/man3.0/man3/RSA_sign.html](https://www.openssl.org/docs/man3.0/man3/RSA_sign.html).

Please distinguish between digest-RSA signing and RSA signing. The RSA_sign only use the RSA algo to sign the data and did not calculate the input data hash value, even if you specified the hash type NID_sha256.

We should use to sign the data.

To perform a signature on a hash using the the `openssl rsautl -raw` tool, the hash file (named payload) needs to be padded.

`openssl rsautl -raw -sign -inkey autox_pri_key.pem -in dec_sign.bin > enc_sign.bin`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312191537336.png)

Then, the following command needs to be used for verification:

`openssl rsautl -raw -verify -in signed_data.bin -inkey autox_pub_key_pkcs8.pem -pubin > dec_sign.bin`

**Note,**
- The public key file shall be the PKCS#8 format.
- The output file `dec_sign.bin` with the padding binary data, the data shall be stripped.

https://github.com/carloscn/ls104x-bsp/tree/master/bsp/unit_test/secboot_sign

The above compressed package contains test data, which can be used for command testing.

The sign server needs to port the function of OpenSSL raw signing `openssl rsautl -raw -sign -inkey autox_pri_key.pem -in dec_sign.bin > enc_sign.bin`

