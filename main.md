# uCore Operating System Introduction

## 实验一：系统软件启动过程

### ex1
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
    这个文件设置512B的bootloader的后两个字节为0x55,0xAA, 使其成为有效的MBR
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

### ex2
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


### ex3
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

### ex4
#### bootmain.c加载内核详细过程
  1. 读入前8个扇区4KB到内存0x10000处, 检查ELF_MAGIC是否匹配, 获取program header第0项内存地址, 遍历所有program header, 每个program header对应一个section, 根据program header中的要加载到的内存地址, 大小和文件中便宜, 将每个section从文件读入到内存. 最后跳转到elf头中指定的的入口内存地址处. 自此, bootloader的工作就完成了, kernel接管.

### ex5
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

### ex6

IDT每个表项是一个门描述符, 占8字节. 0-15位是段内偏移低16位, 16-31位是段选择符(要装载到CS寄存器中), 48-63位是段内偏移高16位. 这48位全指针代表了ISR的地址.

## 实验2: 物理内存管理
实验一过后大家做出来了一个可以启动的系统，实验二主要涉及操作系统的物理内存管理。操作系统为了使用内存，还需高效地管理内存资源。在实验二中大家会了解并且自己动手完成一个简单的物理内存管理系统。

### pmm_init之前的内存布局
    - va-3G = la = pa(仅使用分段, 不使用分页)

### 物理内存管理使用到的数据结构
1. pde_t *boot_pgdir = NULL;
    - ucore之后一直使用的页目录表, 映射了3G(la)~3G+896M(la) => 0(ph)~896M(pa), 这样便于直接操控la即可直接操控物理内存(所以以后可以不区分物理地址和内核线性地址).
2. free_area_t free_area;
    - 维护一个连续空闲物理内存块的链表
    - .free_list, 空闲链表表头(表头不使用)
    - .nr_free, 当前空闲页数量
3. struct Page *pages;
    - 描述每个物理页的对象数组. 指向静态段(end是kernel文件结束的地址), 用来维护物理页的属性(主要是弥补页表的不足, 比如页表项只能标示可读可写等少数几个物理页的属性), 并且pages数组项的地址可以直接推算出该Page对象描述的物理页的物理地址(page2pa)和内核线性地址(page2kva).
        ```
        pages = (struct Page *)ROUNDUP((void *)end, PGSIZE);
        ```
    - Page.ref, 该页被应用的次数(引用是指有虚拟页映射到该物理页)
    - Page.flags
        - bit 0: preserved, 1代表不收pmm_manager管理的内存, 包括内核代码,bss
        - bit 1: 代表该页是连续空闲页中的第一页
    - Page.property: 如果是连续空闲页的第一页, 代表连续空闲页的数量
    - Page.page_link: 如果是连续空闲页的第一页, 该字段是连接了所有空闲块的链表.
4. struct pmm_manager *pmm_manager;
    - 物理内存管理器, 是保存了一些函数指针的结构体, 利用多态.
5. pte_t * const vpt = (pte_t *)VPT;
    - 页目录表的线性地址(页目录表一共4M, 正好占1024个页, 即占用一个页目录表项)
6. pde_t * const vpd = (pde_t *)PGADDR(PDX(VPT), PDX(VPT), 0);
    - vpt所在的页对应的页目录表项的线性地址
7. struct segdesc gdt\[\];
    - ucore之后一直使用的全局描述符表, 其中的段描述符对虚拟地址进行线性映射
8. struct taskstate ts = {0};
    - 供所有用户级进程使用的tss, 只使用了内核堆栈字段, 用于系统调用或者中断时切换堆栈使用

### pmm中的函数分析
1. alloc_pages, free_pages等函数内部关中断, 目的是在分配内存过程中加锁, 防止竞争.
1. pmm.c::page_init()
  - 计算最大可用内存(仅把首地址但是大小不超过KMEMSIZE(896MB)的内存块算作可用内存块, 最后一块内存块末尾的物理地址为maxpa)
  - 将kernel文件和pages数组占用的空间设为保留页, 不会被pmm分配出去
  - 为每一个可用物理内存块调用init_memmap, 从而通知pmm这些地址是可用的
1. pmm.c::boot_alloc_page()
  - 调用alloc_page分配一个物理页, 如果不成功则停止内核.
1. pmm.c::get_pte(pgdir, la, create)
  - 在pgdir中创建或查找la对应的pte(页表项). 如果la对应的页目录表项是无效的, 那么就为页表分配一个物理页, 初始化这个页表(全0), 并且把页目录表项填好(权限设置成用户可读写, 最宽松权限), 不过返回的ptep指向的pte可能还是不存在的(P位未设置). 当物理内存不足时返回NULL.
