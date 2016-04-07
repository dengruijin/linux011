数据结构

    //sched.h
    struct task_struct
    struct tss_struct
    //head.h
    struct desc_struct
    //sched.c
    struct task_struct * task[NR_TASKS] = {&(init_task.task), };
进程创建

进程调度
    
    //sched.c
    void schedule(void)
    void sleep_on(struct task_struct **p)
    