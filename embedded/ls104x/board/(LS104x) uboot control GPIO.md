The use of the trust architecture feature is dependent on programming fuses in the Security Fuse Processor (SFP). To program SFP fuses, the user is required to **supply 1.8 V to the TA_PROG_SFP pin** Power sequencing. TA_PROG_SFP should only be powered for the duration of the fuse programming cycle, with a per-device limit of six fuse programming cycles. At all other times, TA_PROG_SFP should be connected to GND. 

In the AutoX hardware design, the `TA_PROG_SFP` pin is connected to the `UART2_RTS_B` pin. To pull the `TA_PROG_SFP` pin, the `UART2_RTS_B` pin should be pulled high during the U-Boot boot stage.

Refers the link https://www.nxp.com/webapp/Download?colCode=LS1043A (datasheet),  `UART2_RTS_B` pin is shown in the following figure:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202401311629190.png)

In the hardware configuration, `UART2_RTS_B` is multiplexed with `GPIO1_20`. 

Referring to the documentation provided at [Farnell's datasheet](https://www.farnell.com/datasheets/2631233.pdf), the base addresses for GPIO can be obtained. A selection of these GPIO base addresses is presented in the block below:

```
Bank MPC@02300000:
MPC@023000000: input: 0 [ ]
MPC@023000001: input: 0 [ ]
MPC@023000002: input: 0 [ ]
...
Bank MPC@02310000:
MPC@023100000: input: 0 [ ]
MPC@023100001: input: 1 [ ]
MPC@023100002: input: 1 [ ]
MPC@023100003: input: 1 [ ]
...
Bank MPC@02330000:
MPC@023300000: input: 0 [ ]
MPC@023300001: input: 0 [ ]
MPC@023300002: input: 0 [ ]
MPC@023300003: input: 0 [ ]
```

Therefore, the address of the pin GPIO1_20 is `MPC@0230000000 + 020` = `MPC@0230000020`

According to the discussion in the tech community forum titled '[Can't control GPIO2_2/GPIO2_3 in u-boot](https://community.nxp.com/t5/Layerscape/LS1046A-Can-t-control-GPIO2-2-GPIO2-3-in-u-boot/td-p/1426614),' U-Boot provides a method for controlling GPIO pins. This can be achieved using the console command `gpio`.

Usage of read a GPIO pin status:

`gpio status MPC@0230000020`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202401311725098.png)

Usage of write 0 to a GPIO pin status:

`gpio clear MPC@0230000020`

Usage of set a GPIO pin status:

`gpio clear MPC@0230000020`


### RCW or ITS eFUSE

There are two phases options for enabling secure boot, which is the production phase and the development phase. It's depends on whether ITS fuse is blew.

* **Production phase** - Set the **ITS bit in SFP** to ensure that the system operates in secure and trusted manner. Once the SFP ITS fuse is blown, it cannot be changed.
* **Development phase** - Do not blow the ITS fuse, set `RCW[SB_EN] = 1` to enable secure boot.

### OTPMK and SRKH

* 必须先烧录`OTPMK`，这个不是影子寄存器，直接烧录
* 再烧录`SRKH mirror register`，注意SRKH mirror resgister是一个影子寄存器，用于调试阶段；

#### OTPMK

Blowing of **OTPMK** is essential to run secure boot for both Production and Development phases. 

|OTPMKR0..OTPMKR7|SRKHR0..SRKHR7|
|---|---|
|0x1e80234..0x1e80250|0x1e80254..0x1e80270|

