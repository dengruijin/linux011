# *linux-0.11进程管理
linux0.11中一个进程有以下主要要素：
* __*进程属性*__(pid,state,...)
* __*TSS段*__  用于保存进程上下文，它是被包含在PCB中的
* __*LDT段*__  一个表，存放着进程代码段和数据段描述符，它也是被包含在PCB中的
* __*用户态栈*__  进程在用户态时使用
* __*内核态栈*__  进程进入内核态时使用
__*linux0.11中，每一个进程都有自己的用户栈，内核栈和tss段*__

### *进程的内存结构
一个进程关键的内存图像包括PCB(即task_struct)、页表、内核栈、用户栈、代码段和数据段。  
>**PCB**,在fork创建进程时动态分配一页内存给**PCB**和**内核栈**，PCB占据该页内存的低端地址，内核栈则占据高端地址；TSS段和LDT段都放在PCB中，作为PCB的一部分，而不仅仅是一个指针；  
>**页表**所占空间也是fork时动态分配的；  
>**用户栈**处于进程数据段，从fork来看，子进程的**用户栈**与父进程相同，但是通过写时复制可以使他们指向不同物理地址，且在do_execv()时将对堆栈进行调整。（进程0和进程1的用户栈比较特殊,后文将介绍）  
>**代码段和数据**的地址是相互重叠的，只是描述符不一样，在linux0.11分给每个进程64MB线性地址块作为代码段和数据段，所以对4G线性地址来说最多支持64个进程。

### *关键数据
* linux0.11的PCB用结构体task_struct来表示，存放了一个进程的各种信息  
包括pid,state,priority(优先级),counter(剩余时间片),tss,ldt等...
* 全局变量task[NR_TASKS]数组包含所有PCB的指针

        // sched.c Line65
        struct task_struct * task[NR_TASKS] = {&(init_task.task), };

* ldt和tss
每创建一个进程就会在GDT中填充相应的ldt和tss段描述符
* 内核栈
每个进程的内核栈与PCB位于同一内存页中，初始状态esp指向页面末尾地址处。  
当进程从用户态进入内核态(如发生中断或系统调用)时，CPU从tss中取得ss0和esp0并使用它们(内核栈)作为当前堆栈指针，然后将用户态的ss和esp,eflags,cs,eip等保存在内核栈中。

### *进程调度
每10ms发生一次时钟中断，执行的函数如下：
* __*timer_interrupt*__:时钟中断处理程序的入口，在system_call.s中 ，它将jiffies加1后调用sched.c的do_timer函数
* __*do_timer(cpl)*__:
  位于sched.c,
  
          if ((--current->counter)>0) 
              return; //若当前进程时间片用完则调用schedule
          current->counter=0;
          if (!cpl) return; //cpl表示进程被中断时的特权级,0表示内核态
          schedule();//进入调度    
* __*schedule*__:位于sched.c,进程调度函数选取就绪进程中counter最大的进程来执行
* __*switch_to(n)*__:位于sched.h,n表示task数组的下标.造成进程切换的关键语句是

        "ljmp %0\n\t" // 执行长跳转至*&__tmp，造成任务切换。
        
`ljmp tss_selector`指令用于切换任务,执行该指令时CPU自动保存此刻的进程上下文，然后将`tss_selector`所指向的tss段中的上下文恢复给CPU,**ldtr寄存器也是上下文的一部分**(CPU自动将tss中的ldt项自动加载ldtr)这样就完成了进程切换.  
__注意:__在linux高版本内核已经不采用这种切换方式了，而是将上下文保存在进程表中。  
还有一点要注意的是：  
假设当前进程A在执行，此时发生调度，执行下面指令：

        ljmp tss_B //从进程A切换到进程B执行
这时，**ljmp执行时保存的进程A上下文是处于内核态的**,恢复给CPU的进程B上下文也是内核态的，那用户态的上下文保存在哪呢？通过中断的现场保护保存在内核栈中，中断返回时就返回到用户态了.

### *fork创建进程
  进程发出`fork()`系统调用后进入`system_call`，查系统调用表从而进入`sys_fork`,该函数在system_call.s中,`sys_fork`的主要工作如下：
  * __*find_empty_process*__  
  kernel/fork.c 135. 取得一个可用的pid和空的task下标,可用的pid放在全局变量`last_pid`, task下标作为返回值返回
  * 将参数入栈后调用`copy_process()`
  * __*copy_process*__  
  kernel/fork.c 68. 输入参数是当前进程所有寄存器的值。该函数首先`get_free_page()`获得一页空闲内存,用来存放task_struct数据,填充`task_struct`内的数据,然后调用`copy_mem`复制页表
  * __*copy_mem*__  
  kernel/fork.c 40. 设置ldt描述符分配线性地址空间,然后调用`copy_page_tables`复制页表
  * __*copy_page_tables*__
  mm/memory.c 150.   
  输入参数:  
