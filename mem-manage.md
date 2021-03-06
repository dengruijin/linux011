# *linux-0.11内存管理
### *物理内存布局
linux0.11默认支持的最大物理内存是16MB，被划分成五大块：
* 内核映像(system模块占用),不超过192KB
* 高速缓冲区。  
缓冲区的末端地址`BUF_END`是根据物理内存RAM大小而分配的,
  * 若`RAM >12MB`,则`BUF_END=4MB`
  * 若`RAM > 6MB`,则`BUF_END=2MB`
  * 若`RAM < 6MB`,则`BUF_END=1MB`

* ROM BIOS
* RAMDISK(可选)
* 主内存区， 这一块就是内存管理模块主要的用武之地了。  
![物理内存布局](./img/linux0.11-mem-layout.png)

### *线性地址空间布局
![线性地址空间](./img/liner-addr-layout.png)
### *内存的申请与释放
mem_map[]字节数组记录了主内存区中每一个物理页的使用情况，若mem_map[i]=0说明第i页是空闲的。
* __申请内存页__：`get_free_page(void)`  
从mem_map[]数组从后往前扫描，寻找值为0的项（空闲页），若找到了则将该项置1,计算出该页的首地址，并对该页内容清零，然后返回该页开始处的物理地址。
* __释放内存页__：`free_page(unsigned long addr)`  
计算出物理地址addr对应的页号，若mem_map[]对应项的值>0则将其减去1.
* __复制指定线性地址和长度对应的物理页和页表__：  
       //from，to是线性地址， size是要复制的字节数,fork时用到
       int copy_page_tables(unsigned long from,unsigned long to,long size)
* __释放指定线性地址和长度对应的物理页和页表__：  
       //from是线性地址， size是要释放的字节数,exit时用到
       int free_page_tables(unsigned long from,unsigned long size)
* __映射物理页至指定线性地址__：  
      //page是物理页首地址,address是线性地址
      unsigned long put_page(unsigned long page,unsigned long address)
* __申请并映射物理页值指定线性地址__：
      //缺页异常处理时会用到
      void get_empty_page(unsigned long address)
      //该函数的实现是对get_free_page和put_page的封装。
### *缺页&写保护异常处理
当CPU进行线性地址到物理地址的转换如果发现对应页目录项或页表项的**存在标志位为0**或者当前进程**没有权限访问该页**，就会发生页异常中断。发生页异常时CPU会将错误码压入栈中并将引起异常的线性地址放入CR2寄存器。
错误码占32位但只用到了低3位，意义如下：
>位2(U/S)：0表示在超级用户模式下发生异常，1表示在普通模式下发生异常  
>位1(W/R)：0表示读操作引起异常，1表示写操作引起异常  
>位0(P)：0表示页面不存在引起异常,1表示页面存在，是权限问题引起异常

页异常处理程序根据存在位(P)来判断是缺页异常还是写保护异常，然后调用相应的处理程序：
 * __缺页异常处理__：(实现**按需加载**)
       void do_no_page(unsigned long error_code,unsigned long address)
   缺页异常处理过程：
   * 首先判断是不是由于程序执行文件没有载入内存引起的，
   * 若不是则`get_empty_page`申请并映射一页内存即可返回，若是，则进入下一步。
   * 调用`share_page`尝试与其他进程共享该页，共享成功则返回，不成功则进入下一步
   * 申请一页内存，从磁盘加载执行文件的相应页面(通过地址可计算出要加载文件的哪一块)
 * __写保护异常处理__:（实现**写时复制**）
       void do_wp_page(unsigned long error_code,unsigned long address)
 处理过程：
    * 若该页地址大于1MB，且引用次数为1，说面不是共享造成的，只需要将该页的R/W位置1（可写），刷新TLB即可返回，否则进入下一步
    * 申请一页内存，若引起异常的地址大于1MB(说明引用次数>1)，将引用次数减1
    * 将新申请的页属性值设为7，映射给出错的线性地址，并刷新TLB，然后将原来页的内容复制到新页中，返回.
 
