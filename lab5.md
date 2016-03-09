# 实验五：用户进程管理

到这一章整个ucore系统的中断,内存管理,进程切换已经基本成型, 所以在这里写一个ucore系统的总结.
并且在ucore中加一些注释.

1. 中断如何实现
    1. IDT在kern/trap/vectors.S:\_\_vectors符号
    1. IDT中每一个ISR都在kern/trap/vectors.S:vector0-255
    1. 每一个ISR仅仅将中断号压栈(如果该中断没有错误码, 那么0入栈, 平衡堆栈), 然后跳转到kern/trap/trapentry.S:\_\_alltraps
    1. 发生软中断时CPU负责把ss,esp,cs,eip保存在内核堆栈, 并且根据tss切换堆栈, 根据描述符设置cs,eip
    1. \_\_alltraps负责保存剩余寄存器, e\*x,e\*i,original esp, ebp, ds,es,fs,gd, 加载内核代码段描述符到ds,es
    1. \_\_alltraps最终调用kernel/trap/trap.c:trap(tf)函数, tf就是用户态的执行上下文
    1. trap是发生中断后被调用的第一个C函数. 在这里备份了当前进程的tf, 主要是为了能够处理中断重入. 如果是内核中发生的中断, 直接调用trap_dispatch, 否则需要启动调度器, 调度下一个进程 (从这里可以看出在lab5中还不是进程之间不能抢占CPU, 必须进程主动进行系统调用才会调度其他进程)
    1. kern/trap/trap.c:trap_dispatch(struct trapframe \*tf). 这里switch语句根据不同的中断号来分发中断.

2. 系统调用的实现
    - kern/trap/trap.c:trap_dispatch中会当中断号等于T_SYSCALL时调用kern/syscall/syscall.c:syscall, 该函数从current全局变量中获取tf, 这个tf中的各个寄存器中保存了用户系统调用的5个参数, 用eax做返回值
    - 从syscalls这个数组根据系统调用号找到系统调用函数sys_xxx, sys_xxx只负责将 uint32_t数组形式的参数转换成正常的c函数的参数(也就是调用真正的系统调用do_xxx)



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
