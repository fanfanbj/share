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
# 第三部分 几种存储方式详细介绍
