# 18_OPTEE-OS_应用之（三）可信应用的开发

TA的全称是Trust Application，即可信任应用程序。CA的全称是Client Applicant，即客户端应用程序。TA运行在OP-TEE的用户空间，CA运行在REE侧。CA执行时代入特定的UUID和命令ID参数就能实现请求特定TA执行特定操作的需求，并将执行结果返回给CA。通过CA对TA的调用可实现在REE侧对安全设备和安全资源的操作。普通用户无法知道TA的具体实现，例如操作使用了什么算法、操作了哪些资源、获取了哪些数据等，这也就确保了相关资源和数据的安全。

# 1. TA及CA的概念
GP规范定义了CA调用TA的所有接口以及相关结构体和变量类型，同时也定义了TEE侧用户空间的所有接口和相关结构体和变量类型。如果TEE方案提供方是遵循GP规范实现了规范中定义的接口，上层应用开发者按照GP规范开发的CA和TA就能正常运行于各家TEE平台中。CA与TA有一些基本的概念，这些部分组成了TA与CA之间进行交互的基本条件，这些基本概念的说明如下。

**TEE Contexts**
TEE上下文（TEE Contexts）用于表示CA与TEE之间的抽象连接，即通过TEE上下文可将REE侧的操作请求发送到TEE侧。需注意的是，在执行打开CA与TA之间的会话之前必须先获取到TEE上下文。一般该值是打开REE侧的TEE驱动设备时返回的句柄，如果在REE侧支持多个TEE驱动，则在调用TEEC_InitializeContext时可指定具体的驱动设备名来获得特定的TEE上下文。

**Session**
会话（Session）是CA与特定TA之间的抽象连接。只有建立了CA与TA之间的会话后，CA才可调用TA中的命令来执行特定的操作。调用TEEC_OpenSession函数后，TEE会将建立的会话内容返回给CA，一个会话包含TEE上下文和会话ID值。

**Commands**
命令（Commands）是CA与TA之间通过会话进行具体操作的基础。在交互过程中，CA通过指定命令ID通知TA执行与命令ID匹配的操作。至于TA中执行什么操作则完全由TA开发者决定，命令ID只是CA与TA约定的某个特殊操作的ID值。

**Share Memroy**
共享内存（Share Memroy）被用于CA与TEE之间进行数据交互，CA可通过注册或分配的方式通知TEE注册或分配CA与TA之间的共享内存，CA和TEE对该块共享内存都具有指定的读写权限。

**Memory References**
Memroy Reference是CA与TEE之间一段固定范围的共享内存，Memory Reference可指定一个完整的共享内存块，也可指定共享内存块中的特定区域。

**UUID**
UUID是一个TA的身份标识ID。当CA需要调用某个TA时，TEE侧通过UUID来决定要加载和运行哪个TA镜像。

# 2. GP标准
GP标准的全称是GlobalPlatform，该标准对TEE的框架和安全需求做出了明确的规定[1]，并对REE侧提供的接口函数、数据类型和数据结构体也做出了明确的定义，并对TEE侧提供给TA开发者使用的接口函数、数据类型、数据结构体做出了明确的规定和定义。关于GP规范与TEE相关的文档，读者可到如下链接中自行查阅和下载：

https://www.globalplatform.org/mediaguidetee.asp

对CA和TA的开发者而言，需要仔细阅读GP对REE侧和TEE侧各种接口函数和数据结构体的定义，只有熟悉了接口函数以及数据结构体的定义后才能正确使用这些接口来开发特定的CA和TA。

# 3. GP标准对TA属性的定义
TA的属性定义了该TA的运行方式、链接方式、堆栈大小、版本等信息。在GP标准中对一个TA所需要具有的属性进行了严格的定义和说明，这些属性的名称、作用、值的内容说明如下所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007201133.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007201035.png)

需要被设定的TA属性都在TA源代码的user_ta_headr_defines.h文件中被定义，gpd.ta.appID的值通常被设置成该文件中TA_UUID的值。gpd.ta.singleInstance、gpd.ta.multiSession、gpd.ta.instanceKeepAlive的值通过在该文件中定义TF_FLAGS的值来确定。gpd.ta.dataSize的值由该文件中定义TA_DATA_SIZE的值来确定。gpd.ta.stackSize的值由该文件中定义TA_STACK_SIZE的值来确定。在OP-TEE中gpd.ta.version和gpd.ta.description的值使用默认值。gp.ta.description和gp.ta.version的值由TA_CURRENT_TA_EXT_PROPERTIES宏定义来确定。

# 4. GP标准定义的接口

