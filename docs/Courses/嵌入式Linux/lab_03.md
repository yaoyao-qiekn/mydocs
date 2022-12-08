# lab 03
# 0. 使用 qemu 模拟器搭建 ARM 运行环境

**环境**
- parallels 18
- Ubuntu 20.04
- u-boot-2018.11.tar.bz2

**资源文件链接**

- linux 内核 `6.0.11`：https://www.kernel.org/
- busybox `1.35.0`：https://busybox.net/downloads/

**报告目录**
[toc]

# 1. 编译 ARM 架构 Linux 内核

下载内核

```shell
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.0.12.tar.xz
```
![](https://oss.kimidayo.cn/img/2022-12-08%2021.12.28.webp)

解压并进入内核目录

```shell
tar -xvf linux-6.0.12.tar.xz

cd linux-6.0.12
```
![](https://oss.kimidayo.cn/img/2022-12-08%2021.14.16.webp)

配置
```
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- menuconfig -j4 O=./object
```
编译
```
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- -j4 O=./object
```

得到编译文件： `zImage` 和 `vexpress-v2p-cap.dtb`
![](https://oss.kimidayo.cn/img/2022-12-08%2015.41.48.webp)

`qemu` 启动 `arm linux` 命令
```
qemu-system-arm \
        -M vexpress-a9  \
        -m 512M \
        -kernel ~/mylinux/zImage   \
        -dtb ~/mylinux/vexpress-v2p-ca9.dtb    \
        -nographic  \
        -append "root=/dev/mmcblk0 rw console=ttyAMA0"    \
```
参数含义解释

| 参数             | 含义                             |
| -                | -                                |
| -M vexpress-a9   | 指定要仿真的开发板：vexpress-a9  |
| -m 512M          | 指定DRAM内存大小为512MB          |
| -kernel ./zImage | -kernel ./zImage                 |
| -nographic       | 非图形化启动，使用串口作为控制台 |
| -append cmdline  | 设置Linux内核命令行、启动参数    |
| -sd rootfs.ext3  | 使用rootfs.ext3作为SD卡镜像文件  |

![](https://oss.kimidayo.cn/img/2022-12-08%2017.13.03.webp)

最后一行报错 `Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)` 是因为没有挂在根文件系统，`Ctrl + A` + `X` 退出 `QEMU` 下一步开始制作根文件系统

注：如果感觉命令太长不方便查看和编辑，可以用 `\` 转义空格来分行，为了避免每次输入该长命令可以将其保存在 `.sh` 文件中

```shell
cat boot.sh
! /bin/sh
qemu-system-arm \
        -M vexpress-a9  \
        -m 512M \
        -kernel your_output_dir/arch/arm/boot/zImage   \
        -dtb your_output_dir/arch/arm/boot/dts/vexpress-v2p-ca9.dtb    \
        -nographic  \
        -append "console=ttyAMA0"
```



# 2. 制作根文件系统

## 2.1 形成根目录结构

1. 下载、配置、编译、安装 busybox

下载 busybox

```
wget https://busybox.net/downloads/busybox-1.35.0.tar.bz2
tar -xvf busybox-1.35.0.tar.bz2
cd busybox-1.35.0
```

配置 busybox

```
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- menuconfig
```
可以在 `menuconfig` 菜单中指定安装目录，默认会在 `busybox-1.35.0` 目录下的 `_install`
```
Settings  --->
Installation Options ("make install" behavior)
(./_install) Destination path for 'make install'

```

编译 busybox

```
make defconfig
make CROSS_COMPILE=arm-linux-gnueabi-
make install
```
安装 busybox

```
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- install
```

<!-- ![](https://oss.kimidayo.cn/img/%E6%88%AA%E5%B1%8F2022-12-07%2022.32.46.webp) -->

![](https://oss.kimidayo.cn/img/%E6%88%AA%E5%B1%8F2022-12-07%2022.33.41.webp)

安装完成后，默认会在busybox目录下生成 `_install` ，该目录下的程序就是单板运行所需要的命令。


![](https://oss.kimidayo.cn/img/%E6%88%AA%E5%B1%8F2022-12-07%2022.36.18.webp)

注意到上面的 `make` 命令都带 `ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-` 参数，一只输入比较麻烦，可以在一开始修改 `Makefile` 文件
```
vim Makefile
ARCH = arm
CROSS_COMPILE = arm-linux-gnueabi-
```

配置后编译安装命令变简单

```
make
make install
```

先在Ubuntu主机环境下，形成目录结构，里面存放的文件和目录与单板上运行所需要的目录结构完全一样，然后再打包成镜像（在开发板看来就是SD卡），这个临时的目录结构称为根目录

2. 创建 `rootfs` 目录（根目录），根文件系统内的文件全部放到这里

```
mkdir rootfs
```

3. 拷贝 `busybox` 命令到根目录下

```
cp busybox-1.35.0/_install/* rootfs/ -rfd
```

4. 从工具链中拷贝运行库到 `lib` 目录下, 在根文件系统中添加加载器和动态库

```
mkdir rootfs/lib
cp /usr/arm-linux-gnueabi/lib/* rootfs/lib/ -rfp
```

5. 静态创建设备文件，4个tty终端设备

```
mkdir rootfs/dev
cd rootfs/dev
sudo mknod rootfs/dev/tty1 c 4 1
sudo mknod rootfs/dev/tty2 c 4 2
sudo mknod rootfs/dev/tty3 c 4 3
sudo mknod rootfs/dev/tty4 c 4 4
```

![](https://oss.kimidayo.cn/img/%E6%88%AA%E5%B1%8F2022-12-07%2022.41.24.webp)

至此简易根文件目录构建完成，该根文件系统只有最基本的功能。

## 2.2 制作根文件系统镜像

1. 生成一个32M大小的空SD卡镜像
```
$ dd if=/dev/zero of=a9rootfs.ext3 bs=1M count=32
```

2. 将SD卡格式化成ext3文件系统
```
$ mkfs.ext3 a9rootfs.ext3
```

3. 将 `rootfs` 文件夹拷贝到SD镜像中
```
mkdir tmpfs

sudo mount -t ext3 rootfs.ext3 tmpfs/ -o loop

sudo cp -rf rootfs/* tmpfs/ 

sudo umount tmpfs
```

![](https://oss.kimidayo.cn/img/2022-12-08%2018.40.56.webp)

# 3. 使用 qemu 启动系统

将 `zImage` `vexpress-v2p-c9a.dtb` `rootfs.ext3` 三个文件置于 `~/mylinux` 目录下，然后用下面的命令启动，也可不移动文件，自行更改文件路径参数 `-kernel` `dtb` `-sd`

 `qemu` 启动命令
```shell
qemu-system-arm \
        -M vexpress-a9  \
        -m 512M \
        -kernel ~/mylinux/zImage   \
        -dtb ~/mylinux/vexpress-v2p-ca9.dtb    \
        -nographic  \
        -append "root=/dev/mmcblk0 rw console=ttyAMA0"    \
        -sd rootfs.ext3
```

![](https://oss.kimidayo.cn/img/2022-12-08%2017.57.30.webp)

# 4. 成功启动 linux 画面
最终成功启动 linux 画面

![](https://oss.kimidayo.cn/img/2022-12-08%2019.42.43.webp)

---
最初启动画面
这里看到 `can't run '/etc/init.d/rcS': No such file or directorya` ，添加 `/etc/init.d/rcS` 文件即可。

`rcS` 是一个 shell 脚本， 在Linux内核启动以后，需要启动一些服务， 而 `rcS` 就是规定启动哪些文件的脚本文件。文件内容可以是启动后的提示语句。


由于要对根文件目录中的文件进行修改，首先先将根文件目录加载到 `tmpfs` 文件下（当然也可以是其他文件夹，自己选择一个位置就可以）

![](https://oss.kimidayo.cn/img/2022-12-08%2019.07.37.webp)

```
sudo mount -t ext3 rootfs.ext3 tmpfs -o loop
sudo cat etc/init.d/rcS
```

使用自己习惯的编辑其编辑 `rcS` 文件
```
sudo vim etc/init.d/rcS
sudo umount /mnt
```
![](https://oss.kimidayo.cn/img/2022-12-08%2019.41.20.webp)

需要给文件设置可执行权限, 否则会出现如下 `Permission denied` 错误

![](https://oss.kimidayo.cn/img/2022-12-08%2019.28.46.webp)

重新给 `rcS` 设置执行权限

```shell
sudo mount -t ext3 rootfs.ext3 tmpfs -o loop
sudo chmod +x etc/init.d/rcS
sudo umount tmpfs
```
![](https://oss.kimidayo.cn/img/2022-12-08%2019.32.39.webp)
再次使用 `qemu` 启动 linux， 看到打印了欢迎信息

![](https://oss.kimidayo.cn/img/2022-12-08%2019.42.43.webp)

`Ctrl + A` `X` 看到 `QEMU: Terminated` ， 退出了 qemu。 实验结束


# 5. 实验过程中的困难和错误

## 5.1 linux kernel 版本

内核版本较低，和gcc版本不匹配，最开始使用了 `3.16.1` 内核版本，`arm-linux-gnueabi-gcc` 为 `11.3.0` 版本。
在编译的过程中出现了 `fatal error: Linux/compiler-gccl11.h: No such file or directory`，尝试切换 `gcc` 无果后，决定使用最新 kernel 版本 `6.0.11` 编译成功

**当时第一次的具体操作流程如下**:

编译 ARM 架构 Linux 内核

下载内核

```shell
wget https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.16.1.tar.gz
```

![](https://oss.kimidayo.cn/img/%E6%88%AA%E5%B1%8F2022-12-07%2019.44.25.webp)

解压并进入内核目录

```shell
tar zxvf linux-3.16.1.tar.gz

cd linux-3.16.1
```
![](https://oss.kimidayo.cn/img/%E6%88%AA%E5%B1%8F2022-12-07%2019.46.53.webp)

生成 `vexpress-a9` 的 `config` 文件(先执行 `make clean && make mrproper`)

```shell
make CROSS_COMPILE=arm-linux-gnueabi- ARCH=arm O=./out_vexpress_3_16 vexpress_defconfig
```
![](https://oss.kimidayo.cn/img/%E6%88%AA%E5%B1%8F2022-12-07%2019.49.39.webp)

执行内核配置

```
make CROSS_COMPILE=arm-linux-gnueabi- ARCH=arm O=./out_vexpress_3_16 menuconfig
```

![](https://oss.kimidayo.cn/img/%E6%88%AA%E5%B1%8F2022-12-07%2020.00.38.webp)

要注意此时窗口不能太小，否则无法显示 `Menu`，这里把 Terminal 窗口拉大后正常运行

![](https://oss.kimidayo.cn/img/%E6%88%AA%E5%B1%8F2022-12-07%2019.52.04.webp)

在弹出的窗口中 首先选中 `System Type` ，按 `Enter` 进入子菜单后向下找到 `[*] Enable the L2x0 outer cache controller` 选项，按空格键取消该选项，其他选项保持默认，如下图所示。保存后退出

![](https://oss.kimidayo.cn/img/%E6%88%AA%E5%B1%8F2022-12-07%2019.52.59.webp)

编译 linux 内核

```
make CROSS_COMPILE=arm-linux-gnueabi- ARCH=arm O=./out_vexpress_3_16 zImage -j2
```

![](https://oss.kimidayo.cn/img/%E6%88%AA%E5%B1%8F2022-12-07%2019.58.42.webp)

遇到报错没有找到 `compiler-gcc11.h`，去排查错误，进入 `../linux-3.16.1/inlcude/linux`目录，查找 `.h` 文件

```
cd inlcude/linux
ls | grep gcc
```
![](https://oss.kimidayo.cn/img/%E6%88%AA%E5%B1%8F2022-12-07%2020.15.45.webp)

可以看到内核版本较低，和gcc版本不匹配。有多种解决方案

- 重装低版本 gcc
- 在新内核源码中拷贝 compiler-gcc11.h 到 `linux-3.16.1/inlcude/linux` 目录下
- 将其中 一个 `compiler-gcc4.h` 文件 重命名为 `compiler-gcc11.h`

这里首先尝试使用kernel中的 `gcc?.h` 文件。采用改名的方式 `mv compiler-gcc4.h compiler-gcc11.h`, 让其直接使用现有的 `.h` 文件

![](https://oss.kimidayo.cn/img/%E6%88%AA%E5%B1%8F2022-12-07%2020.24.04.webp)

尝试后并果然没有解决问题

![](https://oss.kimidayo.cn/img/%E6%88%AA%E5%B1%8F2022-12-07%2020.25.20.webp)

决定下载最新系统镜像，重复上述步骤进行配置

![](https://oss.kimidayo.cn/img/%E6%88%AA%E5%B1%8F2022-12-07%2020.42.54.webp)

看到编译成功，生成的内核镜像位于 `/.out_express_3_16/arch/arm/boot/zImage`。问题解决

## 5.2 qemu 不能正常启动系统镜像

排查后发现 qemu 启动命令设置不正确，需要指定驱动板 `-dtb`，在指定 `-dtb ~/mylinux/vexpress-v2p-ca9.dtb` 后 `qemu` 成功启动内核镜像

当时出错的具体操作流程如下:

测试 qemu 能够正常启动

```shell
qemu-system-arm \
        -M vexpress-a9  \
        -m 512M \
        -kernel /your_output_dir/arch/arm/boot/zImage   \   
        -nographic  \
        -append "console=ttyAMA0"
```
测试 qemu 能够正常启动镜像，进程长时间卡在这里，不继续运行

![](https://oss.kimidayo.cn/img/%E6%88%AA%E5%B1%8F2022-12-07%2021.13.15.webp)

使用该命令正确启动

```shell
qemu-system-arm \
        -M vexpress-a9  \
        -m 512M \
        -kernel ~/mylinux/zImage   \
        -dtb ~/mylinux/vexpress-v2p-ca9.dtb    \
        -nographic  \
        -append "root=/dev/mmcblk0 rw console=ttyAMA0"    \
```
## 5.3 设置虚拟机终端代理

最初直连下载时速度不理想，设置终端代理。

首先获取主机 ip
查看 mac 本机 ip

```
ipconfig
```

将终端代理命令中的 `127.0.0.1` 本地环回地址改为本机 `ip` 地址
```
export https_proxy=http://127.0.0.1:7890 \
       http_proxy=http://127.0.0.1:7890 \
       all_proxy=socks5://127.0.0.1:7890
```

为了避免每次设置可以加入 `~/.bashrc` 然后 `source ~/.bashrc`
