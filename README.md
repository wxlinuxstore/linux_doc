# Changes of board of ShiJiaZhuang 54

* address 0: change to 8M norflash (AM29LV641)
* address 0x20000000: change to 128M*2 norflash (S29GL01GP13TFIV10)
* ddr change(128M*2) is ok when use ddr phy configuration 1600
* uart debug port is changed to port 1


# 调试使用指北

## 串口连接

调试串口和grmon都是rs422，需要区分连接。我们把grmon的串口连接到了虚拟机，通过虚拟机驱动之。
而调试串口接到了windows主机中，并使用SecureCRT进行操作。

## 调试工具

我们将grmon调试工具进行了封装，封装后的脚本就是该目录下的`burn2run.sh`，使用方法为，
打开一个专门用于调试终端，根据需求输入如下命令：

```bash
./burn2run.sh 0 -n # 0表示使用/dev/ttyUSB0，如果grmon的串口是/dev/ttyUSB1，那么这里应该成1。
                   # -n表示不下载程序，只配置好硬件环境进入grmon终端
                   # -b表示下载flash_3883/burn/flash_burn到0x40000000，并运行之。
                   #   也就是说我们需要固话firmware时，简化了我们在grmon终端的繁琐输入。
                   # -r表示下载u-boot-2017.03/u-boot到0x40000000，并运行之。用途与-b同理。
```

## grmon命令

到目前阶段，我们经常使用的命令也就那么几个，`load`，`go`，`mem`等（一些`grmon`的调试命令也无法达到调试内核的目的）。具体使用方法可以参考使用文档，
或者在grmon终端下使用help命令去查看帮助。

## u-boot编译

u-boot代码使用的是本目录下的`u-boot-2017.03`，进入该目录运行`make -j3`（前提是已经将交叉编译链`sparc-linux-gcc`配置进了`PATH`）即可。

## 内核编译

内核源码为本目录下的`linux-4.9.54`，内核已经配置完善。进入内核目录后使用的编译命令为，

```bash
PATH=~/Workspace/sparc/3823/buildroot/output/host/usr/bin:$PATH LD_LIBRARY_PATH=$LD_LIBRARY_PATH:~/Workspace/sparc/3823/buildroot/output/host/usr/lib make ARCH=sparc CROSS_COMPILE=sparc-linux- uImage
```

这串很长的命令已经配置到zsh的alias中，你只需要输入`bsu -j3`，即可编译内核。一般无需重新配置内核，如果需要，可使用`bsk menuconfig`打开配置界面。

## 内核烧录

由于目前网口无法使用，烧录方式是通过460800 baud的串口通过kermit协议进行的。流程为

1. 通过grmon烧录启动u-boot，即`./burn2run 0 -r`。
2. u-boot命令行输入`loadb 0x42000000; bootm 0x42000000\n`启动kermit接收。
3. 通过相关串口工具传输内核uImage。如通过secureCRT中`Transfer`选项中的`send kermit...`。


## 目录介绍

```
burn2run.sh # 调试封装脚本
ddr3_settings.c # 54s可用的ddr phy等配置
flash_3883 # firmware固化方案
flash_test # norflash驱动及测试（包含54s用的两块flash，型号见本文开头）
grmon_configs # grmon启动硬件配置脚本
linux-4.9.54 # 适配的sparc内核源码
README.md # 本文
test_template # 裸机程序模板
uart_test # 简单的裸机uart测试
u-boot-2017.03 # 适配的u-boot源码
```

## 开发状态介绍

* 新板的硬件资源更新，如ddr容量，uart调试口等，已同步更新到u-boot中。
* 由于没有对端设备，`rapidio`的初始化暂时从内核中注释掉了。需要在接上对端设备后进行调试。
* `flash_test`中的flash驱动已经支持`0x20000000`的S29GL flash，但是0地址的AM29 flash还存在问题，需要进一步调试。
* 由于3883开机子启动地址为0（配置为非调试状态），所以firmware的固化方案，要在AM29 flash调通后进行整合修改（代码框架已经完善，如果flash驱动有问题，修改一下应该就可以了）。
* 除了i2c，uart，（这两个接口都是有相应的硬件验证过的）其它接口的Linux驱动需要根据54s的安排进一步调试验证。需要验证的模块有gpio（无复用的口为0-7），watchdog，spi，rapidio等
* 新板子上有两块3883，根据硬件工程师的需求调通另一块，（这两块对称，调通一块后另一块应该也是通的）
* 如果需要固化linux，那么需要调通`AM29 Flash`并移植到`u-boot`（它也是作为启动flash，u-boot保存启动，所以需要制定相应的存储地址方案），这块flash为8M，存放内核（+initramfs大约6M多）也是可行的。
* S29GL将作为Linux文件系统使用。需要移植到Linux。

