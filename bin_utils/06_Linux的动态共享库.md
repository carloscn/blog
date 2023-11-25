# 06_Linux的动态共享库

这一部分很多都是常识性的知识，我们只整理一部分知识。

* 共享库构造和析构函数
* 清除符号信息
* Linux系统符号版本机制

## 1 共享库构造和析构函数

Linux的共享库里面是有构造和析构函数的，构造函数和析构函数分别是在库的加载和销毁的时候被系统调用的，这里面分为两种情况：

* 使用编译的链接器将动态链接库和目标文件链接
* 使用dlopen在函数体内打开动态链接库

这两种情况的析构函数和构造函数调用的位置是不同的。第一种方法，在load二进制elf文件的时候，动态库被加载之后，就会调用构造函数，析构函数会在elf生命期结束之后调用。第二种方法，在dlopen的时候会调用构造函数，而dlclose的时候会调用析构函数。

构造函数使用方法：

`void __attribute__((constructor(1))) init_function(void) {}`

析构函数使用方法：

`void __attribute__((destructor(5))) deinit_function(void) {}`

在函数内实现这两个函数就可以了，函数名字可以自定，不可以少的是attribute，后面1和5数字代表着优先级。

### 1.1 使用编译链接法

我们准备一个lib文件：

```C
#include <stdio.h>
#include "libprov.h"

#define debug_log printf("%s:%s:%d--",__FILE__, __FUNCTION__, __LINE__);printf

int prov_lib_init(const char *init_info)
{
    if (init_info == NULL) {
        debug_log("bad input parameter\n");
        return -1;
    }

    debug_log("do init: %s\n", init_info);

    return 0;
}

int prov_lib_do(const char *do_info)
{
    if (do_info == NULL) {
        debug_log("bad input parameter\n");
        return -1;
    }

    debug_log("do init: %s\n", do_info);

    return 0;
}

void prov_lib_cleanup()
{
    debug_log("do cleanup\n");
}

int __attribute__((constructor(5))) prov_lib_init_auto()
{
    debug_log("call the auto init\n");
    return 0;
}

void __attribute__((destructor(10))) prov_lib_cleanup_auto()
{
    debug_log("call the auto clean\n");
}
```

编译成动态库：`gcc libprov.c -fPIC -shared -o libprov.so -g`

准备一个user的C文件：

```C
#include <stdio.h>
#include "libprov.h"

#define debug_log printf("%s:%s:%d--",__FILE__, __FUNCTION__, __LINE__);printf
C
int main(void)
{
    int ret = 0;
    debug_log("main start\n");
    ret = prov_lib_init("main init function\n");
    ret = prov_lib_do("main do prov\n");
    prov_lib_cleanup();
    debug_log("main end\n");
    return ret;
}
```

编译:`gcc user_gcc_link.c  libprov.so -o user.elf -g`

 <img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220415160441481.png" alt="image-20220415160441481" width="80%" />

### 1.2 使用dlopen法

准备C文件：

```C
#include <stdio.h>
#include <dlfcn.h>

#define debug_log printf("%s:%s:%d--",__FILE__, __FUNCTION__, __LINE__);printf

int main(void)
{
    int ret = 0;
    debug_log("main start\n");
    void *handle = NULL;
    int (*prov_init)(const char *) = NULL;
    int (*prov_do)(const char *) = NULL;
    void (*prov_cleanup)(void) = NULL;

    handle = dlopen("./libprov.so", RTLD_LAZY);
    if (!handle) {
        fprintf(stderr, "%s\n", dlerror());
        return -1;
    }
    prov_init = (int (*)(const char *)) dlsym(handle, "prov_lib_init");
    prov_do = (int (*)(const char *)) dlsym(handle, "prov_lib_do");
    prov_cleanup = (void (*)(void)) dlsym(handle, "prov_lib_cleanup");

    ret = prov_init("main init function\n");
    ret = prov_do("main do prov\n");
    prov_cleanup();

    dlclose(handle);

    debug_log("main end\n");
    return ret;
}
```

编译：`gcc user_dl_open.c -o user.elf -g`

<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220415160717564.png" alt="image-20220415160717564" width="90%"/>

我们以后编写共享库的时候记得完善析构函数和构造函数。

## 2 清除符号信息

正常情况下编译出来的共享库和执行文件里面带有符号信息和调试信息，这些信息调试的时候非常有用，但是对于最终版来说是没有任何作用的，而且使得文件很大。使用strip的工具可以清除这些信息。

`$ strip libprov.so`

Note 在MACOX上使用strip 需要 -x[^1]。

或者在ld的时候使用-S参数（-s消除所有符号信息，-S消除调试信息）来表示清除调试信息。或者使用gcc中通过"-Wl,-s"和"-Wl,-S"来完成。

我们看看效果，使用`objdump -t libprov.so`打印所有符号

<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220415162253254.png" alt="image-20220415162253254"/>

## 3 Linux版本机制

在Linux下，我们使用ld链接共享库的时候，可以使用`--version-script`参数，控制版本。假设有个版本描述脚本叫做lib.ver。

`gcc -shared -fPIC libprov.c -Xlinker --version-script lib.ver -o libprov.so`

lib.ver脚本按照以下方式写：

```
VERS_1.2 {
		global:
			prov_lib_init;
			prov_lib_do;
			prov_lib_cleanup;
		local:
		    *;
};
```

这里面规定，这三个参数的版本是v1.2。此时我们编译，出来的函数是1.2版本的。如果们重新编译了低版本的库，这个时候会报错如图。

![image-20220415164900139](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220415164900139.png)

综上，我们编写库的时候要注意：

* 完善析构函数和构造函数。
* release的时候注意strip文件。
* 要定义好符号的版本号。

## Ref

[^1]:[Symbol stripping on OSX fails for release build ](https://github.com/pybind/pybind11/issues/595)