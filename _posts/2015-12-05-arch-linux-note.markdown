---
layout: post
title:  "ArchLinux安装与配置小结"
date:   2015-12-05 22:15:52
comments: true
categories: Tech
---

最近无意间发现一个基于ArchLinux的发行版——[BlackArch][BlackArch]，主题十分炫酷（中二）。当然渗透类的Linux
发行版已经有BackTrack和Kali了，不过都是源于Debian的，使用者众多，随波逐流无法突显我们的逼格，
要论小众，ArchLinux算是个中翘楚。呵呵开个玩笑，其实ArchLinux的亮点在于“小”，不同于其他发行版的“最小化安装”，
ArchLinux的安装过程就与众不同，手动分区，手动配置bootloader，然后从网络源下载必要的包安装到指定的分区中。
安装完必备的软件（如gcc）后其余一概自己去添加，包括图形界面。对于喜欢定制而又不怕折腾的朋友来说，Arch系列确实
是一个不错的选择。正好我还没有用过Arch系的发行版，因此就在虚拟机装一个体验一回，顺便把其中遇到的坑记录一下。

# 安装ArchLinux

关于ArchLinux的安装，已经有无数文章介绍过了，不过质量良莠不齐，每个人遇到的问题也不尽相同，因此强烈建议先看一遍
官网的[Beginner-guide][Beginner-guide]。总的来说，操作系统的安装过程不算复杂，问题主要出现在分区和bootloader的配置上，
能解决就好办多了。

## 准备工作

在官网的Download界面里就有最新的iso镜像源可以下载，包括磁力链接和bt种子，下载好后烧录到光盘或者u盘，就可以作为一个操作
系统直接启动了。和一般的系统安装不同，启动后以root用户进入一个传统的linux命令行，内含/root /boot /home /etc /mnt等常见
目录，这时可以看到有很多软件能用，比如ifconfig，因为此时软件都是在镜像上面的。我们所需要做的，就是通过这个命令行来把我们
的操作系统安装到预留的磁盘空间内。

## 磁盘分区

磁盘分区是安装系统的第一步，也是最重要的一步，分区的好坏决定了以后使用磁盘的效率，甚至会影响系统的正常启动。关于磁盘分区可以
先阅读一下[Arch Wiki][partition]的相关部分内容。首先，我们可以用`lsblk`命令查看分区和挂载点，在没分区时应该只能看到空闲的*/dev/sdx*。
分区信息主要存储在分区表中，目前有两种主流的模式，即MBR（Master Boot Record）和GPT（GUID Partition Table），鸟哥的私房菜里讲的就是
传统的MBR分区，但其限制较多，比如只支持4个分区（后来加入拓展分区和逻辑分区），而GPT是较新的一种分区模式，解决了前者许多限制，因此这里
就采用GPT来进行分区。分区的工具有很多，比如`parted`、`fdisk`、`gdisk`、`cfdisk`等，这里以gdisk（GPT版的fdisk）为例，假设空闲磁盘为sda：

    gdisk /dev/sda

进入交互的命令行模式，显示效果如下：

    Command (? for help):
    
此时按‘n’回车可以添加分区，依次交互选择分区号，起始扇区，终止扇区和“hex code”。如第一个分区（boot）选择分区号为默认（1），起始扇区选默认，
终止扇区设置为“+250M”（不要引号）。hex code表示文件系统类型，默认为8300（Linux File System），一般选默认就可以了。不过要注意的是如果有交换分区，
要把hex code设置为8200。都完成后可以在交互模式输入p回车查看分区详细信息，如果没问题，就输入w回车写入分区。退出后可以通过命令查看详细分区：

    fdisk -l

值得一提的是boot分区格式要设置为EFI System，可以在hex code里指定也可以在parted交互命令行中设置：

    parted /dev/sda
    (parted) set 1 boot on

## 格式化并挂载磁盘

假设我们已经将/dev/sda分区为四部分sda1～sda4，分别对应boot，swap，/目录，home，首先格式化一般的存储目录：

    mkfs -t ext4 /dev/sda1
    mkfs -t ext4 /dev/sda3
    mkfs -t ext4 /dev/sda4

