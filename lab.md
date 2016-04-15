# linux-0.11实验

### 制作软盘映像文件
dd if=/dev/zero of=floppy.img bs=512 count=2880
sudo mount -t minix rootimage-0.12-20040306 /mnt -o loop