![](https://raw.githubusercontent.com/carloscn/images/main/typora202402021646109.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202402021727165.png)


![](https://raw.githubusercontent.com/carloscn/images/main/typora202402021731514.png)

![Uploading file...hdk9v]()

#### SRKH mirror register



fdc2fed4317f569e1828425ce87b5cfd34beab8fdf792a702dff85e132a29687
0x1e80254 = fdc2fed4
0x1e80258 = 317f569e
0x1e8025c = 1828425c
0x1e80260 = e87b5cfd
0x1e80264 = 34beab8f
0x1e8026c = df792a70
0x1e80270 = 2dff85e1
0x1e80020 = 32a29687

mw.l 0x1e80254 0xd4fec2fd
mw.l 0x1e80258 0x9e567f31
mw.l 0x1e8025c 0x5c422818
mw.l 0x1e80260 0xfd5c7be8
mw.l 0x1e80264 0x8fabbe34
mw.l 0x1e80268 0x702a79df
mw.l 0x1e8026c 0xe185ff2d
mw.l 0x1e80270 0x8796a232



1a897018
d6e28f90
038cd077
4f4e1a1a
dc045b54
a14dc071
ef414760
8b4e04a6

SFP 0x1e80254 = 1a897018
SFP 0x1e80258 = d6e28f90
SFP 0x1e8025c = 038cd077
SFP 0x1e80260 = 4f4e1a1a
SFP 0x1e80264 = dc045b54
SFP 0x1e80268 = a14dc071
SFP 0x1e8026c = ef414760
SFP 0x1e80270 = 8b4e04a6


         SFP SRKHR0 = 1a897018
         SFP SRKHR1 = d6e28f90
         SFP SRKHR2 = 038cd077
         SFP SRKHR3 = 4f4e1a1a
         SFP SRKHR4 = dc045b54
         SFP SRKHR5 = a14dc071
         SFP SRKHR6 = ef414760
         SFP SRKHR7 = 8b4e04a6
         

mw.l 0x1e80254 0x1870891a
mw.l 0x1e80258 0x908fe2d6
mw.l 0x1e8025c 0x77d08c03
mw.l 0x1e80260 0x1a1a4e4f
mw.l 0x1e80264 0x545b04dc
mw.l 0x1e80268 0x71c04da1
mw.l 0x1e8026c 0x604741ef
mw.l 0x1e80270 0xa6044e8b

md 0x1e80254 0x10

mw.l 0x1e80254 0x1870891a
mw.l 0x1e80258 0x908fe2d6
mw.l 0x1e8025c 0x77d08c03
mw.l 0x1e80260 0x1a1a4e4f
mw.l 0x1e80264 0x545b04dc
mw.l 0x1e80268 0x71c04da1
mw.l 0x1e8026c 0x604741ef
mw.l 0x1e80270 0xa6044e8b



Input string not provided
Generating a random string
-------------------------------------------
* Hash_DRBG library invoked
* Seed being taken from /dev/random
-------------------------------------------
OTPMK[255:0] is:
aa75f9a1a57473d48d0603d53ad3e6d7aa243543c5cc1cb0cbcc3904aa13f3fc

 NAME    |     BITS     |    VALUE  
_________|______________|____________
OTPMKR 0 | 255-224      |   aa75f9a1 
OTPMKR 1 | 223-192      |   a57473d4 
OTPMKR 2 | 191-160      |   8d0603d5 
OTPMKR 3 | 159-128      |   3ad3e6d7 
OTPMKR 4 | 127- 96      |   aa243543 
OTPMKR 5 |  95- 64      |   c5cc1cb0 
OTPMKR 6 |  63- 32      |   cbcc3904 
OTPMKR 7 |  31-  0      |   aa13f3fc 

$ ./otpmk_conv aa75f9a1a57473d48d0603d53ad3e6d7aa243543c5cc1cb0cbcc3904aa13f3fc
mw.l 0x1e80234 0xa1f975aa
mw.l 0x1e80238 0xd47374a5
mw.l 0x1e8023c 0xd503068d
mw.l 0x1e80240 0xd7e6d33a
mw.l 0x1e80244 0x433524aa
mw.l 0x1e80248 0xb01cccc5
mw.l 0x1e8024c 0x0439cccb
mw.l 0x1e80250 0xfcf313aa

md 0x1e80234 0x10


# haochenwei @ sz-dl-275 in ~ [16:30:22] 
$ ./hash_conv 1a897018d6e28f90038cd0774f4e1a1adc045b54a14dc071ef4147608b4e04a6
mw.l 0x1e80254 0x1870891a
mw.l 0x1e80258 0x908fe2d6
mw.l 0x1e8025c 0x77d08c03
mw.l 0x1e80260 0x1a1a4e4f
mw.l 0x1e80264 0x545b04dc
mw.l 0x1e80268 0x71c04da1
mw.l 0x1e8026c 0x604741ef
mw.l 0x1e80270 0xa6044e8b


md 0x1e80254 0x10