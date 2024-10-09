
# 通过GRUB Multiboot2引导自制操作系统


## 前言


之前花了一周时间，从头学习了传统 BIOS 的启动流程。惊讶于背后丰富的技术细节的同时，也感叹 x86 架构那厚重的历史包袱。毕竟，谁能想到，一个现代 CPU 竟然需要通过操作“键盘控制器寄存器”来启用一条地址线呢。


最终，出于兼容性和功能性的考虑，我还是决定投入 GRUB 的怀抱。况且，让自己写的操作系统和本机的 Linux 一同出现在 GRUB 菜单中并成功引导启动，也是一件非常有成就感的事。



> **完整代码在[附录](https://github.com)部分**


## 开发环境




| 项目 | 版本 |
| --- | --- |
| 系统 | Windows Subsystem for Linux \- Arch (2\.3\.24\) |
| 编译器 | gcc version 14\.2\.1 20240910 (GCC) |
| 引导程序 | grub\-install (GRUB) 2:2\.12\-3 |
| 虚拟机 | QEMU emulator version 9\.1\.50 / VirtualBox 7\.0\.6 |


## 1 Multiboot2 规范


想让 GRUB 识别并引导我们自制的操作系统，就得先了解 Multiboot2 规范。


官方文档：[Multiboot2 Specification version 2\.0](https://github.com)


### 1\.1 Multiboot2 header



> An OS image must contain an additional header called Multiboot2 header, besides the headers of the format used by the OS image. The Multiboot2 header must be contained completely within the first 32768 bytes of the OS image, and must be 64\-bit aligned. In general, it should come as early as possible, and may be embedded in the beginning of the text segment after the real executable header.


这段话的大致意思是：


在 Multiboot2 规范中，操作系统镜像的前 32768 字节内，必须包含一个名为 Multiboot2 header 的数据结构，并且起始地址必须 64\-bit 对齐。


Multiboot2 header 的具体结构如下：




| Offset | Type | Field Name |
| --- | --- | --- |
| 0 | u32 | magic |
| 4 | u32 | architecture |
| 8 | u32 | header\_length |
| 12 | u32 | checksum |
| 16\-XX |  | tags |


#### 1\.1\.1 Magic fields


Multiboot2 header 的前四个字段，也就是0\~15字节的内容，被称为 magic fields


`magic` 字段是 Multiboot2 header 的识别标志，它必须是十六进制值 `0xE85250D6`


`architecture` 字段指定 CPU 的架构。`0` 表示 i386 的 32 位保护模式，`4` 表示 32 位 MIPS 架构


`header_length` 字段记录了 Multiboot2 header 的长度（以字节为单位）


`checksum` 字段是一个 32 位无符号数，当它与 magic fields 其他字段（即 `magic` 、 `architecture` 和 `header_length` ）相加时，其 32 位无符号和必须为零。


#### 1\.1\.2 Tags


Tags 由一个接一个的 tag 结构体组成，在必要时进行填充，以**确保每个 tag 都从 8 字节对齐的地址开始**。Tags 以一个 type \= 0 且 size \= 8 的 tag 作为结束标志（相当于字符串结尾的'\\0'）。



> 值得注意的是，官方文档给出的 [boot.S](https://github.com) 代码并 **没有** 给 tag 的地址进行 8 字节对齐。这会导致 GRUB 读取 tag 时出现错位，出现各种奇怪的错误（比如 error: unsupported tag: xxx）


每个 tag 都有如下基本结构：



```
        +-------------------+
u16     | type              |
u16     | flags             |
u32     | size              |
        +-------------------+
```

`type` 用于表示 tag 的类型，因为不同的 tag 在 `size` 之后可能还会有其他数据字段。


如果`flags` 的第 0 位（也称为 `optional`）为 1，表示如果引导加载程序缺乏相关支持，它可以忽略这个 tag。


`size` 表示整个 tag 的长度。


例如，表示程序入口地址的 **entry address tag** 结构如下：



```
        +-------------------+
u16     | type = 3          |
u16     | flags             |
u32     | size              |
u32     | entry_addr        |
        +-------------------+
```

GRUB 会根据 type \= 3 判断它是 entry address tag，并在准备工作完成后，跳转到 `entry_addr` 字段中的地址运行操作系统。


除此之外，还有专门在 EFI 引导使用的 **EFI i386 entry address tag**：



```
        +-------------------+
u16     | type = 8          |
u16     | flags             |
u32     | size              |
u32     | entry_addr        |
        +-------------------+
```

其结构与 **entry address tag** 相同，只是 type \= 8 。既然都是指定程序入口，那么**二者同时存在**时会发生什么呢？


在使用 EFI 引导的情况下，**entry address tag 将被忽略**，引导加载程序会跳转到 **EFI i386 entry address tag** 提供的地址。


而使用传统 BIOS 引导时，引导加载程序会直接**报错** `error: unsupported tag: 0x08` 。这是因为传统引导方式不支持 EFI 相关的 tag ，此时 `flags` 就派上用场了。只要把 `flags` 最低位设为 1 ，就可以让引导程序在不兼容时忽略这个 tag，而在 EFI 引导时又能正常使用它。


如此一来，我们的启动程序就能够兼容多种启动方式。


除此之外的 tag 类型，可以在 [Multiboot2 Specification version 2\.0](https://github.com) 的 3\.1\.3 \~ 3\.1\.13 节找到详细说明。


### 1\.2 机器状态


官方文档中的 [3\.3 I386 machine state](https://github.com) 说明了引导加载程序调用 32 位操作系统时的机器状态。


除了开启保护模式和启用 A20 gate 这些常规操作以外，还有两条比较重要的内容：


1. `EAX` 必须包含标识值 `0x36d76289`，该值的存在向操作系统表明它是由兼容 Multiboot2 的引导加载程序加载的。
2. `EBX` 必须包含引导加载程序提供的 Multiboot2 信息结构体的 32 位物理地址（请参阅 [Boot information format](https://github.com)）。


这两个寄存器中的数据，会在后续内核代码中用到。


## 2 代码实现


官方文档提供了对应的示例代码：[4\.4 Example OS code](https://github.com)


***但是！***


前文有提过，文档给出的 [boot.S](https://github.com) 是有 **BUG** 的！因为 ***没有*** 给 tag 的地址进行 8 字节对齐，导致了 GRUB 读取 tag 会出现错位。


**我将完整的代码放在了[附录](https://github.com)部分。** 本章节主要内容是依据 Multiboot2 规范，对代码片段进行分析。


### 2\.1 定义 Multiboot header



```
#include "multiboot2.h"

        .text
        
multiboot_header:
        /*  Align 64 bits boundary. */
        .align  8
        /*  magic */
        .long   MULTIBOOT2_HEADER_MAGIC
        /*  ISA: i386 */
        .long   MULTIBOOT_ARCHITECTURE_I386
        /*  Header length. */
        .long   multiboot_header_end - multiboot_header
        /*  checksum */
        .long   -(MULTIBOOT2_HEADER_MAGIC + MULTIBOOT_ARCHITECTURE_I386 + (multiboot_header_end - multiboot_header))

entry_address_tag_start:    
        /* 每个 tag 都要 8 字节对齐 */
        .align  8
        .short MULTIBOOT_HEADER_TAG_ENTRY_ADDRESS
        .short MULTIBOOT_HEADER_TAG_OPTIONAL
        .long entry_address_tag_end - entry_address_tag_start
        /*  entry_addr */
        .long multiboot_entry
entry_address_tag_end:

        /* 终止 tag 表示 tags 部分结束 */
end_tag_start:
        /* 每个 tag 都要 8 字节对齐 */
        .align  8
        .short MULTIBOOT_HEADER_TAG_END
        .short 0
        .long end_tag_end - end_tag_start
end_tag_end:
multiboot_header_end:

multiboot_entry:
        /* ...... */
```

为了代码的可读性，使用宏定义来表示数值。具体内容参考： [multiboot2\.h](https://github.com)


根据规范要求，我们将 Multiboot header 定义在了 text 段的开头，使数据尽量靠前。


这里使用了 entry address tag 表示程序入口的地址，也就是 GRUB 执行内核时要跳转的目标。不过，其实还有一个方法可以设置程序入口。



```
#include "multiboot2.h"

        .text

        jmp     multiboot_entry
        
multiboot_header:
        /*  Align 64 bits boundary. */
        .align  8
        /*  magic */
        .long   MULTIBOOT2_HEADER_MAGIC
        /*  ISA: i386 */
        .long   MULTIBOOT_ARCHITECTURE_I386
        /*  Header length. */
        .long   multiboot_header_end - multiboot_header
        /*  checksum */
        .long   -(MULTIBOOT2_HEADER_MAGIC + MULTIBOOT_ARCHITECTURE_I386 + (multiboot_header_end - multiboot_header))

        /* 终止 tag 表示 tags 部分结束 */
end_tag_start:
        /* 每个 tag 都要 8 字节对齐 */
        .align  8
        .short MULTIBOOT_HEADER_TAG_END
        .short 0
        .long end_tag_end - end_tag_start
end_tag_end:
multiboot_header_end:

multiboot_entry:
        /* ...... */
```

相较于上一个代码，我们在 text 段的开头添加 `jmp multiboot_entry` ，并删除 entry address tag ，也能实现相同的功能。


这是因为，在没有使用 tag 指定程序入口时， GRUB 会直接执行我们的操作系统程序。此时，运行的第一条指令就是我们添加的 `jmp multiboot_entry`


### 2\.2 启动入口



```
#include "multiboot2.h"

#ifdef HAVE_ASM_USCORE
# define EXT_C(sym)                     _ ## sym
#else
# define EXT_C(sym)                     sym
#endif

#define STACK_SIZE                      0x4000
        
        .text
        
multiboot_header:
        /* ...... */
multiboot_header_end:

        /* 程序入口位置 */
multiboot_entry:
        /*  Initialize the stack pointer. */
        movl    $(stack + STACK_SIZE), %esp

        /*  Reset EFLAGS. */
        pushl   $0
        popf

        /*  Push the pointer to the Multiboot information structure. */
        pushl   %ebx
        /*  Push the magic value. */
        pushl   %eax

        /*  Now enter the C main function... */
        call    EXT_C(cmain)

        /*  Halt. */
        pushl   $halt_message
        call    EXT_C(printf)
        
loop:   hlt
        jmp     loop

halt_message:
        .asciz  "Halted."

        /*  Our stack area. */
        .comm   stack, STACK_SIZE
```

接下来，到了操作系统入口位置。


首先，需要做的事情是初始化栈指针：


1. `#define STACK_SIZE 0x4000`：用宏定义表示栈的大小
2. `.comm stack, STACK_SIZE`：`.comm` 指令用来分配一块未初始化的数据段内存，这里分配了 `STACK_SIZE` 字节的空间给 `stack`
3. `movl $(stack + STACK_SIZE), %esp`：因为栈从顶部向下生长，所以将栈的顶部地址放入栈指针寄存器 `%esp` 中


接着清空 EFLAGS 寄存器：


1. `pushl $0`：将立即数 0 压入栈
2. `popf`：将栈顶的值弹出，并加载到 EFLAGS 寄存器


初始化完成后，就可以调用内核入口的 C 语言函数。我们之前在 [1\.2 机器状态](https://github.com)中提到过， `EAX` 和 `EBX` 寄存器保存着标识值和 Multiboot2 信息结构体地址，接下来内核会用到这些数据。由于内核代码是 C 函数，因此我们需要根据 C 语言的传参标准，按函数参数相反的顺序将数据压入栈中：



```
/* 内核入口函数定义： */
/* void cmain (unsigned long magic, unsigned long addr) */

pushl   %ebx
pushl   %eax

call    EXT_C(cmain)
```

### 2\.3 内核代码


这部分的代码和官方文档中的 [kernel.c](https://github.com) 一致。该内核的主要功能是在屏幕上打印出 Multiboot2 信息结构，主要用于测试 Multiboot2 引导加载程序，同时也可作为实现 Multiboot2 内核的参考示例。


### 2\.4 代码构建


需要先在附录中获取 [boot.S](https://github.com), [kernel.c](https://github.com) 和 [multiboot2\.h](https://github.com) 代码



```
CC = gcc
LD = ld
CFLAGS = -m32 -fno-builtin -fno-stack-protector -nostartfiles
LDFLAGS = -Ttext 0x100000 -melf_i386 -nostdlib
KERNEL_NAME = kernel.bin

all: $(KERNEL_NAME)

$(KERNEL_NAME): boot.o kernel.o
	$(LD) $(LDFLAGS) $^ -o $@

boot.o: boot.S
	$(CC) -c $(CFLAGS) $< -o $@

kernel.o: kernel.c
	$(CC) -c $(CFLAGS) $< -o $@

clean:
	rm -f *.o $(KERNEL_NAME)
```

简单说明一下编译选项：



```
CFLAGS = -m32 -fno-builtin -fno-stack-protector -nostartfiles
```

* `-m32`：让编译器生成 32 位代码
* `-fno-builtin`：禁用编译器对标准库函数（如 memcpy、strlen 等）的内建优化实现。确保这些函数不被替换为编译器优化版本
* `-fno-stack-protector`：禁用栈保护功能，否则会编译报错
* `-nostartfiles`：不使用标准启动文件，因为操作系统的启动方式与常规程序不同



```
LDFLAGS = -Ttext 0x100000 -melf_i386 -nostdlib
```

这些是传递给链接器（如 ld）的选项，用于控制链接行为：


* `-Ttext 0x100000`：表示程序的代码段（text segment）将被放置在内存地址 0x100000 处，这是加载操作系统常用的位置
* `-melf_i386`：指定输出文件的目标格式是 32 位的 ELF 格式（适用于 i386 架构），这是 Multiboot2 规范建议的格式
* `-nostdlib`：让链接器不要链接标准库。这是必要的，因为在内核或引导程序中，不会使用标准 C 库，而是会自己实现所需的功能或直接操作硬件


## 3 镜像制作


众所周知，现代 PC 有 `Legacy` 和 `UEFI` 两种启动方式，而接下来的镜像将会使用 `Legacy + MBR` 的启动方式。



> 选择 `Legacy` 的主要原因是，`EFI` 似乎只能输出像素，无法直接打印文本。这会导致我们无法查看内核打印的信息。
> 
> 
> 具体说法可以参考这个链接：[https://forum.osdev.org/viewtopic.php?f\=1\&t\=28429](https://github.com)


### 3\.1 创建镜像


使用 `dd` 命令创建一个 64M 的文件：



```
dd if=/dev/zero of=disk.img bs=1M count=64
```

使用 `parted` 为镜像文件分区：



```
parted -s disk.img mklabel msdos mkpart primary ext2 1MiB 100%
```

将镜像文件作为虚拟磁盘挂载，并创建 ext2 文件系统和挂载分区



```
sudo losetup -P /dev/loop0 disk.img
sudo mkfs.ext2 /dev/loop0p1
sudo mount /dev/loop0p1 /mnt
```

### 3\.2 安装 GRUB


安装 `Legacy` 启动兼容的 GRUB ，并创建 `grub.cfg` 配置文件



```
sudo grub-install --target=i386-pc --boot-directory=/mnt/boot /dev/loop0

sudo mkdir -p /mnt/boot/grub
cat <<EOF > /mnt/boot/grub/grub.cfg
set timeout=20
set default=0

menuentry "MyOS" {
    multiboot2 /boot/kernel.bin
    boot
}
EOF
```


> 配置文件里的启动方式要写 `multiboot2` 而不是 `multiboot`


安装完成后，取消挂载



```
sudo umount /mnt
sudo losetup -d /dev/loop0
```

## 4 虚拟机运行


### 4\.1 QEMU


运行目标平台为 i386 的 QEMU，内存要大于 128 M



```
qemu-system-i386 -m 1G -hda disk.img
```

[![](https://img2023.cnblogs.com/blog/2583637/202410/2583637-20241008154047233-1579858167.png)](https://img2023.cnblogs.com/blog/2583637/202410/2583637-20241008154047233-1579858167.png)


[![](https://img2023.cnblogs.com/blog/2583637/202410/2583637-20241008154148973-1230153976.png)](https://img2023.cnblogs.com/blog/2583637/202410/2583637-20241008154148973-1230153976.png)



> 这条蓝线也是 kernel.c 绘制的。如果你的机器和 GRUB 是 EFI 启动，就会发现无法输出字符，但是这个蓝线还在。因为，它是通过设置 framebuffer 像素实现的。


### 4\.2 VirtualBox


在 VirtualBox 中运行需要先将镜像文件转换为 vdi 格式



```
VBoxManage convertdd disk.img disk.vdi --format VDI
```

接着注册硬盘，并添加到虚拟机的 IDE 控制器中


[![](https://img2023.cnblogs.com/blog/2583637/202410/2583637-20241008155249276-635775318.png)](https://img2023.cnblogs.com/blog/2583637/202410/2583637-20241008155249276-635775318.png)


[![](https://img2023.cnblogs.com/blog/2583637/202410/2583637-20241008155413107-649719766.png)](https://img2023.cnblogs.com/blog/2583637/202410/2583637-20241008155413107-649719766.png)


在设置中关闭 EFI


[![](https://img2023.cnblogs.com/blog/2583637/202410/2583637-20241008154925704-762751986.png)](https://img2023.cnblogs.com/blog/2583637/202410/2583637-20241008154925704-762751986.png)


这样就可以正常启动了


[![](https://img2023.cnblogs.com/blog/2583637/202410/2583637-20241008155909652-40861102.png)](https://img2023.cnblogs.com/blog/2583637/202410/2583637-20241008155909652-40861102.png)


## 附录：完整代码


### write\_grub\_cfg.sh



> 别忘了加上可执行权限 `chmod +x write_grub_cfg.sh`



```
#!/bin/bash

KERNEL_NAME=$1
GRUB_CFG_PATH=$2

cat <<EOF > $GRUB_CFG_PATH
set timeout=20
set default=0

menuentry "MyOS" {
    multiboot2 /boot/$KERNEL_NAME
    boot
}
EOF
```

### Makefile



```
CC = gcc
LD = ld
CFLAGS = -m32 -fno-builtin -fno-stack-protector -nostartfiles
LDFLAGS = -Ttext 0x100000 -melf_i386 -nostdlib
KERNEL_NAME = kernel.bin
IMG_NAME = disk.img
IMG_SIZE = 64

all: img

$(KERNEL_NAME): boot.o kernel.o
	$(LD) $(LDFLAGS) $^ -o $@

boot.o: boot.S
	$(CC) -c $(CFLAGS) $< -o $@

kernel.o: kernel.c
	$(CC) -c $(CFLAGS) $< -o $@

.PHONY: img
img: $(IMG_NAME) $(KERNEL_NAME)
	sudo losetup -P /dev/loop0 $(IMG_NAME)
	sudo mount /dev/loop0p1 /mnt
	sudo ./write_grub_cfg.sh $(KERNEL_NAME) /mnt/boot/grub/grub.cfg
	sudo cp $(KERNEL_NAME) /mnt/boot/
	sudo umount /mnt
	sudo losetup -d /dev/loop0


$(IMG_NAME):
	dd if=/dev/zero of=$(IMG_NAME) bs=1M count=$(IMG_SIZE)
	parted -s $(IMG_NAME) mklabel msdos mkpart primary ext2 1MiB 100%
	sudo losetup -P /dev/loop0 $(IMG_NAME)
	sudo mkfs.ext2 /dev/loop0p1
	sudo mount /dev/loop0p1 /mnt
	sudo grub-install --target=i386-pc --boot-directory=/mnt/boot /dev/loop0
	sudo mkdir -p /mnt/boot/grub
	sudo umount /mnt
	sudo losetup -d /dev/loop0

qemu: img
	qemu-system-i386 -m 1G -hda $(IMG_NAME)

clean:
	sudo umount /mnt || true
	sudo losetup -d /dev/loop0 || true
	rm -f *.o $(KERNEL_NAME) $(IMG_NAME)

```

### boot.S



```
/*  boot.S - bootstrap the kernel */
/*  Copyright (C) 1999, 2001, 2010  Free Software Foundation, Inc.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, either version 3 of the License, or
 * (at your option) any later version.
 *
 * This program is distributed in the hope that it will be useful,
 * but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 * GNU General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program.  If not, see .
 */

#define ASM_FILE        1
#include "multiboot2.h"

/*  C symbol format. HAVE_ASM_USCORE is defined by configure. */
#ifdef HAVE_ASM_USCORE
# define EXT_C(sym)                     _ ## sym
#else
# define EXT_C(sym)                     sym
#endif

/*  The size of our stack (16KB). */
#define STACK_SIZE                      0x4000
        
        .text
        
multiboot_header:
        /*  Align 64 bits boundary. */
        .align  8
        /*  magic */
        .long   MULTIBOOT2_HEADER_MAGIC
        /*  ISA: i386 */
        .long   MULTIBOOT_ARCHITECTURE_I386
        /*  Header length. */
        .long   multiboot_header_end - multiboot_header
        /*  checksum */
        .long   -(MULTIBOOT2_HEADER_MAGIC + MULTIBOOT_ARCHITECTURE_I386 + (multiboot_header_end - multiboot_header))
entry_address_tag_start:        
        .align  8
        .short MULTIBOOT_HEADER_TAG_ENTRY_ADDRESS
        .short MULTIBOOT_HEADER_TAG_OPTIONAL
        .long entry_address_tag_end - entry_address_tag_start
        /*  entry_addr */
        .long multiboot_entry
entry_address_tag_end:
end_tag_start:
        .align  8
        .short MULTIBOOT_HEADER_TAG_END
        .short 0
        .long end_tag_end - end_tag_start
end_tag_end:
multiboot_header_end:
multiboot_entry:
        /*  Initialize the stack pointer. */
        movl    $(stack + STACK_SIZE), %esp

        /*  Reset EFLAGS. */
        pushl   $0
        popf

        /*  Push the pointer to the Multiboot information structure. */
        pushl   %ebx
        /*  Push the magic value. */
        pushl   %eax

        /*  Now enter the C main function... */
        call    EXT_C(cmain)

        /*  Halt. */
        pushl   $halt_message
        call    EXT_C(printf)
        
loop:   hlt
        jmp     loop

halt_message:
        .asciz  "Halted."

        /*  Our stack area. */
        .comm   stack, STACK_SIZE
```

### kernel.c



```
/*  kernel.c - the C part of the kernel */
/*  Copyright (C) 1999, 2010  Free Software Foundation, Inc.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, either version 3 of the License, or
 * (at your option) any later version.
 *
 * This program is distributed in the hope that it will be useful,
 * but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 * GNU General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program.  If not, see .
 */

#include "multiboot2.h"

/*  Macros. */

/*  Some screen stuff. */
/*  The number of columns. */
#define COLUMNS                 80
/*  The number of lines. */
#define LINES                   24
/*  The attribute of an character. */
#define ATTRIBUTE               7
/*  The video memory address. */
#define VIDEO                   0xB8000

/*  Variables. */
/*  Save the X position. */
static int xpos;
/*  Save the Y position. */
static int ypos;
/*  Point to the video memory. */
static volatile unsigned char *video;

/*  Forward declarations. */
void cmain (unsigned long magic, unsigned long addr);
static void cls (void);
static void itoa (char *buf, int base, int d);
static void putchar (int c);
void printf (const char *format, ...);

/*  Check if MAGIC is valid and print the Multiboot information structure
   pointed by ADDR. */
void
cmain (unsigned long magic, unsigned long addr)
{  
  struct multiboot_tag *tag;
  unsigned size;

  /*  Clear the screen. */
  cls ();

  /*  Am I booted by a Multiboot-compliant boot loader? */
  if (magic != MULTIBOOT2_BOOTLOADER_MAGIC)
    {
      printf ("Invalid magic number: 0x%x\n", (unsigned) magic);
      return;
    }

  if (addr & 7)
    {
      printf ("Unaligned mbi: 0x%x\n", addr);
      return;
    }

  size = *(unsigned *) addr;
  printf ("Announced mbi size 0x%x\n", size);
  for (tag = (struct multiboot_tag *) (addr + 8);
       tag->type != MULTIBOOT_TAG_TYPE_END;
       tag = (struct multiboot_tag *) ((multiboot_uint8_t *) tag 
                                       + ((tag->size + 7) & ~7)))
    {
      printf ("Tag 0x%x, Size 0x%x\n", tag->type, tag->size);
      switch (tag->type)
        {
        case MULTIBOOT_TAG_TYPE_CMDLINE:
          printf ("Command line = %s\n",
                  ((struct multiboot_tag_string *) tag)->string);
          break;
        case MULTIBOOT_TAG_TYPE_BOOT_LOADER_NAME:
          printf ("Boot loader name = %s\n",
                  ((struct multiboot_tag_string *) tag)->string);
          break;
        case MULTIBOOT_TAG_TYPE_MODULE:
          printf ("Module at 0x%x-0x%x. Command line %s\n",
                  ((struct multiboot_tag_module *) tag)->mod_start,
                  ((struct multiboot_tag_module *) tag)->mod_end,
                  ((struct multiboot_tag_module *) tag)->cmdline);
          break;
        case MULTIBOOT_TAG_TYPE_BASIC_MEMINFO:
          printf ("mem_lower = %uKB, mem_upper = %uKB\n",
                  ((struct multiboot_tag_basic_meminfo *) tag)->mem_lower,
                  ((struct multiboot_tag_basic_meminfo *) tag)->mem_upper);
          break;
        case MULTIBOOT_TAG_TYPE_BOOTDEV:
          printf ("Boot device 0x%x,%u,%u\n",
                  ((struct multiboot_tag_bootdev *) tag)->biosdev,
                  ((struct multiboot_tag_bootdev *) tag)->slice,
                  ((struct multiboot_tag_bootdev *) tag)->part);
          break;
        case MULTIBOOT_TAG_TYPE_MMAP:
          {
            multiboot_memory_map_t *mmap;

            printf ("mmap\n");
      
            for (mmap = ((struct multiboot_tag_mmap *) tag)->entries;
                 (multiboot_uint8_t *) mmap 
                   < (multiboot_uint8_t *) tag + tag->size;
                 mmap = (multiboot_memory_map_t *) 
                   ((unsigned long) mmap
                    + ((struct multiboot_tag_mmap *) tag)->entry_size))
              printf (" base_addr = 0x%x%x,"
                      " length = 0x%x%x, type = 0x%x\n",
                      (unsigned) (mmap->addr >> 32),
                      (unsigned) (mmap->addr & 0xffffffff),
                      (unsigned) (mmap->len >> 32),
                      (unsigned) (mmap->len & 0xffffffff),
                      (unsigned) mmap->type);
          }
          break;
        case MULTIBOOT_TAG_TYPE_FRAMEBUFFER:
          {
            multiboot_uint32_t color;
            unsigned i;
            struct multiboot_tag_framebuffer *tagfb
              = (struct multiboot_tag_framebuffer *) tag;
            void *fb = (void *) (unsigned long) tagfb->common.framebuffer_addr;

            switch (tagfb->common.framebuffer_type)
              {
              case MULTIBOOT_FRAMEBUFFER_TYPE_INDEXED:
                {
                  unsigned best_distance, distance;
                  struct multiboot_color *palette;
            
                  palette = tagfb->framebuffer_palette;

                  color = 0;
                  best_distance = 4*256*256;
            
                  for (i = 0; i < tagfb->framebuffer_palette_num_colors; i++)
                    {
                      distance = (0xff - palette[i].blue) 
                        * (0xff - palette[i].blue)
                        + palette[i].red * palette[i].red
                        + palette[i].green * palette[i].green;
                      if (distance < best_distance)
                        {
                          color = i;
                          best_distance = distance;
                        }
                    }
                }
                break;

              case MULTIBOOT_FRAMEBUFFER_TYPE_RGB:
                color = ((1 << tagfb->framebuffer_blue_mask_size) - 1) 
                  << tagfb->framebuffer_blue_field_position;
                break;

              case MULTIBOOT_FRAMEBUFFER_TYPE_EGA_TEXT:
                color = '\\' | 0x0100;
                break;

              default:
                color = 0xffffffff;
                break;
              }
            
            for (i = 0; i < tagfb->common.framebuffer_width
                   && i < tagfb->common.framebuffer_height; i++)
              {
                switch (tagfb->common.framebuffer_bpp)
                  {
                  case 8:
                    {
                      multiboot_uint8_t *pixel = fb
                        + tagfb->common.framebuffer_pitch * i + i;
                      *pixel = color;
                    }
                    break;
                  case 15:
                  case 16:
                    {
                      multiboot_uint16_t *pixel
                        = fb + tagfb->common.framebuffer_pitch * i + 2 * i;
                      *pixel = color;
                    }
                    break;
                  case 24:
                    {
                      multiboot_uint32_t *pixel
                        = fb + tagfb->common.framebuffer_pitch * i + 3 * i;
                      *pixel = (color & 0xffffff) | (*pixel & 0xff000000);
                    }
                    break;

                  case 32:
                    {
                      multiboot_uint32_t *pixel
                        = fb + tagfb->common.framebuffer_pitch * i + 4 * i;
                      *pixel = color;
                    }
                    break;
                  }
              }
            break;
          }

        }
    }
  tag = (struct multiboot_tag *) ((multiboot_uint8_t *) tag 
                                  + ((tag->size + 7) & ~7));
  printf ("Total mbi size 0x%x\n", (unsigned) tag - addr);
}    

/*  Clear the screen and initialize VIDEO, XPOS and YPOS. */
static void
cls (void)
{
  int i;

  video = (unsigned char *) VIDEO;
  
  for (i = 0; i < COLUMNS * LINES * 2; i++)
    *(video + i) = 0;

  xpos = 0;
  ypos = 0;
}

/*  Convert the integer D to a string and save the string in BUF. If
   BASE is equal to ’d’, interpret that D is decimal, and if BASE is
   equal to ’x’, interpret that D is hexadecimal. */
static void
itoa (char *buf, int base, int d)
{
  char *p = buf;
  char *p1, *p2;
  unsigned long ud = d;
  int divisor = 10;
  
  /*  If %d is specified and D is minus, put ‘-’ in the head. */
  if (base == 'd' && d < 0)
    {
      *p++ = '-';
      buf++;
      ud = -d;
    }
  else if (base == 'x')
    divisor = 16;

  /*  Divide UD by DIVISOR until UD == 0. */
  do
    {
      int remainder = ud % divisor;
      
      *p++ = (remainder < 10) ? remainder + '0' : remainder + 'a' - 10;
    }
  while (ud /= divisor);

  /*  Terminate BUF. */
  *p = 0;
  
  /*  Reverse BUF. */
  p1 = buf;
  p2 = p - 1;
  while (p1 < p2)
    {
      char tmp = *p1;
      *p1 = *p2;
      *p2 = tmp;
      p1++;
      p2--;
    }
}

/*  Put the character C on the screen. */
static void
putchar (int c)
{
  if (c == '\n' || c == '\r')
    {
    newline:
      xpos = 0;
      ypos++;
      if (ypos >= LINES)
        ypos = 0;
      return;
    }

  *(video + (xpos + ypos * COLUMNS) * 2) = c & 0xFF;
  *(video + (xpos + ypos * COLUMNS) * 2 + 1) = ATTRIBUTE;

  xpos++;
  if (xpos >= COLUMNS)
    goto newline;
}

/*  Format a string and print it on the screen, just like the libc
   function printf. */
void
printf (const char *format, ...)
{
  char **arg = (char **) &format;
  int c;
  char buf[20];

  arg++;
  
  while ((c = *format++) != 0)
    {
      if (c != '%')
        putchar (c);
      else
        {
          char *p, *p2;
          int pad0 = 0, pad = 0;
          
          c = *format++;
          if (c == '0')
            {
              pad0 = 1;
              c = *format++;
            }

          if (c >= '0' && c <= '9')
            {
              pad = c - '0';
              c = *format++;
            }

          switch (c)
            {
            case 'd':
            case 'u':
            case 'x':
              itoa (buf, c, *((int *) arg++));
              p = buf;
              goto string;
              break;

            case 's':
              p = *arg++;
              if (! p)
                p = "(null)";

            string:
              for (p2 = p; *p2; p2++);
              for (; p2 < p + pad; p2++)
                putchar (pad0 ? '0' : ' ');
              while (*p)
                putchar (*p++);
              break;

            default:
              putchar (*((int *) arg++));
              break;
            }
        }
    }
}
```

### multiboot2\.h



```
/*   multiboot2.h - Multiboot 2 header file. */
/*   Copyright (C) 1999,2003,2007,2008,2009,2010  Free Software Foundation, Inc.
 *
 *  Permission is hereby granted, free of charge, to any person obtaining a copy
 *  of this software and associated documentation files (the "Software"), to
 *  deal in the Software without restriction, including without limitation the
 *  rights to use, copy, modify, merge, publish, distribute, sublicense, and/or
 *  sell copies of the Software, and to permit persons to whom the Software is
 *  furnished to do so, subject to the following conditions:
 *
 *  The above copyright notice and this permission notice shall be included in
 *  all copies or substantial portions of the Software.
 *
 *  THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
 *  IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
 *  FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.  IN NO EVENT SHALL ANY
 *  DEVELOPER OR DISTRIBUTOR BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY,
 *  WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR
 *  IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
 */

#ifndef MULTIBOOT_HEADER
#define MULTIBOOT_HEADER 1

/*  How many bytes from the start of the file we search for the header. */
#define MULTIBOOT_SEARCH                        32768
#define MULTIBOOT_HEADER_ALIGN                  8

/*  The magic field should contain this. */
#define MULTIBOOT2_HEADER_MAGIC                 0xe85250d6

/*  This should be in %eax. */
#define MULTIBOOT2_BOOTLOADER_MAGIC             0x36d76289

/*  Alignment of multiboot modules. */
#define MULTIBOOT_MOD_ALIGN                     0x00001000

/*  Alignment of the multiboot info structure. */
#define MULTIBOOT_INFO_ALIGN                    0x00000008

/*  Flags set in the ’flags’ member of the multiboot header. */

#define MULTIBOOT_TAG_ALIGN                  8
#define MULTIBOOT_TAG_TYPE_END               0
#define MULTIBOOT_TAG_TYPE_CMDLINE           1
#define MULTIBOOT_TAG_TYPE_BOOT_LOADER_NAME  2
#define MULTIBOOT_TAG_TYPE_MODULE            3
#define MULTIBOOT_TAG_TYPE_BASIC_MEMINFO     4
#define MULTIBOOT_TAG_TYPE_BOOTDEV           5
#define MULTIBOOT_TAG_TYPE_MMAP              6
#define MULTIBOOT_TAG_TYPE_VBE               7
#define MULTIBOOT_TAG_TYPE_FRAMEBUFFER       8
#define MULTIBOOT_TAG_TYPE_ELF_SECTIONS      9
#define MULTIBOOT_TAG_TYPE_APM               10
#define MULTIBOOT_TAG_TYPE_EFI32             11
#define MULTIBOOT_TAG_TYPE_EFI64             12
#define MULTIBOOT_TAG_TYPE_SMBIOS            13
#define MULTIBOOT_TAG_TYPE_ACPI_OLD          14
#define MULTIBOOT_TAG_TYPE_ACPI_NEW          15
#define MULTIBOOT_TAG_TYPE_NETWORK           16
#define MULTIBOOT_TAG_TYPE_EFI_MMAP          17
#define MULTIBOOT_TAG_TYPE_EFI_BS            18
#define MULTIBOOT_TAG_TYPE_EFI32_IH          19
#define MULTIBOOT_TAG_TYPE_EFI64_IH          20
#define MULTIBOOT_TAG_TYPE_LOAD_BASE_ADDR    21

#define MULTIBOOT_HEADER_TAG_END  0
#define MULTIBOOT_HEADER_TAG_INFORMATION_REQUEST  1
#define MULTIBOOT_HEADER_TAG_ADDRESS  2
#define MULTIBOOT_HEADER_TAG_ENTRY_ADDRESS  3
#define MULTIBOOT_HEADER_TAG_CONSOLE_FLAGS  4
#define MULTIBOOT_HEADER_TAG_FRAMEBUFFER  5
#define MULTIBOOT_HEADER_TAG_MODULE_ALIGN  6
#define MULTIBOOT_HEADER_TAG_EFI_BS        7
#define MULTIBOOT_HEADER_TAG_ENTRY_ADDRESS_EFI32  8
#define MULTIBOOT_HEADER_TAG_ENTRY_ADDRESS_EFI64  9
#define MULTIBOOT_HEADER_TAG_RELOCATABLE  10

#define MULTIBOOT_ARCHITECTURE_I386  0
#define MULTIBOOT_ARCHITECTURE_MIPS32  4
#define MULTIBOOT_HEADER_TAG_OPTIONAL 1

#define MULTIBOOT_LOAD_PREFERENCE_NONE 0
#define MULTIBOOT_LOAD_PREFERENCE_LOW 1
#define MULTIBOOT_LOAD_PREFERENCE_HIGH 2

#define MULTIBOOT_CONSOLE_FLAGS_CONSOLE_REQUIRED 1
#define MULTIBOOT_CONSOLE_FLAGS_EGA_TEXT_SUPPORTED 2

#ifndef ASM_FILE

typedef unsigned char           multiboot_uint8_t;
typedef unsigned short          multiboot_uint16_t;
typedef unsigned int            multiboot_uint32_t;
typedef unsigned long long      multiboot_uint64_t;

struct multiboot_header
{
  /*  Must be MULTIBOOT_MAGIC - see above. */
  multiboot_uint32_t magic;

  /*  ISA */
  multiboot_uint32_t architecture;

  /*  Total header length. */
  multiboot_uint32_t header_length;

  /*  The above fields plus this one must equal 0 mod 2^32. */
  multiboot_uint32_t checksum;
};

struct multiboot_header_tag
{
  multiboot_uint16_t type;
  multiboot_uint16_t flags;
  multiboot_uint32_t size;
};

struct multiboot_header_tag_information_request
{
  multiboot_uint16_t type;
  multiboot_uint16_t flags;
  multiboot_uint32_t size;
  multiboot_uint32_t requests[0];
};

struct multiboot_header_tag_address
{
  multiboot_uint16_t type;
  multiboot_uint16_t flags;
  multiboot_uint32_t size;
  multiboot_uint32_t header_addr;
  multiboot_uint32_t load_addr;
  multiboot_uint32_t load_end_addr;
  multiboot_uint32_t bss_end_addr;
};

struct multiboot_header_tag_entry_address
{
  multiboot_uint16_t type;
  multiboot_uint16_t flags;
  multiboot_uint32_t size;
  multiboot_uint32_t entry_addr;
};

struct multiboot_header_tag_console_flags
{
  multiboot_uint16_t type;
  multiboot_uint16_t flags;
  multiboot_uint32_t size;
  multiboot_uint32_t console_flags;
};

struct multiboot_header_tag_framebuffer
{
  multiboot_uint16_t type;
  multiboot_uint16_t flags;
  multiboot_uint32_t size;
  multiboot_uint32_t width;
  multiboot_uint32_t height;
  multiboot_uint32_t depth;
};

struct multiboot_header_tag_module_align
{
  multiboot_uint16_t type;
  multiboot_uint16_t flags;
  multiboot_uint32_t size;
};

struct multiboot_header_tag_relocatable
{
  multiboot_uint16_t type;
  multiboot_uint16_t flags;
  multiboot_uint32_t size;
  multiboot_uint32_t min_addr;
  multiboot_uint32_t max_addr;
  multiboot_uint32_t align;
  multiboot_uint32_t preference;
};

struct multiboot_color
{
  multiboot_uint8_t red;
  multiboot_uint8_t green;
  multiboot_uint8_t blue;
};

struct multiboot_mmap_entry
{
  multiboot_uint64_t addr;
  multiboot_uint64_t len;
#define MULTIBOOT_MEMORY_AVAILABLE              1
#define MULTIBOOT_MEMORY_RESERVED               2
#define MULTIBOOT_MEMORY_ACPI_RECLAIMABLE       3
#define MULTIBOOT_MEMORY_NVS                    4
#define MULTIBOOT_MEMORY_BADRAM                 5
  multiboot_uint32_t type;
  multiboot_uint32_t zero;
};
typedef struct multiboot_mmap_entry multiboot_memory_map_t;

struct multiboot_tag
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;
};

struct multiboot_tag_string
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;
  char string[0];
};

struct multiboot_tag_module
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;
  multiboot_uint32_t mod_start;
  multiboot_uint32_t mod_end;
  char cmdline[0];
};

struct multiboot_tag_basic_meminfo
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;
  multiboot_uint32_t mem_lower;
  multiboot_uint32_t mem_upper;
};

struct multiboot_tag_bootdev
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;
  multiboot_uint32_t biosdev;
  multiboot_uint32_t slice;
  multiboot_uint32_t part;
};

struct multiboot_tag_mmap
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;
  multiboot_uint32_t entry_size;
  multiboot_uint32_t entry_version;
  struct multiboot_mmap_entry entries[0];  
};

struct multiboot_vbe_info_block
{
  multiboot_uint8_t external_specification[512];
};

struct multiboot_vbe_mode_info_block
{
  multiboot_uint8_t external_specification[256];
};

struct multiboot_tag_vbe
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;

  multiboot_uint16_t vbe_mode;
  multiboot_uint16_t vbe_interface_seg;
  multiboot_uint16_t vbe_interface_off;
  multiboot_uint16_t vbe_interface_len;

  struct multiboot_vbe_info_block vbe_control_info;
  struct multiboot_vbe_mode_info_block vbe_mode_info;
};

struct multiboot_tag_framebuffer_common
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;

  multiboot_uint64_t framebuffer_addr;
  multiboot_uint32_t framebuffer_pitch;
  multiboot_uint32_t framebuffer_width;
  multiboot_uint32_t framebuffer_height;
  multiboot_uint8_t framebuffer_bpp;
#define MULTIBOOT_FRAMEBUFFER_TYPE_INDEXED 0
#define MULTIBOOT_FRAMEBUFFER_TYPE_RGB     1
#define MULTIBOOT_FRAMEBUFFER_TYPE_EGA_TEXT     2
  multiboot_uint8_t framebuffer_type;
  multiboot_uint16_t reserved;
};

struct multiboot_tag_framebuffer
{
  struct multiboot_tag_framebuffer_common common;

  union
  {
    struct
    {
      multiboot_uint16_t framebuffer_palette_num_colors;
      struct multiboot_color framebuffer_palette[0];
    };
    struct
    {
      multiboot_uint8_t framebuffer_red_field_position;
      multiboot_uint8_t framebuffer_red_mask_size;
      multiboot_uint8_t framebuffer_green_field_position;
      multiboot_uint8_t framebuffer_green_mask_size;
      multiboot_uint8_t framebuffer_blue_field_position;
      multiboot_uint8_t framebuffer_blue_mask_size;
    };
  };
};

struct multiboot_tag_elf_sections
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;
  multiboot_uint32_t num;
  multiboot_uint32_t entsize;
  multiboot_uint32_t shndx;
  char sections[0];
};

struct multiboot_tag_apm
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;
  multiboot_uint16_t version;
  multiboot_uint16_t cseg;
  multiboot_uint32_t offset;
  multiboot_uint16_t cseg_16;
  multiboot_uint16_t dseg;
  multiboot_uint16_t flags;
  multiboot_uint16_t cseg_len;
  multiboot_uint16_t cseg_16_len;
  multiboot_uint16_t dseg_len;
};

struct multiboot_tag_efi32
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;
  multiboot_uint32_t pointer;
};

struct multiboot_tag_efi64
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;
  multiboot_uint64_t pointer;
};

struct multiboot_tag_smbios
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;
  multiboot_uint8_t major;
  multiboot_uint8_t minor;
  multiboot_uint8_t reserved[6];
  multiboot_uint8_t tables[0];
};

struct multiboot_tag_old_acpi
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;
  multiboot_uint8_t rsdp[0];
};

struct multiboot_tag_new_acpi
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;
  multiboot_uint8_t rsdp[0];
};

struct multiboot_tag_network
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;
  multiboot_uint8_t dhcpack[0];
};

struct multiboot_tag_efi_mmap
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;
  multiboot_uint32_t descr_size;
  multiboot_uint32_t descr_vers;
  multiboot_uint8_t efi_mmap[0];
}; 

struct multiboot_tag_efi32_ih
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;
  multiboot_uint32_t pointer;
};

struct multiboot_tag_efi64_ih
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;
  multiboot_uint64_t pointer;
};

struct multiboot_tag_load_base_addr
{
  multiboot_uint32_t type;
  multiboot_uint32_t size;
  multiboot_uint32_t load_base_addr;
};

#endif /*  ! ASM_FILE */

#endif /*  ! MULTIBOOT_HEADER */
```

## 参考文献


[Multiboot2 Specification version 2\.0](https://github.com)


[写操作系统 之 GRUB 到 multiboot ——从 multiboot 开始对接内核（转载）](https://github.com)


[\[操作系统原理与实现]Multiboot与GRUB\_multiboot2\-CSDN博客](https://github.com)


[grub2详解(翻译和整理官方手册) \- 骏马金龙 \- 博客园](https://github.com)


[用 GRUB 引导自己的操作系统\_切换grub以实现系统的正常引导\-CSDN博客](https://github.com)


[操作系统实现 \- 055 multiboot2 头\_哔哩哔哩\_bilibili](https://github.com)


[MBR和GPT\_mbr gap\-CSDN博客](https://github.com)




---



> 本文发布于2024年10月8日
> 
> 
> 最后修改于2024年10月9日


  * [通过GRUB Multiboot2引导自制操作系统](#%E9%80%9A%E8%BF%87grub-multiboot2%E5%BC%95%E5%AF%BC%E8%87%AA%E5%88%B6%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F):[樱花宇宙官网](https://yzygzn.com)
* [前言](#%E5%89%8D%E8%A8%80)
* [开发环境](#%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83)
* [1 Multiboot2 规范](#tid-jhr7xx)
* [1\.1 Multiboot2 header](#tid-a255zG)
* [1\.1\.1 Magic fields](#tid-ne8a2F)
* [1\.1\.2 Tags](#tid-F4JcfS)
* [1\.2 机器状态](#tid-ZYf5r8)
* [2 代码实现](#tid-YN3aCK)
* [2\.1 定义 Multiboot header](#tid-5yNTK7)
* [2\.2 启动入口](#tid-NyY35C)
* [2\.3 内核代码](#tid-5YEYjZ)
* [2\.4 代码构建](#tid-8iHHKk)
* [3 镜像制作](#tid-2scpis)
* [3\.1 创建镜像](#tid-fYhzRZ)
* [3\.2 安装 GRUB](#tid-rnByXE)
* [4 虚拟机运行](#tid-YFZGkf)
* [4\.1 QEMU](#tid-8ixWTm)
* [4\.2 VirtualBox](#tid-hSPBCP)
* [附录：完整代码](#%E9%99%84%E5%BD%95%E5%AE%8C%E6%95%B4%E4%BB%A3%E7%A0%81)
* [write\_grub\_cfg.sh](#write_grub_cfgsh)
* [Makefile](#makefile)
* [boot.S](#boots)
* [kernel.c](#kernelc)
* [multiboot2\.h](#multiboot2h)
* [参考文献](#%E5%8F%82%E8%80%83%E6%96%87%E7%8C%AE)

   \_\_EOF\_\_

   ![](https://github.com/ThousandPine)ThousandPine  - **本文链接：** [https://github.com/ThousandPine/p/18451835](https://github.com)
 - **关于博主：** \=v\=
 - **版权声明：** 本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com "BY-NC-SA") 许可协议。转载请注明出处！
 - **声援博主：** 如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。
     