1. pmm.c::page_insert(pgdir, page, la, perm)
  - 创建la到page(代表一个物理页)的映射, 与get_pte不同的是, get_pte只是获取la对应的页表项(它只可能在页目录项不存在时创建页目录项, 不会创建页表项). 而page_insert要求la指向的虚拟页一定要映射到page代表的物理页, 而且指定好权限. 所以page_insert有可能由于page已经指向其他物理页了导致该物理页被释放, 然后重新映射给la.
1. pmm.c::boot_map_segment(pgdir, la, size, pa, perm)
  - 创建la到pa的映射, 并且赋给perm的权限. 和page_insert略有区别, boot_map_segment不维护page的引用计数, 不管该物理页是否正在使用, 直接把pa开始长度为size(以页对齐)的物理页映射到la. 这个函数仅仅在pmm.c的pmm_init中被用到, 映射了内核区内存(所以不需要引用计数).
### 实验实现细节

1. ex1. 主要实现default_init_memmap, default_alloc_pages, default_free_pages
  - default_init_memmap, 设置一些可用的内存页
  - default_alloc_pages, 分配n页内存, 具体算法是遍历当前可分配的连续内存块中找到第一块大于等于所需内存的, 分配给用户, 如果该内存块不是正好等于用户所需的大小, 那么, 在空闲列表中减小这个空闲内存块的大小, 并且修改该空闲块的首地址.
  - default_free_pages, 回收内存页重新设置为可分配状态, 如果恰好可以合并, 和前后的内存块合并(可能合并两次), 并且需要注意保持顺序.
2. ex2, ex3.
  - get_pte, 通过虚拟地址返回页表项的地址, 不存在可以选择创建, 需要递增Page的ref. 但是我认为用户级应用程序应该不可用这部分内存, 所以我觉得不应该有PTE_U属性.
  - page_remove_pte, 需要递减一下物理页对应的Page, 如果为0了需要释放物理页.
3. 如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事?
  - 修改链接脚本kernel.ld, 使内核加载地址为0x10000
  - 修改kern/init/entry.S
    ```
    # 如果va=pa, 那么这里重新加载GDT以建立C运行环境就没有必要了
    kern_entry:
        # reload temperate gdt (second time) to remap all physical memory
        # virtual_addr 0~4G=linear_addr\&physical_addr -KERNBASE~4G-KERNBASE
        lgdt REALLOC(__gdtdesc)
        movl $KERNEL_DS, %eax
        movw %ax, %ds
        movw %ax, %es
        movw %ax, %ss

        ljmp $KERNEL_CS, $relocated
    ```
  - 修改KERNBASE宏为0

### 几个遇到的问题和解决
1. 在pmm.c::page_init()中, 我们发现, 最后一个for循环中, 将e820获取的非保留的,并且包含在pages数组之后, 到896MB之间所有4KB内存页调用了init_memmap来将他们初始化成free page, 那么问题是, 难道内核代码区也变成了可分配内存区了吗, 这样不会导致内核代码被覆盖吗?

  - 研究pages这个地址的由来可以发现, 它被赋值为外部符号end, 并且4K对齐, 可以发现找遍了所有C代码和汇编代码, 都没有这个全局符号. 最后在链接脚本中可以找到这个符号, 它被定义为kernel文件的大小(最后一个段的下一个段(不存在的)的开始地址), 这样一来, kernel代码就还是保持不可分配的reserved状态, 不会被内存管理器分配出去.

2. 在lab2实验导引中描述的first fit内存分配算法中, 是用双向链表把空闲内存块按照空闲块第一页的首地址的顺序从小到大排列的. 所以我们应该在分配内存和释放内存的时候都要维护这个顺序, 在释放内存的时候, 实验的帮助注释中写了, 需要维护顺序的问题, 也有测释放内存后顺序的维护的测试用例. 但是分配内存的时候, 如果按照示例的写法也会导致顺序被打乱, 原因是在双向连表中删除了原来的Page后, 未将新Page插入到原来的Page位置, 而是直接放到链表头了. 所以在分配时候记下位置可以改正这个问题, 而且没有对应的测试用例.

3. 关于ex1 first fit内存分配算法的实现, 参考答案中, 将所有的Page对象全部加入了双向链表, 并且设置这些Page全部是reserved, 这样做是低效的, 因为如果在内存的地址较高的地方如果存在一块较大可用内存块, 这样, 分配内存的时候就要遍历这些没用的Page才能找到那块大内存块, 降低了效率. 我认为应该在双向链表中只维护每个内存块的第一页的Page对象, 并且用property记录下这块内存块的大小, 这样, 连续内存块的数量就和双向链表内部元素的个数相同了, 把双向链表中元素个数从三万多减少到几个.