> `old_data_base`, 原进程数据段基址  
> `new_data_base`, 新进程数据段基址  
> `data_limit`, 段限长   
 
  该函数复制原进程的页表   

        int copy_page_tables(unsigned long from,unsigned long to,long size)
        {
            unsigned long * from_page_table,* to_page_table;
            unsigned long this_page, nr;
            unsigned long * from_dir, * to_dir;
            // from和to必须是4MB的倍数
            if ((from&0x3fffff) || (to&0x3fffff))
                panic("copy_page_tables called with wrong alignment");
            //from_dir和to_dir分别指向相应的PDE的指针, 即*from_dir表示PDE内容
            from_dir = (unsigned long *) ((from>>20) & 0xffc);
            to_dir = (unsigned long *) ((to>>20) & 0xffc);
            // 这个运算是进1取整，即末尾不足4MB的按4MB计算,得到的size等于要复制的页表数目(即PDE数目)
            size = ((unsigned) (size+0x3fffff)) >> 22;
            // 循环复制每一个要复制的页表
            for( ; size-->0 ; from_dir++,to_dir++) {
                if (1 & *to_dir)
                    panic("copy_page_tables: already exist");
                if (!(1 & *from_dir))
                    continue;
                // 从PDE的获得页表地址(该地址是4KB对齐的,所以低12位为0)
                from_page_table = (unsigned long *) (0xfffff000 & *from_dir);
                // get_free_page()申请一页空闲内存用于存放新进程的一个页表
                if (!(to_page_table = (unsigned long *) get_free_page()))
                    return -1;	/* Out of memory, see freeing */
                // 将该页表基址或上属性位，赋值给PDE
                *to_dir = ((unsigned long) to_page_table) | 7;
                // from==0 说明老进程是task0(limit=640KB),只需复制160项PTE,即映射640KB物理内存
                nr = (from==0)?0xA0:1024;
                // 循环复制页表中的每一项(PTE)
                for ( ; nr-- > 0 ; from_page_table++,to_page_table++) {
                    this_page = *from_page_table;
                    if (!(1 & this_page))
                        continue;
                    // 新进程的这一页设为只读
                    this_page &= ~2; // R/W位清0，表示只读
                    *to_page_table = this_page;
                    // LOW_MEM=1MB,task0不会进入if里面
                    if (this_page > LOW_MEM) {
                        // 将原进程的该页也设为只读
                        *from_page_table = this_page;
                        // 减去LOW_MEM再除以4096才是该物理页对应的mem_map数组下标
                        this_page -= LOW_MEM;
                        this_page >>= 12;
                        // 对页面的引用记录加1
                        mem_map[this_page]++;
                    }
                }
            }
            invalidate();  //刷新页表高速缓存TLB
            return 0;
        }
  * 页表复制成功之后，对老进程中打开的文件，把文件打开次数加1
  * 在GDT中配置新进程的tss和ldt描述符
  * 将新进程`state`设为`TASK_RUNNING`
  * 返回`last_pid`,即新进程的pid  
#### fork过程总结:

      某进程正在用户态运行，此时一个fork调用，进入到了内核态，进入`int sys_fork()`函数，首先获取一个空的task数组下标和可用的pid,然后以所有寄存器值为参数调用`int copy_process(...)`来初始化新进程的`task_struct`,新进程的`tss.eax`设为0,这就是为什么fork返回时子进程的返回值为0。随后为新进程划分线性地址空间`(nr*64MB)`,复制老进程的页表给新进程，但页目录表只有一个，是所有进程公用的。最后在GDT中添加tss和ldt段的描述符，并把新进程的state设为`TASK_RUNNING`.到此新进程创建完毕并成为就绪状态。`copy_process`函数的返回值是子进程的pid,同时也是`sys_fork`和`fork`的返回值.有一点必须清楚的是，整个进程创建过程都是在父进程的内核态下完成的，也就是说父亲是在进入内核态之后，给自己拍了一张照片，照着照片的样子创建了子进程，创建完了，父进程就沿着中断返回的路子返回到用户态。而子进程还没被调度，依然是那张照片的样子放在就绪队列等着被调度。 所以当子进程被`schedule`调度执行时，它是从那张照片的样子(内核态)开始执行的，幸运的是，`schedule`也是处于内核态的，这个子进程就沿着schedule返回的路子返回到了用户态,这时候子进程就不是一张静止的照片了，而是开始按照动起来。这就是一个完整的进程创建过程。

