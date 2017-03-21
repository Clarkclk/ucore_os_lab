Lab1 实验报告
====

练习1
----
1.操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中
每一条相关命令和命令参数的含义,以及说明命令导致的结果)

生成ucore.img的makefile代码块如下所示，以注释的形式进行解释
```
# kernel
#设置kernel的各个参数，包括include和src文件夹，编译时的一些参数等。
KINCLUDE	+= kern/debug/ \
			   kern/driver/ \
			   kern/trap/ \
			   kern/mm/

KSRCDIR		+= kern/init \
			   kern/libs \
			   kern/debug \
			   kern/driver \
			   kern/trap \
			   kern/mm

KCFLAGS		+= $(addprefix -I,$(KINCLUDE))

#为了生成kernel，需要先生成一些依赖文件
$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))

#生成obj文件
KOBJS	= $(call read_packet,kernel libs)

# create kernel target

#设置kernel的生成函数
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

#生成kernel的具体过程和设置
#根据obj文件生成kernel
#启用输出和调用命令
#objdump反汇编生成kernelasm文件，并显示文件的符号表入口
$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

#调用kernel生成函数
$(call create_target,kernel)

# -------------------------------------------------------------------

# create bootblock

#从boot中提取c文件到bootfiles，并编译成obj文件
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

#设置bootblock生成函数
bootblock = $(call totarget,bootblock)

#生成bootblock的具体过程和设置
#需要调用obj文件和sign的生成函数来生成bootblock
#启用输出和调用命令
#使用objjump反汇编和bojcopy复制函数来生成asm和obj文件
#调用sign工具来处理bootblock.out，生成bootblock
$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

#调用bootblock生成函数
$(call create_target,bootblock)

# -------------------------------------------------------------------

# create 'sign' tools
#生成sign工具
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)

# -------------------------------------------------------------------

# create ucore.img
#设置ucore.img文件的生成函数
UCOREIMG	:= $(call totarget,ucore.img)

#生成ucore.img的具体过程
#根据kernel和bootblock文件生成ucore.img
#初始化ucore.img，设置成10000个块，每块大小为512字节，用0填充
#把bootblock中的内容写到第一个块
#把kernel中的内容写到第一个块后面的块
$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

#调用ucore.img的生成函数
$(call create_target,ucore.img)
```

而运行make "V="后，得到的编译输出分为如下几个部分：

-将各个C文件编译为obj文件，各文件编译代码与下面的代码类似。
```
+ cc kern/init/init.c
gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o
```
其中，“-ggdb”表示生成可供gdb使用的调试信息，“-m32”表示编译环境为32位，“-gstabs”表示生成stabs格式的调试信息，“-nostdinc”表示不使用标准库，“-fno-stack-protector”表示不生成用于检测缓冲区溢出的代码，“-I<dir>"表示添加文件的搜索路径

-将各个obj文件链接起来，产生kernel文件。代码如下：
```
+ ld bin/kernel
ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/readline.o obj/kern/libs/stdio.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/debug/panic.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/intr.o obj/kern/driver/picirq.o obj/kern/trap/trap.o obj/kern/trap/trapentry.o obj/kern/trap/vectors.o obj/kern/mm/pmm.o  obj/libs/printfmt.o obj/libs/string.o
```
其中，“-m elf_i386”表示elf_i386连接器，“-nostdlib”表示不是用标准库，“-T tools/kernel.ld”表示使用kernel.ld中的配置链接文件

-编译bootloader，并使用sign工具将bootloader补齐到512字节，代码如下所示：
```
+ cc boot/bootasm.S
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
+ cc boot/bootmain.c
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
+ cc tools/sign.c
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
+ ld bin/bootblock
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
'obj/bootblock.out' size: 488 bytes
build 512 bytes boot sector: 'bin/bootblock' success!
dd if=/dev/zero of=bin/ucore.img count=10000
10000+0 records in
10000+0 records out
5120000 bytes (5.1 MB) copied, 0.0629148 s, 81.4 MB/s
dd if=bin/bootblock of=bin/ucore.img conv=notrunc
1+0 records in
1+0 records out
512 bytes (512 B) copied, 0.000120466 s, 4.3 MB/s
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
146+1 records in
146+1 records out
74923 bytes (75 kB) copied, 0.000393448 s, 190 MB/s
```
首先将bootasm、bootmain、sign编译成obj文件，然后使用链接生成bootblock，并使用sign工具进行处理生成kernel。其中，“-N”表示设置text和data部分可读可写，“-Ttext 0x7c00”表示段起始地址为0x7c00，“if/of”表示从文件输入/输出到文件。

2.一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?

根据sign.c文件，对于主引导扇区的设置如下：
```
#大小为512字节
char buf[512];
#第510个字节是0x55
buf[510] = 0x55;
#第511个字节是oxAA
buf[511] = 0xAA;
```

练习2
----
1.从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行。

2.在初始化位置0x7c00 设置实地址断点,测试断点正常。

3.从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。

4.自己找一个bootloader或内核中的代码位置，设置断点并进行测试。

1.本问题参考了答案中的提示，更改gdbinit和makefile文件来进行单步跟踪和断点测试。

