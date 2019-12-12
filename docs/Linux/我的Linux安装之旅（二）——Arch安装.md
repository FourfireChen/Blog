# 我的Linux安装之旅（二）——Arch安装

这里还是要特别感谢[wd老哥的Arch安装教程](https://www.viseator.com/2017/05/17/arch_install/)。

And，本文我只按照自己的安装路径写，具体其他问题，可以参考上面的安装教程

### 启动盘制作

在[ArchLinux官网](https://www.archlinux.org/download/)上下载`archlinux-**-x86_64.iso`这个文件，然后写入事先准备好的U盘。Linux下可以用WoeUSB，Windows下可以用Rufus。

### 进入U盘下的Linux

首先进入BIOS下设置好启动顺序，把USB调到第一位。后进入U盘中写入的系统，等待加载完毕。

### 联网

* 
有线网：如果路由器支持DHCP的话，可以插上网线后执行`dhcpcd`命令

* 
  无线网：最坑的地方来了，但这里还问题不大。执行：`wifi-menu`

看看能不能连上，如果可以，完美。如果不可以，若是联想笔记本，可以借鉴我。执行：

```shell
rfkill unblock all
```

如果问题解决，ok。若还是不行，继续执行：

```shell
rfkill list
```

看看是否有模块``XX``(如联想有可能是`ideapad_laptop`)排在`Wireless LAN`前面，若是有，执行：

```shell
modprobe -r XX //按实际情况替换
```

如果没有问题，重新尝试连接。

**连接后可以执行命令测试**：

```shell
ping www.baidu.com
```

### 更新时间

```shell
timedatectl set-ntp true
```

### 分区工作

首先执行命令查看分区情况：

```
fdisk -l
```

我们的目的是准备好一个容量大的挂载根目录`/`，准备一个500M到1G的挂载`boot`分区，也就是EFI分区。

Linux中对硬盘的表示是`/dev/sd?`或者`/dev/nvmen?`，而这个名字之后，再带p？的，就是这个磁盘第？分区。

#### 创建引导分区

**引导分区不用和windows一样，windows10默认下的EFI引导分区才100M，我们自己建一个新分区。**

执行：

```shell
fdisk /dev/nvmen0
```

* 
输入m可以查看命令帮助。
* 
如果是一个全新的硬盘，注意按`g`创建分区表。

* 
  `n`创建分区，`t`更改分区类型，`l`可以查看支持的类型，根据类型输入，更改该分区类型为`EFI`

* 
  `w`别忘了，生效。
* 格式化`mkfs.fat -F32 /dev/nvmen?p?`，`nvmen?p?`是要作为引导分区的分区。

#### 创建根目录

与创建引导分区大致一样，不一样的是不用更改类型，并且格式化为：

```shell
mkfs.ext4 /dev/nvmen?p?
```

#### 挂载

这个每次从U盘引导进来的，都要做一次：
```
mount /dev/nvmen?p? /mnt
mkdir /mnt/boot
mount /dev/nvmen?p? /mnt/boot
```

### 换镜像源

编辑`/etc/pacman.d/mirrorlist`：

```shell
vim /etc/pacman.d/mirrorlist
```

更换一个速度快的国内源到第一行，比如

```
Server=http://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
```

### 安装基本包

```
pacstrap /mnt base base-devel
```

等待安装完成

### 配置自动挂载

这一步很重要，利用genfstab帮助每次启动时进行自动挂载：

```shell
genfstab -L /mnt >> /mnt/etc/fstab
```

执行完检查一下：

```shell
cat /mnt/etc/fstab
```

结果看看是不是达到目的。

同时有个坑，就是如果改动了分区挂载点，比如改动了`boot`分区，就要重新回来这一步，把原来的结果删除掉，重新执行生成自动挂载配置。一般如果没有执行的话，进入硬盘引导出来会报错`emergency mode`相关。

### 操纵权改变

```
arch-chroot /mnt
```

### 设置时区

```shell
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localltime
hwclock --systohc
```

### 安装软件

```shell
pacman -S vim dialog wps_supplicant ntfs-3g networkmanager
```

### 设置语言

```shell
vim /etc/locale.gen
```

把`zh_CN.UTF-8 UTF-8` `zh_HK.UTF-8 UTF-8` `zh_TW.UTF-8 UTF-8` `en_US.UTF-8 UTF-8`四行取消注释。然后执行：

```
locale-gen
```

编辑`/etc/locale.conf`：

```shell
vim /etc/locale.conf
```

输入：

```shell
LANG=en_US.UTF-8
```

### 设置主机名

编辑`/etc/hostname`：

```shell
vim /etc/hostname
```

在文件第一行输入你的主机名，以后这个就是你在Linux下工作时，本计算机的名字。如`mycomputer`

编辑`/etc/hosts`：

```shell
vim /etc/hosts
```

输入：

```
127.0.0.1 localhost
::1 localhost
127.0.1.1	mycomputer.localdomain	mycomputer
```

### 设置root密码

```shell
passwd
```

### IntelCPU安装Intel-ucode

```shell
pacman -S intel-ucode
```

### 安装Bootloader

最流行的是`Grub2`

#### 安装os-prober

它是用来帮助`grub`检测硬盘中存在的其他系统，如双系统时的windows。

```shell
pacman -S os-prober
```

#### 安装配置grub引导

```
pacman -S grub efibootmgr
```

部署`grub`：

```shell
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
```

生成配置：

```shell
grub-mkconfig -o /boot/grub/grub.cfg
```

这一步大概率出错。

* 
如果是`lvmetad/lvm`相关错误，或者是`WARNING: Device /dev//*xxx/* not initialized in udev database even after waiting 10000000 microseconds`等等，应该

```shell
exit //退出arch-chroot
mkdir /mnt/hostlvm
mount --bind /run/lvm /mnt/hostlvm
arch-chroot /mnt
ln -s /hostlvm /run/lvm
```

再执行生成配置的命令，一般可以成功。

**需要注意的是，这一段如果是重启，重新进入，很可能要再执行一次**
* 如果没有扫到Linux的镜像，要先执行

  ```shell
  pacman -S linux
  ```

* 
  如果没有扫到windows，可以忽略，等到不用U盘引导而是硬盘引导进入系统时，再执行一次生成配置。

#### 安装后检查

```shell
vim /boot/grub/grub.cfg
```

查查在文件中`menuentry`是不是有对应的系统引导信息。

### 重启进入硬盘引导的系统

```shell
exit
reboot
```

如果grub部署问题不大，可以直接进入grub引导下的arch系统。

重启后记得，如果在U盘下没有部署其他系统的引导，重新执行一次grub生成配置的命令。

### 创建交换文件

交换文件是用于内存不足时缓存部分旧内容。

分配空间：

```shell
fallocate -l 8G /swapfile
```

大小看内存，足够大的情况下，可以4-8G。

更改权限：

```shell
chmod 600 /swapfile
```

设置交换文件：

```shell
mkswap /swapfile
```

启用交换：

```shell
swapon /swapfile
```

设置交换文件分区的自动挂载，编辑：

```shell
vim /etc/fstab
```

输入：

```shell
/swapfile none swap defaults 0 0
```

### 新建用户

在此之前都是`root`用户下，从此要创建新用户：

```shell
useradd -m -G wheel username
```

`username`自行更改，就是用户名，`wheel`是组名，一般这么写就好了，不用改，后面会用到。

设置用户密码：

```shell
passwd username
```

### 解决网络问题

执行：

```
wifi-menu
```

如果无法连接，和上面相似，需要进行`rfkill unblcok all`的话，可以把它写成开机自动。如果还没有解决，仍然是`rfkill list`，如果有模块`XX`（这里使用我的情况`ideapad_laptop`），编辑`/etc/modprobe.d/ideapad_laptop.conf`：

在里面添加：

```
blacklist ideapad_laptop
```

然后重启再试试。

如果还有问题，看看自己的无线网卡是不是realtek的，realtek无线网卡一生黑。

### 配置sudo

安装：

```shell
pacman -S sudo
```

配置：

```shell
visudo
```

找到

```shell
# %wheel ALL=(ALL)ALL
```

去掉注释，然后重启系统：

```shell
reboot
```

### 显卡驱动安装

英特尔集显：

```shell
sudo pacman -S xf86-video-intel
```

NVIDIA独显：

```shell
sudo pacman -S nvidia nvidia-utils lib32-nvidia-utils
```

**独显安装完后，在桌面环境安装完之后，还要做一点配置，否则经常登录界面是黑屏**

### 桌面环境Deepin

安装xorg：

```shell
sudo pacman -S xorg
```

安装deepin：

```shell
sudo pacman -S deepin deepin-extra
```

配置Deepin，编辑`/etc/lightdm/lightdm.conf`：

```shell
[Seat:/*]
...
greeter-session=lightdm-deepin-greeter
```

### 独显驱动完善

可以参考[Archwiki对NVIDIA独显的配置文章](https://wiki.archlinux.org/index.php/NVIDIA_Optimus)。

由于Deepin用Lightdm进行引导桌面启动，首先编辑`/etc/lightdm/display_setup.sh`：

```
#!/bin/sh
xrandr --setprovideroutputsource modesetting NVIDIA-0
xrandr --auto
```

给权限：

```shell
chmod +x /etc/lightdm/display_setup.sh
```

编辑`/etc/lightdm/lightdm.conf`

```shell
[Seat:/*]
display-setup-script=/etc/lightdm/display_setup.sh
```

启用lightdm

```shell
sudo systemctl enable lightdm
```

如果用的是`SDDM`或者其他桌面管理，可以参考ArchWiki。

### 网络问题完善

桌面环境下用的是`NetworkManager`这个服务，一开始命令行用的是`netctl`，所以要做禁用和启动。

```shell
sudo systemctl disable netctl
sudo systemctl enable NetworkManager
```

### 安装yay

```shell
cd ~/Downloads
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

### 滚Arch

Arch不滚还有何乐趣？

```shell
yay -Syu
```

### 墙墙

个人觉得图形界面版的shadowsocks好用，不过这个萝卜青菜。

```shell
yay -S shadowsocks-qt5
```

之后打开`shadowsocks`，自己玩玩。

### Chrome代理

先安装Chrome，可以用yay：

```shell
yay -S google-chrome
```

在代理下启动：

```shell
google-chrome-stable --proxy-server="socks5://127.0.0.1:1080"
```

这里的socks5有可能是http，这取决于shadowsocks里的设置。

然后安装`SwitchyOmega`这个插件。

### 命令行代理

安装`proxychains-ng`，然后编辑`/etc/proxychains.conf`：

```
[ProxyList]
socks5 127.0.0.1 1080
```

这里的socks5也有可能是http，取决于shadowsocks里的设置。

### 添加中国源

中国源ArchlinuxCN，很多中文软件如搜狗拼音，都在arch中国源中。编辑`/etc/pacman.conf`，在最后面加入：

```shell
[archlinuxcn]
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```

然后执行

```shell
sudo pacman -Sy
```

### 中文字体

刚刚安装完成的Arch中文是乱码的，可以参考[ArchWiki中文字体](https://wiki.archlinux.org/index.php/Fonts_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)/#%E4%B8%AD%E6%96%87%E5%AD%97)

安装自己喜欢的中文字体，然后更新字体缓存：

```shell
fc-cache -vf
```

这个字体是可以在图形界面里面设置的，也可以手动设置配置文件。

### 输入

安装`fcitx fcitx-im fcitx-configtoolfcitx-gtk2 fcitx-gtk3 fcitx-qt4 fcitx-qt5`，然后安装自己喜欢的输入法，如搜狗拼音`fcitx-sogoupinyin`。

然后启动fcitx-configtools，在里面按+号键，添加自己安装的输入法，就可以使用了。

### 安装QQ、Tim、wechat

先启用multilib库，编辑`/etc/pacman.conf`，对

```
[multilib]
Include = /etc/pacman.d/mirrorlist
```

这两行取消注释。然后执行`yay -Syy`

安装Tim

```shell
yay -S deepin.com.qq.office
```

qq

```shell
yay -S deepin.com.qq.im
```

wechat

```shell
yay -S deepin-wechat
```

### 解决无法关机问题

如果关机特别慢甚至无法关机，可以尝试编辑`/etc/systemd/system.conf`文件，将

```shell
# DefaultTimeoutStopSec=90s
```

改为

```shell
DefaultTimeoutStopSec=10s
```

### 蓝牙安装

安装蓝牙
```shell
yay -S bluez bluez-utils
systemctl start bluetooth
systemctl enable bluetooth
```

安装蓝牙音频

```shell
yay -S pulseaudio-bluetooth
sudo vim /etc/pulse/system.pa
```

配置蓝牙音频

```shell
load-module module-bluetooth-policy
load-module module-bluetooth-discover
```