GP标准中对REE侧和TEE侧提供给CA和TA调用的接口都做出了明确的定义，包括接口函数的函数名、作用、参数说明、返回值等。GP官方网站中名称为TEE_Client_API_Specification-Vx.x_c.pdf的文档给出了这些接口的详细说明，根据发布版本的不同，定义的接口可能也会有所不同。TEE侧定义的接口函数属于内部接口，详细内容查阅GP提供的名称为GPD_TEE_Internal_Core_API_Specification_vx.x.pdf的文档。

## 4.1 GP定义的客户端接口
GP定义的客户端接口包括9个函数和1个宏，使用这9个接口函数和宏就可满足CA的开发，只是CA需要配合TA一起使用，双方定义的UUID的值和命令ID的值需保持一致，这9个函数和宏的名称和作用如表所示。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007201453.png)

上述9个函数的函数原型、作用、参数说明、返回值的说明在本书8.2节中已进行了详细的介绍。这部分接口的实现会被编译到libteec库文件中，最终会被CA调用。GP规范CA中API说明文档：[TEE_Client_API_Specification-V1.0_c.pdf](https://higherlogicdownload.s3.amazonaws.com/GLOBALPLATFORM/transferred-from-WS5/TEE_Client_API_Specification-V1.0_c.pdf)

## 4.2 GP定义的内部接口
GP定义的内部接口是供TEE侧的TA或其他功能模块使用。大致可以分为Framwork层API、对数据和密钥操作的API、密码学操作API、时钟API、大整数算法API。由于API较多，就不对每个API进行一一说明，只给出各API的作用和名称。

**Framwork层接口**
Framwork层API是TEE用户空间实现对内存、TA属性等资源进行操作的API，该类API的说明如下表所示。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007201841.png)
![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007201932.png)

**对数据和密钥操作的API**
GP规定了特定的操作接口，用于TEE实现对各种数据流和密钥的操作。在使用安全存储、加解密等操作时都需使用该部分的接口。对数据流的操作是以object的方式完成的，对密钥的操作则是使用attr的方式来完成的。该部分API名称以及作用关系如表所示。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007202057.png)

**密码学操作接口**
TEE最重要的功能之一是提供了各种密码学算法的实现，并确保这些算法运行于安全环境中。GP定义了各种密码学的操作接口，这些API的名称和作用说明如表所示。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007202203.png)
![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007202242.png)

除此之外还有时间、大数的接口。

# 5. TA和CA的实现

本节将详细介绍如何完成CA和TA源代码的实现。本节中并不涉及TA中的特定操作的实现，只是介绍如何搭建CA和TA的整体框架和设定相关的参数，关于TA中的特定操作由读者根据自身的实际需求进行开发。

## 5.1 建立CA和TA的目录结构
秉承功能模块化的理念，建议在创建TA中的源代码文件时分为三个部分。
* 第一个部分为TA的入口调用文件，该TA中TA_xxxEntryPoint接口的实现将保存在该文件中。
* 第二部分为TA的处理文件，该文件中的内容是调用TA_InvokeCommandEntryPoint函数时switch case中各case中的具体实现。
* 第三部分为TA具体操作的实现，建议将不同的功能实现保存在不同的文件中，这样从代码阅读或调试时便于理解。

建立完目录结构和相关文件后，需将OP-TEE中的user_header_defines.h文件保存到TA的源代码中。通过修改该文件中的内容可实现对该TA属性的设定。

## 5.2 CA代码实现
在CA源代码中调用GP规范中定义的客户端的接口就可实现对TA的调用。在CA中调用客户端接口的顺序依次如下。

* **TEEC_InitializeContext**
   初始化CA与TEE之间的上下文，打开TEE驱动设备，得到一个TEEC_context。
* **TEEC_OpenSession**
  调用时代入TA的UUID，建立CA与指定TA之间的会话TEEC_PARAM_TYPES配置需要发送到TA的参数的属性，可将参数设定为input属性和output属性。
* **TEEC_InvokeCommand**
   代入会话ID、命令ID、包含参数内容的operation变量，开始发送请求给TEE来调用TA中的特定操作。
* **TEEC_CloseSession**
   调用完成后关闭CA与TA之间的会话。
* **TEEC_FinalizeContext**
  关闭CA与TEE之间的连接。在编写CA代码时需注意，在关闭上下文之前不要重复调用TEEC_InitializeContext函数，否则会报错，且如果在没有调用TEEC_CloseSession函数之前重复执行打开会话的操作可能会导致TEE中的空间不足。CA中的UUID和命令ID的定义一定要保证与TA中的命令ID和UUID的定义一致