2.进行0x7c00断点测试时，gdb显示Breakpoint 2, 0x00007c00 in ?? ()
在gdb命令行中输入："x /10i 0x7c00",即可查看从0x7c00开始的10条汇编代码：
```
(gdb) x /10i 0x7c00
=> 0x7c00:      cli    
   0x7c01:      cld    
   0x7c02:      xor    %eax,%eax
   0x7c04:      mov    %eax,%ds
   0x7c06:      mov    %eax,%es
   0x7c08:      mov    %eax,%ss 
   0x7c0a:      in     $0x64,%al
   0x7c0c:      test   $0x2,%al
   0x7c0e:      jne    0x7c0a
   0x7c10:      mov    $0xd1,%al	   
```
证明断点正确。

3.区别：指令的书写形式上有所不同，除此之外基本相同。比如bootasm.S中move指令有movew和moveb两种形式，但在反汇编代码中只有mov指令；在bootasm.S中寄存器%ax对应反汇编代码中的%eax。

练习3
----
分析bootloader进入保护模式的过程。

大致分为4个部分：(具体代码块表述信息，源代码中给出的注释已经非常详细，就不再重复分析）

第一部分：初始化过程，重置寄存器值和flag值
-cli：使CPU不再接受外部中断
-cld：使CPU按从低到高地址序处理字符串
-将寄存器DS、ES、SS置0

第二部分：开启A20
+等待8042芯片的输入缓存为空
+向0x64端口发送0xd1,对P2端口写数据
+等待8042芯片的输入缓存为空
+向0x64端口发送0xDF，打开A20 Gate

第三部分：从实模式切换为保护模式
+初始化GDT表，将gdtdesc所指向的内容读入GDT表
+最低2位值为0x17,表明表的大小为24
+设置CR0第一位为1,开启保护模式
  
第四部分：保护模式初始化
+初始化各寄存器的值
+将栈顶设置为地址0X7c00
+调用bootmain

练习4
----
分析bootloader加载ELF格式的OS的过程

分为两个部分：
第一部分为读取硬盘，bootloader读取硬盘的函数是基于函数readsect实现的，大致过程如下所示：
```
/* readsect - read a single sector at @secno into @dst */
static void
readsect(void *dst, uint32_t secno) {
    //等待磁盘准备好
    // wait for disk to be ready
    waitdisk();

    outb(0x1F2, 1);                         // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    // wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4);
}
```
-读取0x1f7,等待磁盘准备;
-向0x1f2端口发送读取的扇区数
-0x1f3到0x1f6表示扇区偏移
-等待磁盘准备

第二部分为加载elf，大致过程如下：
-加载elf文件头
-将kernel载入到相应的地址
-跳到e_entry，调用kernel

练习5
----
我们需要在lab1中完成kdebug.c中函数print_stackframe的实现，可以通过函数print_stackframe来跟踪函数调用堆栈中记录的返回地址。

执行“make qemu”后，得到的输出结果如下所示：
```
ebp:0x00007b08 eip:0x001009a6 args:0x00010094 0x00000000 0x00007b38 0x00100092 
    kern/debug/kdebug.c:305: print_stackframe+21
ebp:0x00007b18 eip:0x00100c95 args:0x00000000 0x00000000 0x00000000 0x00007b88 
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b38 eip:0x00100092 args:0x00000000 0x00007b60 0xffff0000 0x00007b64 
    kern/init/init.c:48: grade_backtrace2+33
ebp:0x00007b58 eip:0x001000bb args:0x00000000 0xffff0000 0x00007b84 0x00000029 
    kern/init/init.c:53: grade_backtrace1+38
ebp:0x00007b78 eip:0x001000d9 args:0x00000000 0x00100000 0xffff0000 0x0000001d 
    kern/init/init.c:58: grade_backtrace0+23
ebp:0x00007b98 eip:0x001000fe args:0x001032fc 0x001032e0 0x0000130a 0x00000000 
    kern/init/init.c:63: grade_backtrace+34
ebp:0x00007bc8 eip:0x00100055 args:0x00000000 0x00000000 0x00000000 0x00010094 
    kern/init/init.c:28: kern_init+84
ebp:0x00007bf8 eip:0x00007d68 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
    <unknow>: -- 0x00007d67 --
```
输出结果与答案要求大致一致。最后一行中7bf8出现是因为：在bootblock.asm中，函数堆栈从ox7c00开始，而使用call bootmain转入bootmain函数，减去call指令后，bootmain的ebp为0x7bf8。而0x7d68出现原因：在bootblock.asm中，0x7d66为函数readseg()的最后一条指令，0x7d68为outw()的第一条指令，在调用过程中不会调用到0x7d67,所以显示为unknow。

练习6
---
请完成编码工作和回答如下问题：

1.中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

每个中断符表项占8个字节，其中2-3字节是段选择子，利用段选择子查找GDT获取基址，0-1字节和6-7字节拼成offset，两者联合便是中断处理程序的入口地址。

2.请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。

见代码trap.c

3.请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”。

见代码trap.c

练习7
---


