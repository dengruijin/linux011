# 块设备驱动

## 块设备读写
### 读写接口函数
块设备的读写接口是`ll_rw_block()`
输入参数：
>rw 表示读写命令 READ 或 WRITE  
    `#define READ 0`  
    `#define WRITE 1`  
    `#define READA 2`  	
    `#define WRITEA 3`  
>bh 高速缓冲块指针

      void ll_rw_block(int rw, struct buffer_head * bh)
      {
          unsigned int major;

          if ((major=MAJOR(bh->b_dev)) >= NR_BLK_DEV ||
          !(blk_dev[major].request_fn)) {
              printk("Trying to read nonexistent block-device\n\r");
              return;
          }
          make_request(major,rw,bh);
      }
当内核需要读写块设f备时传入适当参数调用ll_rw_block()即可。如bread中的调用：`ll_rw_block(READ,bh);`。在ll_rw_block中即调用make_request来根据参数把读写请求封装成request结构。
### 读写请求的封装
linux用struct request来封装一个块设备读写请求:  

    struct request {
        int dev;		/* -1 if no request */
        int cmd;		/* READ or WRITE */
        int errors;
        unsigned long sector;
        unsigned long nr_sectors;
        char * buffer;
        struct task_struct * waiting;
        struct buffer_head * bh;
        struct request * next;
    };

全局的request数组来存放request:  

      //ll_rw_block.c
      struct request request[NR_REQUEST];
        
并将其加入到request链表中