## 5.3 TA代码的实现
TA代码需实现具体功能的所有操作，TA被TEE调用的各种操作的入口函数就是在表部分的API。所以需要在TA中实现这些API，最重要的是对TA_InvokeCommand-EntryPoint函数的实现。该函数需要定义各种命令ID对应的操作，至于每个命令ID需要实现什么功能就由开发者决定，但该命令ID的定义需要与CA中的命令ID的定义保持一致。TA属性的设定可通过修改user_ta_head_defines.h文件来实现，主要需修改如下的宏定义：
* **TA_UUID**：该TA的UUID值；
* **TA_FLAGS**：TA的访问属性，具体内容请参阅21.3节和GP规范；
* **TA_STACK_SIZE**：指定该TA运行时栈空间的大小；
* **TA_DATA_SIZE**：指定该TA运行时堆空间的大；
* **TA_CURRENT_TA_EXT_PROPERTIES**：该TA的扩展属性，主要包括TA名字、版本等。

# 6. TA和CA的集成
编辑完TA和CA的源代码后，需修改源代码中的Makefile文件和OP-TEE工程源代码中对应的板级mk文件和common.mk文件。

## 6.1 CA和TA的Makefile修改
需将CA所有源代码文件对应的目标文件添加到CA的Makefile文件中的OBJS目标中，并修改all目标的内容，将BINARY变量的值修改成开发者指定的值，并修改CFLAGS变量，将CA包含的头文件路径添加到cflag中。

对于TA部分则需修改ta目录下的Makefile文件和sub.mk文件。将ta/Makefile文件中的BINARY变量修改成UUID的值，将TA所有源代码文件的名称添加到ta/sub.mk文件中的srcs-y变量中，同时修改该文件中的global-incdirs-y变量，将TA的头文件目录添加到全局头文件路径中。对于srcs-y和globalincdirs-y变量的名字，开发者也可将其修改成s`rcs$(XXX)和global-incdirs-$(XXX)`的形式，然后通过在optee_os/mk/config.mk文件中定义XXX？=n或者是XXX？=y来控制在编译OP-TEE整个工程时是否需要编译该TA。

## 6.2 OP-TEE中comm.mk和xxx.mk文件的修改
若需要将该TA和CA集成到OP-TEE系统中，则需修改build/xxx.mk文件和build/common.mk文件。对xxx.mk文件的修改主要是将该TA和CA的编译集成到系统的编译目标当中，而对common.mk文件的修改则是指定编译TA和CA的具体依赖关系和编译路径，以及编译结果的保存路径和CA的编译结果是否需要集成到REE的文件系统中等。

# 7. TA和CA的调试
调试一个TA和CA程序时最主要的手段就是在报错的地方打印。在开发TA和CA的过程中会牵扯到程序编译、应用层、内核层、驱动层的问题。

关于程序编译的问题只需要根据编译报错的日志进行修改即可，若对编译过程不熟悉可在编译系统中添加打印的方式跟踪整个编译过程，然后定位编译报错的位置后进行对应的修改，一般都是函数和变量的定义问题以及相关选项的设置问题。

对于应用层的调试，最实用的方法就是在出错的地方添加打印信息，将错误时的数据打印出来然后结合实际的代码逻辑进行代码的调整和修改。为方便形成自己的调试风格，建议读者建立一套自己的调试打印模块，将系统提供的打印接口与自己的打印模块进行对接之后就可以很好地进行调试。

OP-TEE的内核层面的调试主要是各种密码学算法的报错调试。为确定在哪一步操作地方出现了错误，读者可以在代码中添加对应的打印信息，然后根据打印的信息进行对应的修改。关于AES和RSA算法部分，注意输入数据的长度对齐问题，至于加解密出来的数据是否正确，读者可使用openssl提供的接口进行实现后对两者的结果进行对比验证。

驱动层面则需要接口J-TAG或者Trace32等工具来进行调试，但到了该级别的调试就比较复杂，首先是调试环境以及调试工具的使用，但使用该方法更容易定位问题。

# 8. TA和CA的使用
整个CA可被编译成库文件供上层使用也可编译成可执行文件作为服务或指令在REE侧被使用。

当CA被编译成库文件后，使用该库文件时需为使用者提供对应的头文件。头文件中需要声明该库文件暴露给上层用户调用的API原型，在Android系统中也可将CA实现的接口以JNI的方式进行封装供APP使用。

当CA需要被编译成可执行文件时，需要添加main函数，在main函数中调用CA实现的接口来完成具体的操作。
