# *linux-0.11进程管理
linux0.11中一个进程有以下要素：
* 进程属性
* 

linux0.11的PCB用结构体task_struct来表示，存放了一个进程的各种信息，全局变量task[NR_TASKS]数组包含所有PCB

    // sched.c Line65
    struct task_struct * task[NR_TASKS] = {&(init_task.task), };
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
    