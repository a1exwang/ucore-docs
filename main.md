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
    - .sm_priv, 给swap_manager使用的私有数据(这样可以保证swap_manager的可组装性), 现在使用的是fifo_swap, 这个字段用于保存一个链表, 链表中元素是将要交换到外存中的物理页(struct Page).
    - 每个mm_struct包含一个页表, 可以有独立的内存映射(基本上一个mm_struct和一个进程相关联), 并且所有的mm_struct中的页表都将这个页表映射到自己的VPT这个位置(va).
    - vma_struct对应一段连续的va, 和这段va的映射. 虚拟内存不会在创建时在页表中被映射, 而是当第一次访问该虚拟内存时会根据对应的vma_struct创建对应的页表.
    - vmm.c:mm_map负责创建一段虚拟内存, 会通过参数vma_store返回这段虚拟内存对应的vma
1. vma_struct
    - vm_mm, 该vma属于的mm_struct
    - vm_start, vm_end, 标示一段vma
    - vm_flags, 读写执行
    - list_link, 对应mm_struct.mmap_list
1. Page新增的两个字段
    - .pra_page_link
    - .pra_vaddr

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
7. swap manager
  - pmm.c::alloc_pages()中如果调用pm->alloc_pages失败, 即物理内存不足, 则调用swap_out将pra_list队列首的物理页交换到外存中.
  - 换出物理页之后, 在该物理页对应的va的pte上记录换出的地址(由于是256字节对齐, P为必为0, 访问该页产生页面错误)
  - swap_out, swap_in, 访问磁盘换出换入页
  - swap_manager需要实现的函数
    - map_swappable, 一个物理页可以被交换时被调用
    - swap_out_victim, 一个物理页被交换是被调用
### 初始化流程分析
  - 主要问题
    - 分页机制什么时候启动, 物理内存页如何分配, 内核使用了哪些数据结构, 如何知道哪些物理页分配了.
    - 初始化pmm时都做了什么, 初始化vmm都干了什么
    - 有哪些情况会切换页目录表(cr3), 也就是说切换了虚拟内存映射
    - pmm_manager, vmm_struct, vma_struct都是干嘛的
    - 在分页机制建立起来之前, 我们使用的地址必须是物理地址, 然而我们的链接脚本都假定了内核被加载到0xC0100000, 为什么函数调用不会失败呢? 也就是说pmm_init之前的代码中0xC0100000的地址为什么不会出错?
  - 从内核被加载到内存中到分页机制开启之前
    - 内核被加载到0x00100000(PA), 此时使用的段描述符的起始地址还是0, 即la=pa. 接下来在kern/init/entry.S:kern_entry中, 重新加载了GDT, 而这个临时的GDT的中所有描述符的起始地址都是-0xC0000000, 所以la-0xC0000000=pa, 这正好抵消了kernel文件中+0xC0000000的偏移量.
  - 分页机制
    - 内核页表: kern/mm/pmm.c:pmm_init是pmm初始化函数, 总体来说, 工作可以概括为初始化物理内存分配器, 建立一个新的页目录(这个新的页目录是用pmm_manager申请的空间), 新的页目录
    这里先调用page_init, 检查可用的物理内存, 将可用的物理内存全部线性映射到0xC0000000(注意这里页目录还是bootloader里面的), 同时也告诉了pmm_manager哪些物理内存是可用的哪些物理内存是给内核保留的(这部分主要是kernel文件, 包含代码和kernel的静态全局变量)
  - 分段机制
    - kern/mm/pmm.c:gdt_init中, 加载了GDT,TR. gdt只用了5项, 内核代码/数据段, 用户代码/数据段, TSS段, 而且TSS只设置了内核堆栈字段, 这在特权级切换时会用到, 其他字段都没用.
  - 物理内存pmm_manager.
    - 宏观上pmm_manager主要功能是接受内核提供的可用连续物理内存区间, 维护一个可用物理内存页的列表, 并且可以分配和回收这些物理内存页. 在`init_pmm_manager(); page_init();`之后我们就可以用alloc_page分配物理内存页.
  - 内核中物理内存的堆式管理, kmalloc: 分配内核堆内存, 依赖于pmm的alloc_pages分配物理页
  - 虚拟内存mm_struct
    - 每个mm_struct包含一个页表, 可以有独立的内存映射, 并且所有的mm_struct中的页表都将这个页表映射到自己的VPT这个位置(va).
    - vma_struct对应一段连续的va, 和这段va的映射. 虚拟内存不会在创建时在页表中被映射, 而是当第一次访问该虚拟内存时会根据对应的vma_struct创建对应的页表.
    - vmm.c:mm_map负责创建一段虚拟内存, 会通过参数vma_store返回这段虚拟内存对应的vma