### *进程0与进程1
进程0称为idle进程，进程1称为init进程
进程0的数据在sched_init()中被初始化，但执行move_to_user_mode()后才真正以进程0的身份运行.进程0和进程1与其他进程相比比较特殊,他们的代码段和数据段长度为640KB，而不是64MB，执行过`do_execv`的进程和他产生出的后代进程才拥有64MB地址空间，linux0.11中最先执行`execve`的是2号进程(task2), `do_execv`会改变进程的内存结构和页表。进程0和进程1的用户态堆栈也比较特殊，进程0的用户栈是在head.s时初始化的，task1通过fork也和task0有相同的用户栈,但当task1写内存时将发生写时复制，分配新的内存页，所以当task1要使用堆栈的时候将复制当前堆栈所在页到新申请的属于自己的内存页中。为了使task1创建时有一个干净的用户栈,task0并没有使用用户栈.

### *加载可执行文件:do_execve()
    int do_execve(unsigned long * eip,long tmp,char * filename, \
	char ** argv, char ** envp)
do_execve()可以加载文件系统上的可执行文件至进程空间，然后执行该文件.输入参数：

  >**eip**: 指向内核栈中eip的指针,这个eip是进程进入内核态之前的  
  >**filename**: 要执行文件的文件名，注意个文件可能是二进制文件也可能是脚本文件，两种文件处理方式是不同的  
  >**argv**: 传递给执行文件的参数列表,  
  >**envp**: 传递给执行文件的环境变量  
    
**这些参数除了eip外都是用户空间的地址，所以涉及到用户空间和内核空间的参数复制。** 内核限定这些参数占用空间不能超过32个页(128KB).

该函数执行过程如下：(`sh_bang`初值=0)
1. 获得文件的inode:  
       inode=namei(filename)
2. 检查是否有权限执行inode对应的文件,若无权限则返回错误
3. 取出执行文件头部数据,判断是否为脚本文件(文件开头为'#!'就认为是脚本文件)  
4. **如果不是脚本文件，跳至步骤6**。如果是脚本文件则读取与`#!`同一行的脚本解释器名称(如/bin/bash)和参数(如果有)。然后`if (sh_bang++ == 0)`则调用`copy_string()`将`argv`和`envp`指定的用户空间字符串拷贝到新申请的主内存中, 拷完之后再把`#!`后面指定的脚本解释器名称和参数(如果有)放到`argv`和`envp`的前面(主内存区)。这时主内存区的参数就包含了脚本解释器名称和参数以及`argv`和`envp`指定的数据，例如：    
        bash -iarg1 -iarg2  test.sh -arg1 -arg2 HOME=/
5. 获取解释器inode并**跳转到步骤2**:  
        inode=namei(interp)
        goto restart_interp;
6. 至此已经得到了要执行的二进制程序的程序头结构,根据头结构判断该文件是否符合内核要求，不合要求则返回错误。
7. `if (!sh_bang)`则调用`copy_string()`将`argv`和`envp`指定的用户空间字符串拷贝到新申请的主内存中。（sh_bang==0说明步骤4和5没有执行，也就没有从用户空间拷贝参数，所以if条件成立则需要拷贝参数)
        if (!sh_bang) {
                p = copy_strings(envc,envp,page,p,0);
                p = copy_strings(argc,argv,page,p,0);
                if (!p) { // 内存不足
                    retval = -ENOMEM;
                    goto exec_error2;
                }
        }


---

数据结构

    //sched.h
    struct task_struct
    struct tss_struct
    //head.h
    struct desc_struct
    
    //sched.c
    struct task_struct * task[NR_TASKS] = {&(init_task.task), };
    long volatile jiffies=0;
    union task_union {
        struct task_struct task;
        char stack[PAGE_SIZE];
    };
进程创建
    
    //system_call.s
    sys_fork
    
    //fork.c
    int find_empty_process(void)
    int copy_process(int nr,long ebp,long edi,long esi,long gs,long none,
		long ebx,long ecx,long edx,
		long fs,long es,long ds,
		long eip,long cs,long eflags,long esp,long ss)
     
进程调度
    
    //sched.c
    void schedule(void)
    void sleep_on(struct task_struct **p)
    void wake_up(struct task_struct **p)
