# 实验五：用户进程管理

到这一章整个ucore系统的中断,内存管理,进程切换已经基本成型, 所以在这里写一个ucore系统的总结.
并且在ucore中加一些注释.

1. 中断如何实现
    1. IDT在kern/trap/vectors.S:\_\_vectors符号
    1. IDT中每一个ISR都在kern/trap/vectors.S:vector0-255
    1. 每一个ISR仅仅将中断号压栈(如果该中断没有错误码, 那么0入栈, 平衡堆栈), 然后跳转到kern/trap/trapentry.S:\_\_alltraps
    1. 发生软中断时CPU负责把ss,esp,cs,eip保存在内核堆栈, 并且根据tss切换堆栈, 根据描述符设置cs,eip
    1. \_\_alltraps负责保存剩余寄存器, e?x,e?i,original esp, ebp, ds,es,fs,gd, 加载内核代码段描述符到ds,es
    1. \_\_alltraps最终调用kernel/trap/trap.c:trap(tf)函数, tf就是用户态的执行上下文
    1. trap是发生中断后被调用的第一个C函数. 在这里备份了当前进程的tf, 主要是为了能够处理中断重入. 如果是内核中发生的中断, 直接调用trap_dispatch, 否则需要启动调度器, 调度下一个进程 (从这里可以看出在lab5中还不是进程之间不能抢占CPU, 必须进程主动进行系统调用才会调度其他进程)
    1. kern/trap/trap.c:trap_dispatch(struct trapframe \*tf). 这里switch语句根据不同的中断号来分发中断.

2. 系统调用的实现
    - kern/trap/trap.c:trap_dispatch中会当中断号等于T_SYSCALL时调用kern/syscall/syscall.c:syscall, 该函数从current全局变量中获取tf, 这个tf中的各个寄存器中保存了用户系统调用的5个参数, 用eax做返回值
    - 从syscalls这个数组根据系统调用号找到系统调用函数sys_xxx, sys_xxx只负责将 uint32_t数组形式的参数转换成正常的c函数的参数(也就是调用真正的系统调用do_xxx)

3. 物理内存的管理
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
        - 用户空间页表:
    - 分段机制
        - kern/mm/pmm.c:gdt_init中, 加载了GDT,TR. gdt只用了5项, 内核代码/数据段, 用户代码/数据段, TSS段, 而且TSS只设置了内核堆栈字段, 这在特权级切换时会用到, 其他字段都没用.
    - 物理内存pmm_manager.
        - 宏观上pmm_manager主要功能是接受内核提供的可用连续物理内存区间, 维护一个可用物理内存页的列表, 并且可以分配和回收这些物理内存页. 在`init_pmm_manager(); page_init();`之后我们就可以用alloc_page分配物理内存页.
    - 内核中物理内存的堆式管理, kmalloc: 分配内核堆内存, 依赖于pmm的alloc_pages分配物理页
    - 虚拟内存mm_struct
        - 每个mm_struct包含一个页表, 可以有独立的内存映射, 并且所有的mm_struct中的页表都将这个页表映射到自己的VPT这个位置(va).
        - vma_struct对应一段连续的va, 和这段va的映射. 虚拟内存不会在创建时在页表中被映射, 而是当第一次访问该虚拟内存时会根据对应的vma_struct创建对应的页表.
        - vmm.c:mm_map负责创建一段虚拟内存, 会通过参数vma_store返回这段虚拟内存对应的vma
    - 分配内存的几种方式
        1. kern/mm/pmm.c:alloc_pages(npages): 分配物理内存并且映射到内核空间. 只能分配内核空间的内存(3G~3G+896M)
        2.
        3.

4. 进程调度
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
