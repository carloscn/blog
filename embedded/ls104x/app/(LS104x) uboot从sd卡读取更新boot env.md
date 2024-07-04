## 目的

本文档旨在说明如何通过U-Boot Standalone 应用从SD卡读取配置文件（例如`boot.cfg`），解析该文件中的关键环境变量，并更新系统的启动参数。这种机制对于需要从外部介质动态更新启动配置的嵌入式系统特别有用，如在固件更新或启动时根据条件选择不同的启动流程。

```
                   +------------------+
                   |  Start           |
                   +------------------+
                           |
                           v
           +-------------------------------+
           |  S1: Load `boot.cfg` from SD  |
           +-------------------------------+
                           |
                           v
         +---------------------------------------+
         |  S2: Parse boot_flag, hash, rollback  |
         |      flag from loaded content         |
         +---------------------------------------+
                           |
                           v
          +------------------------------------+
          |  S3: Validate boot_flag & rollback |
          |      flag values                   |
          +------------------------------------+
                           |
                           v
      +----------------------------------------------+
      |  S4: Update bootargs with parsed values for  |
      |      ostree-based system boot                |
      +----------------------------------------------+
                           |
                           v
                   +------------------+
                   |  End             |
                   +------------------+

```

## 功能

本应用实现以下功能：

1. 从指定的SD卡分区加载`boot.cfg`文件。
2. 解析文件内容，提取`boot_flag`、`hash`和`rollback_flag`。
3. 校验提取值的合法性。
4. 将提取的值用于更新U-Boot环境变量`bootargs`，以支持基于ostree的系统启动。

## 预期环境

- U-Boot版本：2021.01（请根据实际使用版本调整）
- 文件系统类型：EXT4
- 存储介质：SD卡
- 目标板支持的命令：`ext4load`，用于从EXT4文件系统加载文件。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202404151425353.png)


## 实现步骤

### S1: 从SD卡加载配置文件

使用`ext4load`命令从SD卡的指定分区加载`boot.cfg`配置文件到内存中的预定地址。此步骤使用`do_load`函数实现，函数参数包括存储设备编号、分区编号和目标内存地址。

```C
static int32_t read_boot_env_from_mmc_ext4(int mmc_dev, unsigned int part_num, const char *file_name, char *out_str)
{
    // Function body
}

```

### S2: 解析配置文件内容

配置文件`boot.cfg`应包含关键信息，格式如下：
`boot_flag=1;hash=<SHA-256 hash>;rollback_flag=0;`

应用程序首先将文件内容读入缓冲区，然后逐项解析出`boot_flag`、`hash`和`rollback_flag`。

```C
static int fetch_boot_env_from_string(const char *str_in, int32_t *boot_flag, char *hash_value, int32_t *rollback_flag)
{
    // Function body
}

```

### S3: 校验提取的值

校验`boot_flag`和`rollback_flag`的值是否为0或1，以确保值的合法性。

### S4: 更新U-Boot环境变量

将解析得到的值拼接到`bootargs`环境变量中。

```C
static int32_t update_bootargs_for_ostree_based_system(uint8_t boot_flag, uint8_t rollback_flag, uint8_t part_num, const char *hash)
{
    // Function body
}

```

