# # 01_Script_makefile_summary

A project has innumerable source files that are organized along various paths based on type, function, and module. The rules in the makefile state which files should be rebuilt, which files should be built later, and which files should be constructed first. The makefile prefers a shell with sophisticated functions. [^1][^2]

# 1. grammer 

## 1.1 basic

```Makefile
object:dependence
	<TAB> command line
```

* The `object` is the target to be compiled or an action.
* The `dependence` is a pre-item performing the object depended.
	* One object allows having multiple dependencies.
* The `command` is the specific command under the performing object.
	* Each command occupies one line.
* By default, the first object will be executed when no option is specified.  

Examples:
https://github.com/carloscn/clab/blob/master/macos/test_makefile/single/Makefile.hello

```Makefile
a:
	@echo "a"

b:
	@echo "b"

hello:
	@echo "hello world!"

clean:
	@rm -rf *.out

```

## 1.2 common options

```bash
make [-f file] [options] [target]
```

By default, make command search `GUNmakefile` `makefiles` `Makefile` in current directory using as the make input file, except for that use option`-f` to specify a make file by filename.

* `-f`: specify a make file by file path
* `-v`: show the version number
* `-n`: perform the makefile, output commands record only, don't perform.
* `-s`: perform the makefile, don't show commands record.
* `-w`: show paths before and after execution.
* `-C`: show paths where the makefile is located.

## 1.3 using a common include make

We can use a makefile as a public makefile liking as the function `#include` in C language. 

```makefile
TARGET = z
OBJ = a.o

include ../makefile.common
```

Note, the `=` and `:=`:

* `=` : the final value assignment, whatever the calling before the assigning value or behind the assigning value.
* `:= `

There is a example for describing the difference between `=` and `:=`:

```makefile
#### example 1:

a = 123
b = $(a)

a:
	echo $(a) $(b) $(c)

a = 456
c = 789


#### example 2:
x = 789
y = $(x)
y = $(y)

b:
	echo $(x) $(y)
```
The `make a`output is `456 456 789`, and the `make b` output is an error, becasue the object b cannot ensure the y value that referenced itself.

Now, we replaced the `=` to `:=`:

```makefile
#### example 1:

a = 123
b := $(a)

a:
	echo $(a) $(b) $(c)

a = 456
c = 789


#### example 2:
x = 789
y = $(x)
y := $(y)

b:
	echo $(x) $(y)
```
The `make a` output is `456 123 789`, and the `make b` is no longer an error, rather then `789 789`.

## 1.4 using the linux shell within makefile

https://github.com/carloscn/clab/blob/master/macos/test_makefile/Makefile.shll

```makefile

.PHONY: clean

file = test.txt
A := $(shell ls)
B := $(shell pwd)
C := $(shell (if [ ! -f $(file) ];then touch $(file); fi;))

show:
	echo $(A)
	echo $(B)
	echo $(C)

clean:
	$(RM) $(file)
```

## 1.5 using the nesting function of makefile

https://github.com/carloscn/clab/blob/master/macos/test_makefile/multi/src/Makefile

```makefile

.PHONY: clean algo print

algo:
	make -C ./algo

print:
	make -C ./print

clean:
	make -C ./algo clean
	make -C ./print clean
	rm -rf $(OBJ) *.elf
```

## 1.6 using condition define

https://github.com/carloscn/clab/blob/master/macos/test_makefile/Makefile

```makefile

A = 123
B = default
C := $(B)

ifeq ($(A), 123)
	B := yes
else
	B := no
endif

ifneq ($(A), 123)
	C := yes
else
	C := no
endif

show:
	echo $(A)
	echo $(B)
	echo $(C)
ifdef A
	@echo "ifdef A : the A defined"
endif

ifndef A
	@echo "ifndef A: THE A not defined"
endif

```

This is always combined `gcc -D` with `ifdef`. Sometimes, we want to use some variables of the makefile in a C source file, different variables different flows, such as using the debug, and setting the version number. Using the `gcc -D` is equal to defining a macro by `#define`.

```Makefile
version=0.0.2
release_number=2
 
all: test
 
test: test.c
	$(CC) -o $@ $^ -DDEBUG_PRINT -D VERSION='"$(version)"' -D RELEASE_NUMBER=$(release_number)
 
.PHONY: clean
 
clean:
	rm test
```

