# 内存管理
用字节数组mem_map来记录主内存中内存页的状态

    // memory.c
    #define PAGING_MEMORY (15*1024*1024)
    #define PAGING_PAGES (PAGING_MEMORY>>12)
    #define USED 100
    static long HIGH_MEMORY = 0;
    static unsigned char mem_map [ PAGING_PAGES ] = {0,};