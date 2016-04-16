# 块设备驱动

### 块设备读写
块设备的读写接口是`ll_rw_block()`

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