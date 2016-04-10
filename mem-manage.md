# *linux-0.11内存管理
### *内存布局
linux0.11默认支持的最大物理内存是16MB，被划分成五大块：
* 内核映像(system模块占用),
* 高速缓冲区
* ROM BIOS
* RAMDISK(可选)
* 主内存区
![内存布局](file:///home/deng/pictures/linux0.11-mem-layout.png)

### *缺页异常处理

### *内存的申请与释放

### *写保护异常处理




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