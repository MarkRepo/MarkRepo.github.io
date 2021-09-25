1. 准备宿主机环境

   1. 使用ultraISO制作centos u盘启动盘
   2. 安装centos 7双系统（坑：可能u盘的盼复名有问题，在启动时需要修复）
   3. 升级gcc，使用centos-release-scl，yum直接升级
   4. 安装ntfs-3g, 引导win10， grub2-mkconfig -o /boot/grub2/grub.cfg
   5. 设置gcc链接，指向新版本

2. 检查软件版本

   1. 升级了make到4.2版本
   2. 安装texinfo

3. 分区，建立文件系统，设置LFS环境变量，挂载，修改/etc/fstab,开机自动挂载

   1. 只创建了一个分区 /dev/sda7 (考虑是否需要boot, home 分区？)
   2. Ext4 文件系统
   3. export LFS=/mnt/lfs, 需要在用户主目录下的.bash_profile和 .bashrc中都设置下
   4. /etc/fstab: /dev/sda7 /mnt/lfs ext4 defaults 1 1

4. 下载软件包到$LFS/source 目录,  使用lfs的备用网站下载稳定版11.0的所有软件包的tar文件

5. 在#LFS目录下创建目录树

6. 创建lfs用户和组

7. 修改lfs目录下的所有目录的所有者为lfs，切换到lfs用户

8. 创建.bash_profile, .bashrc文件，构建lfs的干净环境

9. 构建交叉编译器和与之相关的库

   1. 理解本地编译器和交叉编译器的概念，理解build（构建程序使用的机器），host（运行构建出来的程序的机器），				target（编译器使用的术语，编译器为这台机器产生代码），如交叉编译器会指定target，为目标机器生成代码。
   2. 构建过程：(后面列出build, host, target)（lfs指chroot环境）
      1. 在pc上使用 cc-pc（本地编译器）构建交叉编译器cc1（pc,pc,lfs）
      2. 在pc上使用cc1构建cc-lfs(本地编译器)(pc,lfs,lfs)
      3. 在lfs上用cc-lfs重新构建，以测试cc-lfs自身（lfs,lfs,lfs）
   3. 构建依赖(值得都是为lfs环境构建库)
      1. 需要使用交叉编译器cc1，为lfs构建glibc库
      2. ccl1依赖libgcc，libgcc必须依赖glibc功能才是完整的； libstdc++ 也依赖glibc
      3. 为了打破上面的循环依赖，先构建一个降级的cc1，他依赖的libgcc缺失线程和异常功能
      4. 用降级的cc1构建glibc（功能完整），再构建libstdc++，libstc++功能会有缺失
      5. 因此进入chroot环境后要使用cc-lfs再次构建libstdc++
   4. 其他细节
      1. 交叉编译器会被安装在tools目录
      2. 先安装binutils
      3. 安装降级cc1
      4. 安装linux api头文件
      5. 安装glibc
      6. 构建libstdc++
      7. 进入chroot，再次安装libstdc++

10. 使用交叉工具链构建一些工具（构建时和宿主分离）

    编译临时工具

    1. m4
    2. ncurses
    3. bash
    4. coreutils
    5. diffutils

11. 进入chroot环境，进一步提高与宿主的隔离度，并构建剩余的、构建最终系统时必须的工具

    1. 修改LFS所有者为root
    2. 准备虚拟内核文件系统
    3. 进入chroot
    4. 按FHS标准创建必要目录
    5. 创建必要的文件和符号链接
    6. 使用cclfs安装libstdc++（第二遍）
    7. 其他工具

12. 构建LFS系统

    1. 安装基本系统软件
    2. 