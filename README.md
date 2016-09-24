#Docker存储方式实践分享
# 第一部分 问题诊断事情从一次实施项目说起，我们需要帮助客户将他们的应用容器化并在数人云平台上发布应用。客户应用是传统WAS应用。应用是通过WAS console界面进行手工部署，暂时无法通过Dockerfile进行自动化应用部署，最后的镜像是通过Docker commit完成。镜像启动执行命令是startwas.sh,并通过tail应用日志将输出到标准输出。启动容器，WAS Server启动失败，错误日志如下：
![image](https://github.com/fanfanbj/sharing/blob/master/Picture1.jpg)

WAS Server标准日志文件startServer.log和native_stderr.log没有更加详细的错误信息。最后功夫不负有心人， 在configuration目录下找到可以定位错误的信息 =_=!：

![image](https://github.com/fanfanbj/sharing/blob/master/Picture2.jpg)

文件访问IO异常，查看相应目录文件的属性：

![image](https://github.com/fanfanbj/sharing/blob/master/Picture3.jpg)

到现在为止，可以初步判断是Docker存储方式(storage drive)在镜像容器分层管理上的问题。
当前宿主机是Centos7.2，内核3.10.0。查看当前宿主机是Docker1.12.0，存储方式是Overlay，宿主机的文件系统是xfs:
![image](https://github.com/fanfanbj/sharing/blob/master/Picture4.jpg)

为了验证我们的推断，我们做了如下几方面的尝试：
尝试1: 使用数据卷挂载的方式，挂载整个washome 目录。（数据卷是Docker宿主机的目录或文件，通过mount的方式加载到容器里，不受存储驱动的控制。）重新制作镜像启动容器，WAS Server能正常启动。尝试2: 改变Docker engine的存储方式，改成Devicemapper，重新拉取镜像，并启动容器，WAS Server能正常启动。那么这个问题是普遍问题吗？
尝试3: 在其他的宿主机上，启动原镜像，这个问题是无法复现的。综上所述，我们得到了一个结论，这个问题的根本原因是overlayFS在xfs上出现了兼容性的问题。同时，我们从Docker社区找到相关问题的issue报告：https://github.com/docker/docker/issues/9572这个问题的修改在内核 4.4.6以上。事情的起因到此为止，下面让我们深入的看看Docker的存储方式及其比较，并给出一些技术选型的建议。# 第二部分 镜像容器存储方式概述Docker在启动容器的时候，需要创建文件系统，为rootfs提供挂载点。最底层的引导文件系统bootfs主要包含 bootloader和kernel，bootloader主要是引导加载kernel，当kernel被加载到内存中后 bootfs就被umount了。 rootfs包含的就是典型 Linux 系统中的/dev，/proc，/bin，/etc等标准目录和文件。Docker 模型的核心部分是有效利用分层镜像机制，镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像。Docker1.10引入新的可寻址存储模型，使用安全内容哈希代替随机的UUID管理镜像。同时，Docker提供了迁移工具，将已经存在的镜像迁移到新模型上。不同 Docker 容器就可以共享一些基础的文件系统层，同时再加上自己独有的可读写层，大大提高了存储的效率。其中主要的机制就是分层模型和将不同目录挂载到同一个虚拟文件系统。

![image](https://github.com/fanfanbj/sharing/blob/master/sharing-layers.jpg)

Docker存储方式提供管理分层镜像和Docker容器自己的可读写层的具体实现。最初Docker仅能在支持AUFS文件系统的ubuntu发行版上运行，但是由于AUFS未能加入Linux内核，为了寻求兼容性、扩展性，Docker在内部通过graphdriver机制这种可扩展的方式来实现对不同文件系统的支持。Docker有如下几种不同的drivers：
* AUFS* Device mapper* Btrfs* OverlayFS* ZFS
# 第三部分 存储方式技术方案选择
## AUFS
AUFS（AnotherUnionFS）是一种联合文件系统。所谓 UnionFS 就是把不同物理位置的目录合并 mount 到同一个目录中。UnionFS 的一个最主要的应用是，把一张 CD/DVD 和一个硬盘目录给联合 mount 在一起，然后就可以对这个只读的 CD/DVD 上的文件进行修改（当然，修改的文件存于硬盘上的目录里）。 AUFS 支持为每一个成员目录（类似 Git 的分支）设定只读（readonly）、读写（readwrite）和写出（whiteout-able）权限, 同时 AUFS 里有一个类似分层的概念, 对只读权限的分支可以逻辑上进行增量地修改(不影响只读部分的)。### 基础实验：1.	环境准备ubuntu14.04－3.10.xx内核默认安装了aufs模块(linux主线不支持，如果升级了主线内核会缺乏aufs模块)
		# lsmod | grep aufs
		aufs	202783  2152. `/data/dataman`目录结构
  
        /data/dataman# tree
        .
        ├── aufs
        ├── dir1
        │   └── file1
        └── dir2
            └── file2
- `file1`和`file2`的文件内容

        # cat dir1/file1
        dataman1
        # cat dir2/file2
        dataman2
- 将 `dir1` & `dir2` `mount` 到 `aufs` 目录
    
        # mount -t aufs -o br=/data/dataman/dir1=ro:/data/dataman/dir2=rw none /data/dataman/aufs     
    `mount` 参数说明
        
        -o 指定mount传递给文件系统的参数
        br 指定需要挂载的文件夹，这里包括dir1和dir2
        ro/rw 指定文件的权限只读和可读写
        none 这里没有设备，用none表示
- 再次查看目录结构

        # tree
        .
        ├── aufs
        │   ├── file1
        │   └── file2
        ├── dir1
        │   └── file1
        └── dir2
            └── file2
- 进入 `aufs` 目录测试

        # cd aufs
        # echo "test1" > file1
        -su: file1: Read-only file system
        # echo "test2" > file2
- 查看结果
        
        # cat file1
        dataman1
        # cat file2
        test2
- 如果映射的两个目录文件相同如何处理,在`dir1`创建`file2`，重新`mount`
        
        # umount /data/dataman/aufs
        # cat dir1/file2
        dataman3
        # mount -t aufs -o br=/data/dataman/dir1=ro:/data/dataman/dir2=rw none /data/dataman/aufs
        #echo 
        # cat  aufs/file2
        dataman3
结论若出现有同名文件的情况，则以先挂载的为主，其他的不再挂载。说明Docker镜像为什么采用增量的方式：利用Aufs的特性达到节约空间的目的。 
优点： 
* AUFS存储方式特点是稳定，大量生产部署及丰富的社区支持。AUFS唯一一个 storage driver 可以实现容器间共享可执行及可共享的运行库, 跑成千上百个拥有相同程序代码或者运行库时时候，AUFS是个相当不错的选择。
* 具有快速启动，有效的使用存储和内存等特点。* Ubuntu 10.04，Debian6.0, Gentoo Live CD 默认已经支持* Docker 第一版支持缺点：
* AUFS 到现在还没有加入内核主线( centos 无法直接使用)
* AUFS不支持rename系统调用，将失败当执行“copy”和“unlink”
## Device mapper
 ## Btrfs
 ## OverlayFS

