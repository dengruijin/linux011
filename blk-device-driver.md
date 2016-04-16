# 块设备驱动

## 块设备读写
### 读写接口函数
块设备的读写接口是`ll_rw_block()`
* __输入参数：__  
>rw 表示读写命令 READ 或 WRITE  
    `#define READ 0`  
    `#define WRITE 1`  
    `#define READA 2`  	
    `#define WRITEA 3`  
>bh 高速缓冲块指针  

* __函数体：__

      void ll_rw_block(int rw, struct buffer_head * bh)
      {
          unsigned int major;

          if ((major=MAJOR(bh->b_dev)) >= NR_BLK_DEV ||
          !(blk_dev[major].request_fn)) { // request_fn即do_XX_request函数指针
              printk("Trying to read nonexistent block-device\n\r");
              return;
          }
          make_request(major,rw,bh);
      }
当内核需要读写块设f备时传入适当参数调用ll_rw_block()即可。如bread中的调用：`ll_rw_block(READ,bh);`。在ll_rw_block中即调用make_request来根据参数把读写请求封装成request结构。
### 读写请求的封装
* __请求结构体__  
linux-0.11用`struct request`来封装一个块设备读写请求,并用一个全局数组来存放所有的request:  

    // blk.h
    struct request {
        int dev;		/*设备号 -1 if no request */
        int cmd;		/* READ or WRITE */
        int errors;
        unsigned long sector;
        unsigned long nr_sectors;
        char * buffer;
        struct task_struct * waiting;
        struct buffer_head * bh;
        struct request * next; // 用来形成请求链表
    };
    // ll_rw_block.c  NR_REQUEST=32 
    // 全局的request数组来存放request:
    struct request request[NR_REQUEST];
* __封装函数__  

      static void make_request(int major,int rw, struct buffer_head * bh)
      {
          struct request * req;
          int rw_ahead;
          // ......参数及锁检查......      
      repeat:
          if (rw == READ)
              req = request+NR_REQUEST;
          else
              req = request+((NR_REQUEST*2)/3);
          /* *###获得空闲request */
          while (--req >= request)
              if (req->dev<0)
                  break;      
          if (req < request) {
              // 获取失败则睡眠等待,唤醒后重新找
              goto repeat;
          }
      /* fill up the request-info, and add it to the queue */
          req->dev = bh->b_dev;
          req->cmd = rw;
          req->errors=0;
          req->sector = bh->b_blocknr<<1;
          req->nr_sectors = 2;
          req->buffer = bh->b_data;
          req->waiting = NULL;
          req->bh = bh;
          req->next = NULL;
          add_request(major+blk_dev,req);
      }
      
该函数从全局request数组中取得一个空闲的项
  

### 块设备的请求队列
* __基本结构__  
linux-0.11用主设备号为索引的块设备表来索引每一种设备的请求操作函数`request_fn`和当前正在处理的请求`current_request`:  

    // ll_rw_block.c  32, NR_BLK_DEV=7
    struct blk_dev_struct blk_dev[NR_BLK_DEV]
    // blk.h 45. 块设备结构体: 
    struct blk_dev_struct {
        void (*request_fn)(void);
        struct request * current_request;
    };
`current_request`指针和request中的`next`共同为一种块设备构成了请求链表，current_request指向该链表的头。  
request_fn的复制分别在`hd_init()`、`floppy_init()`和`rd_init()`中

* __队列的增长__  



