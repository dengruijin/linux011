# linux-0.11实验

### 制作软盘映像文件
dd if=/dev/zero of=floppy.img bs=512 count=2880

### 查看软盘影像中的内容
sudo mount -t minix rootimage-0.12-20040306 /mnt -o loop

### 查看硬盘映像中的内容
查看分区信息
losetup /dev/loop1  hdc-011.img
fdisk /dev/loop1
losetup -d /dev/loop1

挂载分区
losetup -o 512 /dev/loop1 hdc-011.img #512是分区开始处字节偏移
mount -t minix /dev/loop1  /mnt

diff -r linux linux-mdf>dif.out

### Bochs安装
下载源码包：
http://nchc.dl.sourceforge.net/project/bochs/bochs/2.4.5/bochs-2.4.5.tar.gz  

      sudo tar zxvf bochs-2.4.5.tar.gz
      cd bochs-2.4.5/
      sudo apt-get install build-essential
      sudo apt-get install xorg-dev  
      sudo apt-get install libgtk2.0-dev 
      sudo ./configure --enable-debugger --enable-disasm  
      Makefile的LIBS后面添加：-lz -lrt -lm -lpthread 
      