### 实验内容
  - 问题1
  - 如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap_manager框架是否足以支持在ucore中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。并需要回答如下问题
    - 需要被换出的页的特征是什么？
     - 是由pgdir_alloc_page()分配的物理页
    - 在ucore中如何判断具有这样特征的页？
     - 调用了pgdir_alloc_page()就会直接调用map_swappable()加入到可交换的链表中
    - 何时进行换入和换出操作？
     - alloc_pages分不到内存的时候交换
    - 具体如何实现?
     - 在swap_out_victim()中, 遍历所有可交换的页(struct Page), 对于每个页, 通过Page找到页表项, 即
      ```
      pte = pa2kva(mm->pgdir\[PDX(page->pra_vaddr)\]\&0x3FF)\[PTX(page->pra_vaddr)\]
      dirty = PTE \& DIRTY
      accessed = PTE \& ACCESSED
      ```
      从dirty=0,accessed=0的, 然后是dirty=1,accessed=1, 然后等等..如果找到则返回该page. 当下次被换入时, 调用map_swappable加入可交换链表
### 遇到的问题
1. 哪些内存是可以交换到外存的, 哪些不可以?
  - 只需看哪里调用了map_swappable, 一共两处
    1. pgdir_alloc_page(), 也就是说pgdir_alloc_page分配的虚拟页对应的物理页可能被交换出去, 其余的kmalloc, alloc_pages都不会交换.
    2. do_page_fault(), 这里只是设置刚从外存交换到内存中的物理页为可交换, 本质上还是上一个函数添加的物理页.
2. vma_struct中的flags是如何限制内存块读写执行权限的?
    1. 在do_pgfault中, vma_struct的权限会被赋给页表项, 从而限制该页的权限, 但是页表项没有可执行这个标志, 所以ucore中可执行权限无法限制.

## 实验四：内核线程管理

### 内核线程在ucore中的实现方式
1. 概述
  - ucore中的内核线程是非抢占的, 必须线程主动返回才会切换到下一个线程
  - 每个内核线程包含独立的执行上下文(context), trapframe和堆栈 但是内存是共享的, 所有内核线程的页表都是使用的boot_pgdir, 段描述符都使用gdt中的内核代码段和内核数据段.
1. 数据结构分析: proc_struct结构体指针的组织
  - 系统中所有的进程的proc_struct以多种方式连接
    - 哈希表, pid => proc_struct, 这种方式可以快速通过pid来查找进程
    - 双向链表, 可以以创建的顺序便利
    - 每个proc_struct有一个parent指针指向创建者, 这导致所有的进程组织成以init进程为根的树形
  - proc_struct 结构体的字段分析
    - state, need_resched, 当前进程状态和是否需要调度(即被其他进程抢占)
    - pid, hash_link, list_link, pid是每个当前存在的进程唯一的(进程被销毁后,pid会被重新利用), hash_link是pid=>proc_struct的哈希表元素的链表, list_link是根据创建顺序维护的链表
    - runs, name, parent, 顾名思义
    - kstack, 进程在进入内核时(中断异常系统调用)使用的堆栈, 大小是2个物理页8KB, 在进入用户态之前被保存到tss中.
    - mm, cr3, 进程使用的虚拟内存(内核线程不使用)
    - flags, 暂时不使用
    - context, 保存进程在内核态的通用寄存器
    - tf, 保存进程进入内核时(中断异常系统调用时CPU自动保存到内核栈中, 而且tf就是指向内核栈栈底)的执行上下文