And the test.c is :

```C
#include <stdio.h>
 
int main(int argc, char **argv) {
	if (DEBUG_PRINT) {
		printf("%s (version: " VERSION ", release number: %d)\n", 
				argv[0], RELEASE_NUMBER);
	}
 
	return 0;
}
```

So, we can utilize the `ifdef` of makefile and `-D` of gcc to do something. 

## 1.7 using the foreach

In a makefile, the foreach can provide the loop function for us. We can make use of the foreach to create a directory or something.

https://github.com/carloscn/clab/blob/master/macos/test_makefile/Makefile

```makefile
TARGET=a b c d

loop:
	mkdir $(foreach v, $(TARGET), $v.txt)

clean:
	rm -rf $(foreach v, $(TARGET), $v.txt)
```

## 1.8 using the function

https://github.com/carloscn/clab/blob/master/macos/test_makefile/Makefile

```makefile

define func_1
	return 123
endef

define func_2
	return $(1)
endef

call:
	echo $(call func_1)
	echo $(call func_2, 9999)
```

# 2. make val

We need to set up a C build environment shown in the following picture, and you can find the original source code in https://github.com/carloscn/clab/tree/master/macos/test_makefile/single

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221206210243.png)

The main.c file depends on add.c div.c mul.c sub.c, and the main.c is implement simple add or div functions by calling the sub-functions.

We written the makefile as follow:

```makefile
main :
	gcc add.c div.c sub.c mul.c main.c -o main.elf

clean :
	rm -rf *.o *.elf
```

It has a very basic makefile and appears quite silly. All the objects would be recompiled when we altered a single file. If the project is particularly large, compiling will take a lot of your time.

So we need to optimize the makefile and make it easier and more convenient.

```makefile
# make -o to dump the VAL

OBJ = add.o div.o sub.o mul.o main.o
TARGET = main

$(TARGET).elf : $(OBJ)
	gcc $(OBJ) -o $(TARGET).elf

main.o : main.c
	gcc -c main.c -o main.o

add.o : add.c
	gcc -c add.c -o add.o

div.o : div.c
	gcc -c div.c -o div.o

sub.o : sub.c
	gcc -c sub.c -o sub.o

mul.o : mul.c
	gcc -c mul.c -o mul.o

clean :
	@rm -rf *.o *.elf
	@echo "make clean finish"
```

When one of these files changed, the optimized makefile would re-compile the changed .c file. 

## 2.1 using user env

Based on the simple makefile, we replaced `*.o` files with `OBJ` by user environment. The difference between the simple makefile and the new makefile is shown in the following figure:

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221206211419.png)

## 2.2 using sys env

* `$*`: the object name that does not include the extension name.
* `$+`: all the dependence files, seperated by space.
* `$<`: the first condition in rules.
* `$?`: the dependence files that the time later than object.
* `$^`: all the dependence files are not repeated.
* `$%`: if the object is an archive member, the variable indicates the member name of the object.

Based on the user env makefile, we replaced `OBJ` and `TARGET` with `$^` and `^@` by the system environment. 

```makefile
# make -p to dump the VAL

OBJ = add.o div.o sub.o mul.o main.o
TARGET = main

$(TARGET).elf : $(OBJ)
	gcc $(OBJ) -o $@

main.o : main.c
	gcc -c $^ -o $@

add.o : add.c
	gcc -c $^ -o $@

div.o : div.c
	gcc -c $^ -o $@

sub.o : sub.c
	gcc -c $^ -o $@

mul.o : mul.c
	gcc -c $^ -o $@

clean :
	@rm -rf $(OBJ) $(TARGET).elf
	@echo "make clean finish"
```

The difference between these makefiles is shown in the following figure:

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221206212409.png)

## 2.3 using build env

The build env can be shown by `make -p`.

* `AS`:  as
* `CC`:  gcc
* `CPP`: cc
* `CXX`: g++ or c++
* `RM`: rm -rf 

Based on the system env makefile, we replaced `gcc` to `$(CC)`by build environment. 

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221206212535.png)

## 2.4  phony object

```bash
.PHONY clean hello
```

