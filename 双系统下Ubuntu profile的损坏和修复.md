# 双系统Windows11+Ubuntu22.04LTS下profile的损坏和修复

___

#### bug背景

在安装nvm的时候有教程让修改/etc/profile来建立环境变量，最后选了另一个方案。

不过在使用**sudo vi /etc/profile**进行编辑的时候，**方向键**在上面生成了大量的**AB字符**：

```profile
A
A
A
A
A
B
B
B
# 这里是原本的profile
```

:wq退出后，关闭Ubuntu22.04LST后重新打开，输入用户密码后无法进入桌面

报错：**找不到命令 /etc/profile A**

___

#### Debug历程一：修改profile

由于上次退出时已经发现了AB字符的存在，但当时不确定profile是否本身就包含这些AB字符。

确定profile应该是什么样子后，明确修复目标——删除vim产生的AB字符

先后尝试了**Linux Reader**和**ext2explore**两个软件，其中**Linux Reader可以读取并导出profile到Windows11，再由记事本进行修改，但无法写入/etc**，而**ext2explore**无法读取文件

最终找到了**Ext2Fsd**将Linux分区挂载到win上（参考教程：[双系统：windows下编辑linux文件。_windows 下如何编辑 linux 分区内容-CSDN博客](https://blog.csdn.net/hnllc2012/article/details/52847381)），并在win中修改了profile

重启Ubuntu，甚至未能到达用户密码输入界面，直接报错：

```
fsck exited with status code 8
```

然后进入新的界面

```
Free initramfs and switch to another root fs:
chroot to NEW ROOT, delete all in /, move NEW_ROOT tO /,
execute NEW INIT, PID must be 1. NEW_ROOT must be a mountpoint.
	-c DEV Reopen stdio to DEV after switch
	-d CAPS Drop capabililies 
	-n Dry run
No init found, Try passing init= bootarg.
BusyBox v1.30.1 (Ubuntu 1:1.30.1-7ubuntu3) built-in shell (ash)
Enter 'help' for a list of built-in commands.
(initramfs) _
```

___

#### Debug历程二：文件修复

先试了试**help**，给出的命令中**yes**和**reboot**都没有反应

综合看了一些帖子，应该**使用fsck修复磁盘**（参考教程：[initramfs模式介绍及解决方法-CSDN博客](https://blog.csdn.net/qq_44673299/article/details/114295223)）

先查看自己磁盘分区的编号：（大部分教程会让之间修复/dev/sda1，但并不只有这一种编号）

```
(initramfs)blkid
```

获取磁盘信息：

（本机两根固态，

第一根从左到右为**Windows EFI**, **Windows-SSD(C:)**, **Data(D:)**, **Linux主分区**, **恢复分区**

第二根从左到右为**Work(E:)**, **/home**, **/**, **swap**）

```
/dev/nvme1n1p5: UUID="XXXX",TYPE="swap",PARTUUID="XXXX"
/dev/nvme1n1p3: UUID="XXXX",BLOCK_SIZE="XXX",TYPE="ext4",PARTUUID="XXXX"
/dev/nvme0n1p3: LABEL="Windows-SSD",BLOCK_SIZE="XXX",UUID="XXXX",TYPE="ntfs",PART_LABEL="XXXX",PARTUUID="XXXX"
```

查看编号后注意看后续信息，有些编号是windows的磁盘分区名，还有一个是windows保留分区

获取linux的分区号后进行修复（-y可以对后续的修复请求全部同意，不然需要按住y键一个一个同意）：

```
(initramfs)fsck -t ext4 /dev/nvme1n1p3 -y
```

显示进程：

```
fsck from util-linux 2.37.2
e2fsck 1.46.5(30-Dec-2021)
ext2fs_open2: superblock checksum does not match superblock
fsck,ext4: superblock invalid, trying backup blocks...
/dev/nvme1n1p3 was not cleanly unmounted, check forced.
Pass 1: checking inodes, blocks, and sizes
Pass 2: checking directory structure
Pass 3: checking directory connectivity
Pass 4: Checking reference counts
Pass 5: checking group summart information
Free blocks count wrong for group #0(23495,counted=6827).
FIx<y>? yes
Free blocks count wrong for group #1 (31725, counted=25581)
......
```

修复完成后

```
(initramfs)reboot
```

```
(initramfs)exit
```

长按开机键关机后，再按一次重启，正常启动

___

#### 写在最后的碎碎念

由于个人对linux知识的匮乏，所以整个教程看上去很像是一种玄学流程一样（?）

因为debug的时候没有记录磁盘号等信息，准备重开Ubuntu去gparted看一看

但很难相信在写这篇教程的时候Ubuntu又崩了一次

所以我才能再一次记录一些操作细节



然后愉快地又失败了，决定重装系统

Linux，又菜又爱玩

___

#### 重装系统后再次崩掉的发现

重装系统以后还是会出现这样的问题，大概发现了一个现象：

**Windows点击重启电脑，然后进入Ubuntu会报错**

大概是因为Windows重启时有一个**快速启动功能**，导致重启时Windows系统**没有完全关闭**，启动Linux时报错（这也是为什么Linux每次都会**检查磁盘是否clean**）

关机以后再按物理按键开机就没有问题！