1. 内核线程的启动和切换过程
    1. do_fork中创建了描述proc的数据结构, 并且把该线程(进程)添加到进程链表中, 并且设置该进程可调度. kernel_thread函数中把进程的入口地址(kernel_thread_entry)存入进程context的eip中, 这样当进程恢复执行的时候就会从这个地址开始运行;
    2. kernel_thread_entry: 该符号在kern/process/entry.S中, 它用ebx传入真正的进程入口地址, 调用该入口函数, 当入口函数返回后, 调用proc_exit结束进程;
    3. cpu_idle中, schedule函数找到下一个可调度的进程, 调用proc_run;
    4. proc_run中, 修改cr0切换页表, 切换堆栈, 这些信息都存在proc_struct结构体中, 最后调用switch_to这个汇编函数
    5. switch_to 这个函数负责保存前一个进程的上下文, 并且设置下一个进程的上下文, 最后压入保存的eip, ret指令(模拟)函数返回, 跳转到了进程入口地址.
    6. 当进程返回, 根据2所述, 会调用proc_exit. 当前这个函数只会停止内核. 这个函数真正的实现应该是设置进程为ZOMBIE状态并且sched使其他进程得到控制权.
### 内核线程相关的函数分析
1. trapentry.S::forkrets
2. sched.c::schedule()
  - 关中断, 当前进程放弃执行, 调度下一个进程执行, 如果没有进程要执行, 则执行idle_proc
  - 下一个进程的选取: 在proc_list中找下一个RUNNABLE的进程, 最终调用proc_run函数切换进程.
3. proc.c
  1. proc_run()
    - 切换进程函数. 仅在内核态执行. 一个进程的内核态切换到另一个进程的内核态.
    - 进程刚被创建时, context.eip指向forkret, context.esp指向tf(这句话是冗余的, 因为forkret中直接设置esp=tf), 所有进程的内核态入口点都是内核态的forkret.
    - 进程暂停执行, 再继续执行时, context.eip
    - forkret调用forkrets, forkrets设置esp后跳转到__trapret. 此时esp=tf, 即内核堆栈已经模拟了一个中断发生时的状态, 所以只需按tf中顺序弹出寄存器并且iretd就可以跳转到真正的进程入口啦.
    - 内核线程的入口是kernel_thread_entry, 用edx传递参数, ebx传递真正的线程函数入口.
  3. kernel_thread(fn, args, clone_flag)
    - 设置内核线程的入口kernel_thread_entry和线程函数的入口ebx, 参数edx
  4. copy_thread(proc, esp, tf)
    - 设置堆栈的内核态上下文context, 和用户态上下文tf, 此处的esp是用户态堆栈地址, 对于内核线程来说, esp=NULL.
  5. do_fork(clone_flags, stack, tf)
    - 创建一个proc_struct, parent=current, 并且设置进程的执行上下文. 但是不真正执行进程, 仅仅添加到进程列表中.

### 思考题
1. 第1题: get_pid的实现. last_pid和next_safe两个static变量之间是一个未被使用的pid区间, 每次需要新生成一个pid时, last_pid++, 如果没有将区间长度变为0, 那么就成功获得了一个pid. 否则遍历pid列表, 从last_pid和MAX_PID之间找一个没有呗使用的pid, 并且维护last_pid和next_safe之间都是未被使用的pid. 这样可以是pid唯一. 但是当pid全部使用光了的时候, get_pid会死循环.

1. 第2题: context在switch.S文件的switch_to中被使用, 用来保存和恢复线程的执行上下文, trapframe是用来在线程返回内核(中断,系统调用..)的时候恢复内核执行上下文用的, 如果是用户级线程, 还会发生特权级切换, 会用到trapframe中保存的内核堆栈.

1. 第3题:
    - 创建了两个proc_struct, 即创建了两个内核进程, init和idle. 而idle的工作只是不停地放弃执行并且唤醒init.
    - 保存并且关闭中断的作用:
        1. 防止这段修改current代码重入, 如果之后改成了以时钟中断来出发进程切换的话, 这就相当于加了锁, 禁止其他进程运行
        2. 保存中断状态防止新进程中修改中断使能状态


## 实验五: 用户进程

到这一章整个ucore系统的中断,内存管理,进程切换已经基本成型, 所以在这里写一个ucore系统的总结.
并且在ucore中加一些注释.
### 数据结构

### 函数