### *进程间共享内存share_page
若两个进程的可执行文件相同`(*p)->executable == current->executable`,则可以尝试共享内存页,share_page过程如下：  

    // address是进程中的逻辑地址，共享成功返回1失败返回0
    static int share_page(unsigned long address)
    {
        struct task_struct ** p;

        if (!current->executable)
            return 0;
        if (current->executable->i_count < 2)
            return 0;
        for (p = &LAST_TASK ; p > &FIRST_TASK ; --p) {
            if (!*p)
                continue;
            if (current == *p)
                continue;
            if ((*p)->executable != current->executable)
                continue;
            // 到此说明可以试一试与进程p共享
            if (try_to_share(address,*p))
                return 1;
        }
        return 0;
    }
    // address是当前进程中的逻辑地址,p指向的是提供共享页的进程
    // 共享成功返回1失败返回0
    static int try_to_share(unsigned long address, struct task_struct * p)
    {
        unsigned long from;
        unsigned long to;
        unsigned long from_page;
        unsigned long to_page;
        unsigned long phys_addr;

        from_page = to_page = ((address>>20) & 0xffc);
        /* 得到进程p的address对应的页目录地址 */
        from_page += ((p->start_code>>20) & 0xffc);
        // 得到当前进程的address对应的页目录地址
        to_page += ((current->start_code>>20) & 0xffc);
    /* is there a page-directory at from? */
        // 得到进程p的address对应的PDE内容
        from = *(unsigned long *) from_page;
        if (!(from & 1)) 
            return 0; // 存在位=0，无法共享
        from &= 0xfffff000; // 得到页表地址
        // 得到PTE地址
        from_page = from + ((address>>10) & 0xffc);
        phys_addr = *(unsigned long *) from_page; //// PTE内容
    /* is the page clean and present? */
        if ((phys_addr & 0x41) != 0x01)
            return 0; // 若Dirty位=1说明修改过，无法共享
        phys_addr &= 0xfffff000; // 得到物理地址
        if (phys_addr >= HIGH_MEMORY || phys_addr < LOW_MEM)
            return 0;
        to = *(unsigned long *) to_page; // 当前进程PDE内容
        if (!(to & 1)) { // 若存在位=0则要申请一页存放页表
            if ((to = get_free_page()))
                *(unsigned long *) to_page = to | 7;
            else
                oom();
        }
        to &= 0xfffff000; // 当前进程address对应的页表地址
        to_page = to + ((address>>10) & 0xffc); // PTE指针
        if (1 & *(unsigned long *) to_page) // PTE存在位=1，出错
            panic("try_to_share: to_page already exists");
    /* share them: write-protect */
        *(unsigned long *) from_page &= ~2; // 要共享了，设该页为只读
        // 填充PTE，共享成功
        *(unsigned long *) to_page = *(unsigned long *) from_page;
        invalidate(); // 刷新TLB
        phys_addr -= LOW_MEM;
        phys_addr >>= 12;
        mem_map[phys_addr]++; 该页引用数加1
        return 1; // 共享成功，返回1
    }

     


---
### 关键变量
用字节数组mem_map来记录1MB以上物理内存页的状态,其中的值表示该页被占用的次数，0表示该页空闲，当申请一页物理内存时该字节值增加1.
初始化时将mem_map[]所有项设为100(表示已占用)，然后将主内存区的mem_map[]设为0（空闲）。

    // memory.c
    #define PAGING_MEMORY (15*1024*1024)
    #define PAGING_PAGES (PAGING_MEMORY>>12) //3840个物理页
    #define USED 100
    static long HIGH_MEMORY = 0;
    static unsigned char mem_map [ PAGING_PAGES ] = {0,};
    
### 关键函数
    
    unsigned long get_free_page(void)
    void free_page(unsigned long addr)
    int free_page_tables(unsigned long from,unsigned long size)
    int copy_page_tables(unsigned long from,unsigned long to,long size)
    unsigned long put_page(unsigned long page,unsigned long address)
    // address是进程中的逻辑地址
    static int try_to_share(unsigned long address, struct task_struct * p)
    // address是进程中的逻辑地址
    static int share_page(unsigned long address)