The phony object is that whether the file is changed or not, the object declared by phony must be performed.

If there is a hello file on your makefile path, and hello is still an object in your makefile.  When the hello is not declared by phony:

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221206214114.png)

If the hello is declared by .PHONY, the object will be performed by making command whether the file with the same name exists in the directory or not.

## 2.5 mode matching

* `%.*:%.c`: .o files depend on .c files corresponding.
* `wildcard`: `$(wildcard ./*.c)` obtain the all .c file in the current directory.
* `patsubst`:`$(patsubst %.c, %.o, $(wildcard *.c))` replace .c to .o corresponding.

```makefile
.PHONY: clean hello

OBJ = add.o div.o sub.o mul.o main.o
TARGET = main

$(TARGET).elf : $(OBJ)
	$(CC) $(OBJ) -o $@

%.o : %.c
	$(CC) -c $^ -o $@

hello :
	@echo "hello test"

clean :
	@$(RM) $(OBJ) $(TARGET).elf
	@echo "make clean finish"
```

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221206215055.png)

using the wildcard

```makefile
.PHONY: clean hello

OBJ = $(patsubst %.c, %.o, $(wildcard *.c))
TARGET = main

$(TARGET).elf : $(OBJ)
	$(CC) $(OBJ) -o $@

%.o : %.c
	$(CC) -c $^ -o $@

hello :
	@echo "hello test"

show:
	@echo $(wildcard *.c)
	@echo $(patsubst %.c, %.o, $(wildcard *.c))

clean :
	@$(RM) $(OBJ) $(TARGET).elf
	@echo "make clean finish"
```

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221206215142.png)

# 3. Using libraries

## 3.1 Using the dynamic library

We need to review how to comiple and use dynamic library using the `gcc`.  Please refer to section 1 of [05_ELF文件_动态链接](https://github.com/carloscn/blog/issues/21). 

When we generate a dynamic library on Linux:
* `-fPIC`: this is the position-independent code
* `-shared`: this is specifying the type of dynamic library.

When we use a dynamic library on Linux:
* `-l`: this is specifying the dynamic library name
* `-L`: this is specifying the path of the dynamic library.
* `-I`: this is specifying the path of included files.

Now, we create files shown in the following figure:

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221210110924.png)

cacu.c:
```C
#include "cacu.h"
// gcc -fPIC -shared -o libcacu.so cacu.c

int cacu(int a, int b) {
    return a * b + a + b;
}
```

cacu.h:
```C
int cacu(int a, int b);
```

Makefile:
```makefile
.PHONY: clean

SOURCE = $(wildcard *.c)
TARGET = libcacu.so
LD = -fPIC -shared

$(TARGET):
	$(CC) $(LD) -o $@ $(SOURCE)

clean:
	rm -rf *.so
```

## 3.2 Using the static library

We can make a static library compiling within a makefile. Compared with a dynamic library, the static library needs not original library files during the application running, in other words, the static library has been inserted into the object file, which leads to the application being bigger than the one compiled by the dynamic library. For a static library generation, we can refer to section 3 of [# 03_ELF文件_静态链接](https://github.com/carloscn/blog/issues/11). The common commands summary is as follows:

As we compile a static library:
* `gcc -c hello.c -o hello.o`
* `ar -r libhello.a hello.o`

As we use a static library:
* `gcc -lhello -L./ main.c -o main.elf`

Example:

Touch a `mod.c`
```C
#include "mod.h"

int mod(int a, int b) {
    return a % b;
}
```

Touch a `mod.h`
```C
int mod(int a, int b);
```

The makefile is:
```makefile
.PHONY: clean

SOURCE = $(wildcard *.c)
TARGET = libmod.a
OBJ = mod.o

$(TARGET): $(OBJ)
	$(AR) -r $@ -o $^

mod.o:
	$(CC) -c -o $@ $(SOURCE)

clean:
	rm -rf $(OBJ) $(TARGET)
```


# 4. Ref
[^1]: [ make.html ](https://www.gnu.org/software/make/manual/make.html#toc-Overview-of-make)
[^2]:[ makefile从入门到项目编译实战 ](https://www.bilibili.com/video/BV1Xt4y1h7rH?p=5&vd_source=edb82fc22cc42ae5edc71f0a1c41410e)