### 过程分析
1. 中断如何实现
    1. IDT在kern/trap/vectors.S:\_\_vectors符号
    1. IDT中每一个ISR都在kern/trap/vectors.S:vector0-255
    1. 每一个ISR仅仅将中断号压栈(如果该中断没有错误码, 那么0入栈, 平衡堆栈), 然后跳转到kern/trap/trapentry.S:\_\_alltraps
    1. 发生软中断时CPU负责把ss,esp,cs,eip保存在内核堆栈, 并且根据tss切换堆栈, 根据描述符设置cs,eip
    1. \_\_alltraps负责保存剩余寄存器, e\*x,e\*i,original esp, ebp, ds,es,fs,gd, 加载内核代码段描述符到ds,es
    1. \_\_alltraps最终调用kernel/trap/trap.c:trap(tf)函数, tf就是用户态的执行上下文
    1. trap是发生中断后被调用的第一个C函数. 在这里备份了当前进程的tf, 主要是为了能够处理中断重入. 如果是内核中发生的中断, 直接调用trap_dispatch, 否则需要启动调度器, 调度下一个进程 (从这里可以看出在lab5中还不是进程之间不能抢占CPU, 必须进程主动进行系统调用才会调度其他进程)
    1. kern/trap/trap.c:trap_dispatch(struct trapframe \*tf). 这里switch语句根据不同的中断号来分发中断.

1. 系统调用的实现
    - kern/trap/trap.c:trap_dispatch中会当中断号等于T_SYSCALL时调用kern/syscall/syscall.c:syscall, 该函数从current全局变量中获取tf, 这个tf中的各个寄存器中保存了用户系统调用的5个参数, 用eax做返回值
    - 从syscalls这个数组根据系统调用号找到系统调用函数sys_xxx, sys_xxx只负责将 uint32_t数组形式的参数转换成正常的c函数的参数(也就是调用真正的系统调用do_xxx)

1. 进程调度
    - 创建一个内核线程的过程
        1. do_fork.
            1. 初始化proc_struct
            2. 用alloc_page分配内核栈, 也就是说所有进程都有一个内核栈, 即使是用户进程也会在中断时用到这个内核栈.
            3. 拷贝current的mm(也就复制了当前进程的内存映射), 内核线程以共享mm的方式, 用户进程则复制mm.
            4. tf参数可以指定中断返回时如何恢复执行上下文, 即可以指定进程入口地址.
            5. 把新建的proc_struct和当前进程建立父子进程关系.
            6. 分配一个唯一pid.
    - 一个内核线程拥有的资源:
        1. 和所有其他内核线程共享的页目录表
        2. 使用pmm分配的内核栈
        3. 唯一pid
    - 创建一个用户进程
        1. do_fork创建新的内核线程
        2. 在新的内核线程中系统调用do_execve->load_icode
        3. load_icode
            1. 接受参数binary, size, 这是要加载的elf文件在内存中的地址
            1. 创建mm_struct, 复制boot_pgdir从而创建一个新的页目录表
            2. 解析elf文件, 将每个section加载到elf文件中指定的va(这些va都是在新页表中的)
            3. 建立一个4页的用户栈, 栈顶为0xC0000000
            4. 新页表加载到cr3, 用户地址空间可用
            5. 设置tf, 其中保存了中断返回时恢复的执行上下文, 都是用户态的.

    - 启动内核线程过程中堆栈的使用情况
        '''
        位置			             堆栈
        do_fork系统调用会设置proc的返回地址是fork_ret, do_fork系统调用返回
        switch_to_1		         idle's kernel stack, 这个栈会被保存在from中
        switch_to_2		         to.esp	= proc.tf
        forkret			           to.esp	= proc.tf
        forkrets		           proc.tf = proc.kstack + KSTACKSIZE - sizeof(struct trapframe)
        \_\_trapret		         proc.tf 		0 (中断返回)
        kernel\_thread\_entry	 proc.tf + xx 这里开始tf不维护了, 给内核线程当做线程栈使用, 由于kstack是用alloc_page分配的,不是和proc_struct保存在一起的, 所以不维护这个栈不会导致其他数据被破坏
        '''
    - 启动用户线程过程中堆栈使用情况
        位置                    堆栈
        fork, sys_fork          caller's kernel stack(idle或者其他线程的内核栈)
        proc_run                设置了进程的内核栈(proc.kstack+KSTACKSIZE), 放在tss中, 用于用户态进程转到内核中时使用
        syscall(execv)          ..
        do_execve
        load_icode              tf中设置用户堆栈USTACKTOP(0xC0000000)
        iret                    切换到上面设置的用户栈
        \_start                 用户栈
        syscall                 切换到tss中指定的内核栈()
