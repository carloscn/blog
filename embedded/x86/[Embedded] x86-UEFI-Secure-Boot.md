# [Embedded] x86-UEFI-Secure-Boot

依托系统固件BIOS，使用认证的固件以提高安全性。在2011年，NIST发布BIOS Protection Guidelines ([SP800-147](https://csrc.nist.gov/publications/detail/sp/800-147/final))，并在2018年，扩大概念从BIOS到platform，则发布Platform Firmware Resiliency Guidelines ([SP800-193](https://csrc.nist.gov/publications/detail/sp/800-193/final))。NIST的目的是提供一个针对固件完整性的通用的准则。

1987年，Clark-Wilson提出了一个完整性模型 [Clark-Wilson integrity policy](http://theory.stanford.edu/~ninghui/courses/Fall03/papers/clark_wilson.pdf)。 UEFI secure boot总的原则是遵循该模型的rules的。一些rules可以参考： https://edk2-docs.gitbook.io/understanding-the-uefi-secure-boot-chain/overview#integrity-models


# 1. UEFI Boot Chain

根据NIST-SP800-147和SP800-193，系统在引导流程期间需要保证完整性(integrity)和有效性（availability）。对于固件而言，secboot（a.k.a verified boot）在执行下一阶段引导流程之前需要执行一系列验证策略。

UEFI secure boot在UEFI手册中定义的一个特性。它来确保只有正确的第三方固件程序可以在OEM的固件环境中执行。UEFI secure boot假设系统固件是可信的，任何其他三方固件都是不可信的，其中包括，OSV（Operating System Vendor）提供的bootloader。作为管理验证策略的一部分，最终用户可以选择注册和撤销UEFI安全引导映像安全数据库中的入口。

UEFI secure boot分为两部分，一部分是boot image的验证，一部分是image安全数据库更新的验证。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221123204809.png)

| **Key** | **Verifies**                                                 | **Update is verified by** | **NOTES**                 |
| ------- | ------------------------------------------------------------ | ------------------------- | ------------------------- |
| PK      | New PK \ New KEK \ New db/dbx/dbt/dbr \ New OsRecoveryOrder \  New OsRecovery#### | PK                        | Platform Key              |
| KEK     | New db/dbx/dbt/dbr  New OsRecoveryOrder  New OsRecovery####  | PK                        | Key Exchange Key          |
| db      | UEFI Image                                                   | PK/KEK                    | Authorized Image Database |
| dbx     | UEFI Image                                                   | PK/KEK                    | Forbidden Image Database  |
| dbt     | UEFI Image + dbx                                             | PK/KEK                    | Timestamp Database        |
| dbr     | New OsRecoveryOrder  New OsRecovery####                      | PK/KEK                    | Recovery Database         |




