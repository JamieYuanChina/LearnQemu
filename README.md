Qemu应用   
QEMU 模拟A9开发板  

一、在Deepin 中编译Qemu4，   
1、下载Qemu4源码   
wget https://download.qemu.org/qemu-4.0.0.tar.xz   
2、解压缩   
tar xvJf qemu-4.0.0.tar.xz   
3、安装依赖库   
sudo apt-get install libpixman-1-dev   
4、配置编译选项   
./configure --enable-kvm --target-list=arm-softmmu,aarch64-softmmu,i386-softmmu,x86_64-softmmu   
注释：target-list指定需要编译的target(guest)，arm-softmmu表示要编译system mode的QEMU，arm-linux-user表示要编译user mode的QEMU，--enable-vnc 表示支持VNC显示，具体target可以使用./configure --help查看    
5、编译   
make -j8   
6、安装   
sudo make install   
7、文件路径   
安装完成后可执行命令在/usr/local/bin下面，配置文件在/usr/local/share/qemu下   
通过apt-get命令安装的旧版本目录在/usr/bin下。   
8、相关资料   
https://www.cnblogs.com/BinBinStory/p/7618303.html   
https://blog.csdn.net/weixin_33851429/article/details/85959566   
二、创建磁盘并启动(x86_64)   
1、创建磁盘   
qemu-img create -f qcow2 deepin.img 10G   
2、启动虚拟机   
qemu-system-x86_64 -m 2048 -enable-kvm deepin.img -cdrom ./deepin-live-system-2.0-amd64.iso   
三、图形方式管理   
命令行启动虚拟机比较繁琐，适合开发者，但对于普通用户来说，采用图形界面管理虚拟机则更为方便。采用图形界面管理QEMU虚拟机需要安装virt-manager：   
$sudo apt-get install virt-manager -y   
安装完成后用root用户启动virt-manager：（可以使用sudo passwd root设置root密码）   
$su -   
#virt-manager   
四、常见的处理器架构   
架构   实体     虚拟硬件      OS   
X86    PC       virtualbox     Deepin/Windows   
Arm    imx6ul  Qemu          Uboot/Kernel   
RiscV    
五、模拟Arm架构   
常见的Arm开发   
mcimx6ul-evk(A7)、sabrelite(A9)、vexpress-a9   

下面以vexpress A9为例进行开发   
1、下载并安装arm平台交叉编译工具   
sudo apt-get install gcc-arm-linux-gnueabi   
完成后在/usr/bin下就会有arm-linux-gnueabi-gcc等工具了   
2、下载编译 u-boot   
①方法一   
wget ftp://ftp.denx.de/pub/u-boot/u-boot-2016.09.tar.bz2   
tar jxvf u-boot-2016.09.tar.bz2   
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- vexpress_ca9x4_defconfig   
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-   
qemu-system-arm -M vexpress-a9 -kernel u-boot -nographic -m 512M   
②方法二   
从uboot的git仓库下载uboot   
git clone git://git.denx.de/u-boot.git   
执行编译   
export ARCH=arm   
export CROSS_COMPILE=arm-linux-gnueabi-   
make clean   
make vexpress_ca9x4_defconfig   
提示bison: not found   
sudo apt-get install bison   
提示flex: not found   
sudo apt-get install flex   
make vexpress_ca9x4_defconfig   
make -j8   
执行仿真u-boot   
qemu-system-arm -M vexpress-a9 -m 256 -kernel u-boot -nographic   
3、linux内核   
从kernel的git仓库下载kernel   
git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git kernel   
也可以送清华服务器下载   
wget https://mirrors.tuna.tsinghua.edu.cn/kernel/v5.x/linux-5.2.tar.xz   
执行编译   
export ARCH=arm   
export CROSS_COMPILE=arm-linux-gnueabi-   
make clean   
make vexpress_defconfig   
make menuconfig   
去掉System Type 把 Enable the L2x0 outer cache controller 取消，否则qemu起不来   
make -j8   
执行仿真kernel，这里必须加入dtb不然仿真失败   
qemu-system-arm -M vexpress-a9 -m 512M -kernel arch/arm/boot/zImage -dtb arch/arm/boot/dts/vexpress-v2p-ca9.dtb -nographic -append "console=ttyAMA0"   
4、文件系统   
从busybox的git仓库下载busybox   
git clone git://busybox.net/busybox.git   
git clone https://github.com/mirror/busybox.git   
执行编译   
make defconfig   
sudo apt-get install libncurses5-dev   
make menuconfig   
make CROSS_COMPILE=arm-linux-gnueabi- -j8   
make install   
获取etc配置文件   
wget http://files.cnblogs.com/files/pengdonglin137/etc.tar.gz   
制作rootfs   
#!/bin/bash   
sudo rm -rf rootfs   
sudo rm -rf tmpfs   
sudo rm -f a9rootfs.ext3   
sudo mkdir rootfs    
sudo cp _install/* rootfs/ -raf   
mkdir -p rootfs/{lib,proc,sys,tmp,root,var,mnt}   
sudo cp -arf /usr/local/gcc-arm-none-eabi/arm-none-linux-gnueabi/lib rootfs/   
sudo cp etc rootfs/ -arf   
sudo rm rootfs/lib/*.a   
sudo mkdir -p rootfs/dev/   
sudo mknod rootfs/dev/tty1 c 4 1   
sudo mknod rootfs/dev/tty2 c 4 2pro   
sudo mknod rootfs/dev/tty3 c 4 3   
sudo mknod rootfs/dev/tty4 c 4 4   
sudo mknod rootfs/dev/console c 5 1   
sudo mknod rootfs/dev/null c 1 3   
sudo dd if=/dev/zero of=a9rootfs.ext3 bs=1M count=32   
sudo mkfs.ext3 a9rootfs.ext3   
sudo mkdir -p tmpfs   
sudo mount -t ext3 a9rootfs.ext3 tmpfs/ -o loop   
sudo cp -r rootfs/* tmpfs/   
sudo umount tmpfs   
执行仿真，这里必须加入dtb不然仿真失败   
qemu-system-arm -M vexpress-a9 -m 512M -kernel arch/arm/boot/zImage -dtb arch/arm/boot/dts/vexpress-v2p-ca9.dtb -nographic -append "root=/dev/mmcblk0 rw console=ttyAMA0" -sd a9rootfs.ext3   

参考资料：   
http://www.bubuko.com/infodetail-1955476.html   
https://www.cnblogs.com/pengdonglin137/   
https://www.veryarm.com/65016.html   
注释：在Qemu工具中，都有一个以w结尾的命令行工具，这个工具与不带w的区别在于：带w的命令行在一个窗口中分离并运行，所以不会在背景中留下一个命令行窗口、这对于喜欢打包Qemu的人来说很有用。   


