数据结构

    //sched.h
    struct task_struct
    //sched.c
    struct task_struct * task[NR_TASKS] = {&(init_task.task), };
进程创建

进程调度