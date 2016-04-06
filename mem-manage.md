# 内存管理
    #define LOW_MEM 0x100000
    #define PAGING_MEMORY (15*1024*1024)
    #define PAGING_PAGES (PAGING_MEMORY>>12)
    #define MAP_NR(addr) (((addr)-LOW_MEM)>>12)
    #define USED 100

    #define CODE_SPACE(addr) ((((addr)+4095)&~4095) < \
    current->start_code + current->end_code)

    static long HIGH_MEMORY = 0;

    #define copy_page(from,to) \
    __asm__("cld ; rep ; movsl"::"S" (from),"D" (to),"c" (1024))

    static unsigned char mem_map [ PAGING_PAGES ] = {0,};