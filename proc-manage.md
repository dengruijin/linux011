# *linux-0.11进程管理
linux0.11中一个进程有以下主要要素：
* 进程属性(pid,state,...)
* TSS段  用于保存进程上下文
* LDT段  
* 用户态栈  进程在用户态时使用
* 内核态栈  进程进入内核态时使用
__*linux0.11中，每一个进程都有自己的用户栈，内核栈和tss段*__

### *进程的内存结构


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
  进程发出fork系统调用后进入system_call，查系统调用表从而进入sys_fork,该函数在system_call.s中,sys_fork的主要工作如下：
  * find_empty_process   
  kernel/fork.c 135.取得一个可用的pid和空的task下标,可用的pid放在全局变量last_pid,task下标作为返回值返回
  * 将参数入栈后调用copy_process()
  * copy_process
  kernel/fork.c 68.输入参数是当前进程所有寄存器的值。该函数首先get_free_page()获得一页空闲内存,用来存放task_struct数据,填充task_struct内的数据,然后调用copy_mem复制页表
  * copy_mem
  kernel/fork.c 40. 设置ldt描述符分配线性地址空间,然后调用copy_page_tables复制页表
  * copy_page_tables
  mm/memory.c 150.   
  输入参数:
    * old_data_base,原进程数据段基址
    * new_data_base,新进程数据段基址
    * data_limit,段限长
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
  * 将新进程state设为TASK_RUNNING
  * 返回last_pid,即新进程的pid
#####fork过程总结：
进入int sys_fork()函数，首先获取一个空的task数组下标和可用的pid,然后以所有寄存器值为参数调用int copy_process(...)来初始化新进程的task_struct,新进程的tss.eax设为0,这就是为什么fork返回时子进程的返回值为0。随后为新进程划分线性地址空间(nr*64MB),复制老进程的页表给新进程，但页目录表只有一个，是所有进程公用的。最后在GDT中添加tss和ldt段的描述符，并把新进程的state设为TASK_RUNNING.到此新进程创建完毕并成为就绪状态。copy_process函数的返回值是子进程的pid,同时也是sys_fork和fork的返回值
---
### *进程0与进程1
进程0称为idle进程，进程1称为init进程
进程0的数据在sched_init()中被初始化，但执行move_to_user_mode()后才真正以进程0的身份运行



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
    