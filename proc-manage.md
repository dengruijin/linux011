# *linux-0.11进程管理
linux0.11中一个进程有以下主要要素：
* 进程属性(pid,state,...)
* TSS段  用于保存进程上下文
* LDT段  
* 用户态栈  进程在用户态时使用
* 内核态栈  进程进入内核态时使用

__*linux0.11中，每一个进程都有自己的用户栈，内核栈和tss段*__
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
  位于sched.c,输入参数c,
        if ((--current->counter)>0) 
            return; //若当前进程时间片用完则调用schedule
        current->counter=0;
        if (!cpl) return; //cpl表示进程被中断时的特权级,0表示内核态
        schedule();//进入调度    
* __*schedule*__:位于sched.c,进程调度函数选取就绪进程中counter最大的进程来执行
* __*switch_to(n)*__:位于sched.h,n表示task数组的下标.造成进程切换的关键语句是
      "ljmp %0\n\t" // 执行长跳转至*&__tmp，造成任务切换。
ljmp tss_selector指令用于切换任务,执行该指令时CPU自动保存此刻的进程上下文，然后将tss_selector所指向的tss段中的上下文恢复给CPU,这样就完成了进程切换.  
**注意：**在linux高版本内核已经不采用这种切换方式了，而是将上下文保存在进程表中。  
还有一点要注意的是：  
假设当前进程A在执行，此时发生调度，执行下面指令：
      ljmp tss_B //从进程A切换到进程B执行
这时，**ljmp执行时保存的进程A上下文是处于内核态的**,恢复给CPU的进程B上下文也是内核态的，那用户态的上下文保存在哪呢？通过中断的现场保护保存在内核栈中，中断返回时就返回到用户态了.

### *fork创建进程  
  * find_empty_process
  * copy_process

### *进程0与进程1
进程0称为idle进程，进程1称为init进程




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
    