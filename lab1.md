# 实验一：系统软件启动过程

## ex1
#### ucore系统的构建过程

  1. ucore.img构建: Makefile中`$(UCOREIMG)` recipe在`bin/ucore.img`生成系统磁盘镜像. 它由`bin/bootblock`和`bin/kernel`经过填0对齐到10000个block构成, 默认block为512字节, 所以磁盘镜像大小为4.88MB.
  2. `bin/bootblock`构建:
  - `tools/function.mk`文件定义一些Makefile中的帮助函数,
    - `listf`是根据文件目录和文件名选中特定文件
    - `listf_cc`是只选中某目录下的.c和.S文件
    - `totarget`是生成参数1加上bin/前缀.
    - `toobj` 生成将参数1后缀名变为.o的函数
  - bootfiles
    利用这几个函数生成`bootfiles`变量, `bootfiles = $(call listf_cc,boot)`, 从而bootfile为boot目录下的所有.c,.S文件.
  - sign
    *******此文件运行于host, 是构建工具, 之后详细写
  - 构建过程, 我们把构建过程倒过来分析, 从链接开始, `make "V="`, 可以看到链接命令行如下. 不链接标准库, 入口点是bootasm.S中的`start`符号, 代码段从0x7C00开始, 即BIOS将磁盘第一个扇区默认加载的地址, 编译成i386架构的elf格式文件.

        ld -m elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o


	- .S和.c的编译过程, 命令行如下. 禁止包含builtin-function, 生成最多的调试信息给gdb, 32位ABI, 无默认包含头文件,  优化代码最小,

			`gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o`
	3. `bin/kernel`构建: 从构建过程反过来分析
	- 链接和生成调试符号(用sed去掉标题和符号的属性标志,大小等信息, 只留下地址和符号名), 生成脚本用的是tools/kernel.ld(****之后分析)

#### 标准MBR结构
| Address(Hex)    |  Address(Dec)   | Description | Length |
| ------------- |:-------------:| :-----:| -----: |
| 0000			| 0000 			| 代码区	  |440(最大446)|
|01B8|440|	選用磁碟標誌 | 4 |
|01BC|444|	一般為空值; 0x0000 |	2 |
|01BE|446|	标准MBR分区表规划（四个16 byte的主分区表入口）|	64 |
|01FE|510|	55h | 2 |
|01FF|511|	AAh | 2 |

## ex2
#### 用gdb跟踪qemu启动的第一条指令
  - CPU启动后设置$cs=0xf000, $ip=0x0xfff0, 即从内存0xffff0开始执行
  (用bochs模拟器可以自动区分实模式和保护模式, 无需配置, 而且bochs可以获取idtr,gdtr等寄存器的值,这个是gdb连接qemu调试无法做到的. 在Makefile中添加了启动bochs的recipe)
  - CPU上电启动后BIOS主要完成的工作:
    - 设置堆栈
    - 关中断
    - 关NMI
    - 设置A20(超过20位地址就回滚,为了兼容286以前cpu)
    - 调用很多函数(太多没法每个都看)自检并打印信息
    - 最后调用BIOS的0x19号中断读入启动设备上0柱面0磁头1扇区的512B字节到0x7c00
    - 移交控制权给0x7c00
  - bootasm.S主要是设置临时的GDT, 进入保护模式, 然后调用C函数bootmain
  - bootmain.c主要是从磁盘读入kernel, 它先读入kernel文件的头部到ELFHDR(0x10000), 然后根据头部的elf头信息, 将整个文件读入并移交控制权


## ex3
#### bootloader进入保护模式过程
  1. 进入保护模式需要先构建gdt, 设置gdtr, 然后long jump使用gdt中的描述符跳到32位地址, 然后在设置各个数据段包括堆栈段寄存器, 这部分内容在`boot/bootasm.S`文件的`start`符号后面的seta20之后
  ```
    lgdt gdtdesc
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
    ljmp $PROT_MODE_CSEG, $protcseg
  ```
  2. GDT内容在符号`gdt`处定义

  ```
  gdt:
    SEG_NULLASM                                     # null seg
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
    SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel

  gdtdesc:
    .word 0x17                                      # sizeof(gdt) - 1
    .long gdt                                       # address gdt
  ```
  根据80386规范, GDT第零项应为全零的无效描述符. 第一项为代码段描述符, 属性为可读可执行, 首地址0, 长度4G, 第二项为代码段描述符, 属行为可读可写, 首地址0, 长度4G. 被载入到gdtr的GDT描述符指定了GDT长度为3个描述符, 地址是`gdt`符号的值.

  3. 开启保护模式和到32位模式的跳转: 将CR0的最低1位置1表示开启保护模式, 紧接着使用刚建立好的GDT的代码段描述符long jump跳转到`protcseg`符号地址, `protcseg`中将ds,es,ss等数据段寄存器设置为指向GDT中的数据段描述符. 至此无论指令还是数据寻址都是段内偏移=线性地址=物理地址. 接着为了调用C函数必须设置ebp和esp, 从而能正常使用堆栈和访问局部变量, 这里把0-0x7c00作为C函数堆栈使用. 这样就可以直接调用C函数bootmain了.

## ex4
#### bootmain.c加载内核详细过程
  1. 读入前8个扇区4KB到内存0x10000处, 检查ELF_MAGIC是否匹配, 获取program header第0项内存地址, 遍历所有program header, 每个program header对应一个section, 根据program header中的要加载到的内存地址, 大小和文件中便宜, 将每个section从文件读入到内存. 最后跳转到elf头中指定的的入口内存地址处. 自此, bootloader的工作就完成了, kernel接管.

## ex5
#### 打印函数调用栈信息

  我们知道, c语言函数调用遵循c调用约定
  过程主要是
    0. 参数从右向左入栈
    1. eip入栈 (caller)
    2. ebp入栈 (callee)
    3. ebp = esp
  所以我们可以通过ebp来遍历函数调用链, 算法为
  初始: eip = $eip, ebp = $ebp, args = \*($ebp+8)
  迭代: eip = \*($ebp+4), ebp = \*$ebp, args = \*($ebp+8)

## ex6
