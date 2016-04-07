数据结构

    //sched.h
    struct task_struct
    struct tss_struct
    //head.h
    struct desc_struct
    //sched.c
    struct task_struct * task[NR_TASKS] = {&(init_task.task), };
    union task_union
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
    