对于交换分区我们要用mkswap命令设置格式：

    mkswap /dev/sda2

设置好文件格式后需要把我们的分区(临时)挂载到文件系统上：

    mount /dev/sda3 /mnt
    mount /dev/sda1 /mnt/boot
    mount /dev/sda4 /mnt/home
    swapon /dev/sda2

## 在挂载点安装Arch Linux

这里用到Arch系列的常用命令pacstrap从网络安装基础包和基础开发包：

    pacstrap /mnt base base-devel

经过大概半小时就可以安装好所有基础的文件，此时Arch Linux已经算是安装好了，但是如果直接重启是进不了系统的，因为硬件启动时候还没有挂载分区。
因此我们先生成操作系统当前的分区信息文件：

    genfstab -p /mnt >> /mnt/etc/fstab

这个文件的作用是在以后启动时自动挂载你的磁盘分区，同时也会检测前面设置的交换分区（如果有的话）。

## 安装bootloader

在电脑刚刚启动的时候，并不知道该如何进入操作系统，因此需要一个引导，bootloader起的就是引导作用。最常见的bootloader就是grub和[syslinux][Syslinux]，
如果使用GRUB legacy作为bootloader，必须使用MBR，因此我们选择syslinux。

首先我们现在分区里安装syslinux：

    pacstrap /mnt syslinux

关于syslinux的配置过程可以参考官网的wiki[Syslinux]，安装可分为自动安装和手动安装，推荐自动安装：

    syslinux-install_update -i -a -c /mnt

安装完成后用arch-chroot命令进入我们的新系统设置语言，时区等其他配置：

    arch-chroot /mnt

修改`/etc/locale.gen`文件选择语言，建议不要选中文不然会出现命令行乱码。

    locale-gen
    echo LANG="en_US.UTF-8" > /etc/locale.conf
    ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

修改syslinux的配置信息，在文件`/boot/syslinux/syslinux.cfg`中可以进行自定义的配置。这里注意的是在Comboot modules
中可以看到有启动时需要的模块XXX.c32，我们需要把对应的`/usr/lib/syslinux/bios/XXX.c32`复制到`/boot/syslinux/`目录下。
然后运行：
    
    extlinux --install /boot/syslinux

此时bootloader已经安装好了，对于PC来说，还需要一个启动协议，即启动系统的指令，对GPT而言就需要适合GPT分区的启动协议，这里是gptmbr.bin:

    dd conv=notrunc bs=440 count=1 if=/usr/lib/syslinux/bios/gptmbr.bin of=/dev/sda

最后初始化磁盘环境， “Make an initial ramdisk environment (mkinitcpio) using presets (-p) suitable for Linux” ：

    mkinitcpio -p linux

然后退出chroot，取消挂载再重启就可以了，重启之后记得拔出安装光盘（u盘）：

    umount -R /mnt
    swapoff /dev/sda2

更详细的安装图文过程可以参考ArchWiki或者国外一些博客如[Braun's guide][Braun's guide]。

# 使用ArchLinux

这里先简单介绍下ArchLinux的包管理工具pacman，类似与debian系列的apt和redhat系列的rpm，
方便我们搜索，安装，升级和卸载软件。源的配置文件在`/etc/pacman.conf`，镜像列表在`/etc/pacman.d/mirrorlist`，可以根据速度选择一个合适的源。

在安装软件前我们需要先和软件源进行一次同步，相当于apt-get update:

    pacman -Syy

同步之后可以选择批量更新，类似与apt-get upgrade:

    pacman -Syu

在源里搜索并安装单个软件包，会自动安装其依赖：

    pacman -S package

安装从其他地方下载到本地的软件包：

    pacman -U /path/to/package/package_name.pkg.tgz

安装某个软件前可以先在源里搜索一有没有，类似apt-cache search xxx:

    pacman -Ss package

类似的在本地搜索软件包：

    pacman -Qs package

删除某个软件包，以及只有其用到的依赖包，如果要保留依赖可以只指定-R：

    pacman -Rs package

更详细的命令可以查看pacman的manpage。

## 安装驱动

**显卡驱动**