4. boot_pgdir\[0\] = boot_pgdir\[PDX(KERNBASE)\]的作用
    ```
    pmm.c::pmm_init()
    ...
    boot_pgdir[PDX(VPT)] = PADDR(boot_pgdir) | PTE_P | PTE_W;
    boot_map_segment(boot_pgdir, KERNBASE, KMEMSIZE, 0, PTE_W);

    // 这一句
    boot_pgdir[0] = boot_pgdir[PDX(KERNBASE)];

    enable_paging();
    gdt_init();

    boot_pgdir[0] = 0;
    ```
    - 使内存布局(enable_paging后)变为virtual_addr 3G~3G+4M = linear_addr 0~4M = linear_addr 3G~3G+4M = phy_addr 0~4M;
    - 如果去掉这条语句, enable_paging到gdt_init完成之间这段时间的内存布局是va-3G=la (entry.S中设置的分段), la=pa+3G(enable_paging设置的分页), 也就是说va减了两次3G才得到pa;
    - 这显然会发生内存错误, 为了防止这个错误, 把la的前4M(包含了kernel代码使用的地址)直接映射到KERNBASE对应的物理地址(其实就是前4M物理地址), 这样的即可以访问前3G+4M的内存了(包含了kernel代码和堆栈).
    - 但是要注意这部分代码一旦访问超过了这部分的地址就会出错.
    - 完成后取消了0-4M的映射, 因为这部分以后是要给用户进程使用的

## 实验3, 虚拟内存管理

### vmm使用的数据结构
1. mm_struct
    - .mmap_list, 连接了很多个vma_struct, 每个vma_struct代表一个连续va内存块, 根据起始地址排序
    - .map_count, vma_struct的数量
    - .pgdir, 页目录表, 上面那些vma_struct对应的va都是指在这个页表中的va
    - .sm_priv, 给swap_manager使用的私有数据(这样可以保证swap_manager的可组装性)
    - 每个mm_struct包含一个页表, 可以有独立的内存映射(基本上一个mm_struct和一个进程相关联), 并且所有的mm_struct中的页表都将这个页表映射到自己的VPT这个位置(va).
    - vma_struct对应一段连续的va, 和这段va的映射. 虚拟内存不会在创建时在页表中被映射, 而是当第一次访问该虚拟内存时会根据对应的vma_struct创建对应的页表.
    - vmm.c:mm_map负责创建一段虚拟内存, 会通过参数vma_store返回这段虚拟内存对应的vma
1. vma_struct
    - vm_mm, 该vma属于的mm_struct
    - vm_start, vm_end, 标示一段vma
    - vm_flags, 读写执行
    - list_link, 对应mm_struct.mmap_list
1. 

### pmm中新增函数和vmm中的函数分析
1. pmm的实现解决了物理内存映射, 物理内存分配, 内核内存分配的问题, 但是没有解决如何分配除了KERNAL_BASE以上的内存的问题, 即用户空间的虚拟内存分配问题. vmm实现了用户进程地址空间的映射(每个mm_struct维护一个页表), 同时添加了虚拟内存交换到外存的功能.
1. pmm.c::pgdir_alloc_page(pgdir, la, perm)
  - 把la指向的虚拟页映射到物理地址(物理地址随意), 并且标记la这个物理页是可交换的
2. pmm.c::kmalloc/kfree
  - 基于alloc_pages/free_pages来实现分配页对齐的内存块, 会保证物理页连续, 不被交换到外部储存.
  - 这样非常浪费内存, 可以使用buddy system分配算法.
3. vmm.c::insert_vma_struct(mm, vma)
  - 在mm_struct.mmap_list链表中, 按照起始地址排序, 添加一个vma(vma的start,end,flags已经初始化好了), 检查vma之间是否有重叠, 有重叠则崩溃系统.
4. vmm.c::find_vma(mm, addr)
  - 根据vm得到vma, 现在cache中找, 找不到则遍历vma链表查找.
5. mm_destroy(mm)
  - 销毁mm_struct中所有vma, 并且销毁mm
6. do_pgfault(mm, errcode, addr)
  - 首先获得addr所在的vma, 如果没有一个vma对应addr, 那么addr是一个未分配的非法地址.
  - 然后通过vma检查权限
  - 如果情况是是分配了va但是没创建页表项(没给分配物理地址), 则创建页表项且把权限写到页表项中.(注意这里可执行权限其实是没法检查的, 需要nXe的支持. 因此ucore把所有可读的内存都当成是可执行的)
  - 如果是该页交换到了磁盘上, 则从磁盘交换回来, 得到了一个新的物理页(可能和之前的物理页不一样, 所以需要page_insert一下重新建立物理地址到va的映射).
  - 这里的swap_in实现很简单, 它假定了可用物理页足够, 直接分配物理页. 但是实际情况可能是物理页不够, 需要把一些物理页交换到磁盘上从而腾出一些物理页.
### 实验的实现

### 遇到的问题
1. 哪些内存是可以交换到外存的, 哪些不可以?
2. vma_struct中的flags是如何限制内存块读写执行权限的?