Xorg默认安装的时候已经安装了部分开源的图形驱动，一般来说已经够用，我们也可以安装自己显卡对应的闭源驱动，比如可以去其
显卡官网或者笔记本电脑官网下载Linux版本的最新驱动，或者从镜像源下载不那么新的稳定版：

    pacman -S mesa xf86-video-intel #A卡
    pacman -S nvidia  nvidia-utils  #N卡

**声卡驱动**

声卡驱动有oss和alsa，一般选择后者，装好后可以用`alsamixer`命令调节音量:

    pacman -S alsa-utils

**其他**

除了声卡和显卡的驱动，我们一般还需要无线网卡的驱动，可以在笔记本官网下载。值得一提的是，如果笔记本没有
有线网络的话，这个最好在安装之前就先准备好无线网卡的驱动，以方便随后用pacman安装其他软件。

## 配置网络

上述步骤完成后我们就能重启进入ArchLinux的命令行了，默认是tty1，没有桌面环境，而此时也好不能上网，需要手动配置。
首先在`/etc/rc.conf`文件的NETWORKING段配置ip（你的IP）/netmask（子网掩码）/broadcast（广播地址）/ROUTES（路由），
如果为动态，将/etc/rc.conf文件的eth0改为dhcp形式。

> - **注意 :** 对于新版本的ArchLinux，启动程序的管理已经从initscript
    变成systemd，统一用systemctl命令管理而不用/etc/rc.conf管理。

如果只有无线网络，我们可以用iwconfig命令连接上路由器：

    iwconfig wlan0 essid AP_NAME key KEY_NAME

其中essid后面是Wifi的名字，key后面是Wifi密码。此时应该就可以访问互联网了。值得一提的是如果为虚拟机，默认网络就
已经连接好了，我们只需要通过`sudo dhcpcd`命令打开即可。

## 图形界面

利用pacman命令我们就可以安装需要的软件了，对一般PC而言先装一个图形界面是挺有必要的，首先安装Xorg。

    pacman -S xorg-server xorg-xinit xorg-utils xorg-server-utils

Xorg是X11的其中一个实现，而X11(X Window System)是一个C/S结构的程序，Xorg只是提供了一个X Server，
负责底层的操作，当你运行一个程序的时候，这个程序会连接到X server上，由X server接收键盘鼠标输入和负责屏幕输出窗口的移动，
窗口标题的样式等等，都是由一种叫做窗口管理器的程序来完成的，安装好后startx，如下图所示，其中包涵了Xorg，4个窗口及对应的程序，
和Xorg自带的窗口管理器twm:

![](http://images2015.cnblogs.com/blog/676200/201512/676200-20151205170047111-2131363673.png)

可以看到这些窗口已经有了图形界面的雏形，如果为了更好的显示效果可以自己去安装其他窗口管理器如Gnome，KDE，xfce等。
下面三个命令分别安装xfce窗口管理器，字体以及输入法：

    sudo pacman -S xfce4
    sudo pacman -S ttf-arphic-uming ttf-arphic-ukai ttf-bitstream-vera 
    sudo pacman -S ibus ibus-pinyin

安装好后reboot，然后进入tty，用`startxfce4`就可以进入窗口管理器了：

![](http://images2015.cnblogs.com/blog/676200/201512/676200-20151205175931877-746864784.png)

可以看到画面十分简洁清秀～嘿嘿。

## 后记

至此ArchLinux的安装和配置就完成了，虽然步骤比较多，但由于其Wiki非常详细且全面，因此90%的问题都能找到解决方法。
前期安装bootloader的时候也有好几次因为配置不全/不正确导致开机黑屏无法进入引导几近抓狂，但俗话说得好，失败是免费的，
从失败中学到正确的方法，对自己也没什坏处。另外通过玩一遍ArchLinux，又对系统的分区和启动过程了解不少，真算得上是"获益匪浅"了。



[BlackArch]:http://blackarch.org/downloads.html
[partition]:https://wiki.archlinux.org/index.php/Partitioning_(简体中文)
[Syslinux]: https://wiki.archlinux.org/index.php/Syslinux
[Beginner-guide]: https://wiki.archlinux.org/index.php/Beginners'_guide
[Braun's guide]:http://wideaperture.net/blog/?p=3851wpals
