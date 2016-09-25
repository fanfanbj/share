#Docker存储方式实践分享
# 第一部分 问题诊断事情从一次实施项目说起，我们需要帮助客户将他们的应用容器化并在数人云平台上发布此应用。客户的应用是传统WAS应用。应用是通过WAS console界面进行手工部署，暂时无法通过Dockerfile进行自动化应用部署，最后的镜像是通过Docker commit完成。镜像启动执行命令是startwas.sh,并通过tail将应用日志输出到标准输出。启动容器，WAS Server启动失败，错误日志如下：
![image](https://github.com/fanfanbj/sharing/blob/master/Picture1.jpg)

WAS Server标准日志文件startServer.log和native_stderr.log都没有更加详细的错误信息。最后功夫不负有心人， 在configuration目录下找到可以定位的错误信息 =_=!：

![image](https://github.com/fanfanbj/sharing/blob/master/Picture2.jpg)

文件访问IO异常，查看相应目录文件的属性：

![image](https://github.com/fanfanbj/sharing/blob/master/Picture3.jpg)

到现在为止，可以初步判断是Docker存储方式(storage drive)在镜像容器分层管理上的问题。
当前宿主机是Centos7.2，内核3.10.0。并且查看当前宿主机信息是Docker1.12.0，存储方式是Overlay，宿主机的文件系统是xfs:

![image](https://github.com/fanfanbj/sharing/blob/master/Picture4.jpg)

为了验证我们的推断，我们做了如下几方面的尝试：
尝试1: 使用数据卷挂载的方式，挂载整个washome 目录。（数据卷是Docker宿主机的目录或文件，通过mount的方式加载到容器里，不受存储驱动的控制。）重新制作镜像启动容器，WAS Server能正常启动。尝试2: 改变Docker engine的存储方式，改成Device mapper，重新拉取镜像，并启动容器，WAS Server能正常启动。那么这个问题是普遍问题吗？
尝试3: 在其他的宿主机上，启动原镜像，这个问题是无法复现的。经过多次测试发现在相同内核、系统版本、docker版本有些机器有问题有些机器没有问题，最终发现是 Centos 提供的新文件系统 XFS 和 Overlay 兼容问题导致。同时，我们从Docker社区找到相关问题的issue报告：https://github.com/docker/docker/issues/9572 这个问题的修复在内核 4.4.6以上。综上所述，我们得到了一个结论，这个问题的根本原因是overlayFS在xfs上出现了兼容性的问题。事情的起因到此为止，下面让我们深入的看看Docker的几种存储方式，并给出一些技术选型的建议。# 第二部分 概述Docker在启动容器的时候，需要创建文件系统，为rootfs提供挂载点。最底层的引导文件系统bootfs主要包含 bootloader和kernel，bootloader主要是引导加载kernel，当kernel被加载到内存中后 bootfs就被umount了。 rootfs包含的就是典型 Linux 系统中的/dev，/proc，/bin，/etc等标准目录和文件。Docker 模型的核心部分是有效利用分层镜像机制，镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像。Docker1.10引入新的可寻址存储模型，使用安全内容哈希代替随机的UUID管理镜像。同时，Docker提供了迁移工具，将已经存在的镜像迁移到新模型上。不同 Docker 容器就可以共享一些基础的文件系统层，同时再加上自己独有的可读写层，大大提高了存储的效率。其中主要的机制就是分层模型和将不同目录挂载到同一个虚拟文件系统。

![image](https://github.com/fanfanbj/sharing/blob/master/sharing-layers.jpg)

Docker存储方式提供管理分层镜像和Docker容器自己的可读写层的具体实现。最初Docker仅能在支持AUFS文件系统的ubuntu发行版上运行，但是由于AUFS未能加入Linux内核，为了寻求兼容性、扩展性，Docker在内部通过graphdriver机制这种可扩展的方式来实现对不同文件系统的支持。Docker有如下几种不同的drivers：
* AUFS* Device mapper* Btrfs* OverlayFS* ZFS
# 第三部分 方案分析
## AUFS
AUFS（AnotherUnionFS）是一种Union FS，是文件级的存储驱动。所谓 UnionFS 就是把不同物理位置的目录合并 mount 到同一个目录中。简单来说就是支持将不同目录挂载到同一个虚拟文件系统下的文件系统。这种文件系统可以一层一层地叠加修改文件。无论底下有多少层都是只读的，只有最上层的文件系统是可写的。当需要修改一个文件时，AUFS创建该文件的一个副本，使用CoW将文件从只读层复制到可写层进行修改，结果也保存在可写层。在Docker中，底下的只读层就是image，可写层就是Container。结构如下图所示：

![image](https://github.com/fanfanbj/sharing/blob/master/aufs_layers.jpg)
### 例子
运行一个实例应用是删除一个文件`/etc/shadow`，看aufs的结果

    # docker run centos rm /etc/shadow
    # ls -la /var/lib/docker/aufs/diff/$(docker ps --no-trunc -lq)/etc
    
    total 8
    drwxr-xr-x 2 root root 4096 Sep  2 18:35 .
    drwxr-xr-x 5 root root 4096 Sep  2 18:35 ..
    -r--r--r-- 2 root root    0 Sep  2 18:35 .wh.shadow
### 目录结构
- 容器挂载点(只有容器运行时才被加载)

        /var/lib/docker/aufs/mnt/$CONTAINER_ID/
- 分支(和镜像不同的文件，只读活着读写)

        /var/lib/docker/aufs/diff/$CONTAINER_OR_IMAGE_ID/
- 镜像索引表(每个镜像引用镜像名)

        /var/lib/docker/aufs/layers/
        
### 其他
#### AUFS 文件系统可使用的磁盘空间大小
    # df -h /var/lib/docker/
    
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/vda1        20G  4.0G   15G  22% /
#### 系统挂载方式
启动的 Docker

    docker ps
    
    CONTAINER ID        IMAGE                        COMMAND                CREATED             STATUS              PORTS                      NAMES
    3f2e9de1d9d5        mesos/bamboo:v0.1c           "/usr/bin/bamboo-hap   5 days ago          Up 5 days                                      mesos-20150825-162813-1248613158-5050-1-S0.88c909bc-6301-423a-8283-5456435f12d3
    dc9a7b000300        mesos/nginx:base             "/bin/sh -c nginx"     7 days ago          Up 7 days           0.0.0.0:31967->80/tcp      mesos-20150825-162813-1248613158-5050-1-S0.42667cb2-1134-4b1a-b11d-3c565d4de418
    1b466b5ad049        mesos/marathon:omega.v0.1    "/usr/bin/dataman_ma   7 days ago          Up 16 hours                                    dataman-marathon
    0a01eb99c9e7        mesos/nginx:base             "/bin/sh -c nginx"     7 days ago          Up 7 days           0.0.0.0:31587->80/tcp      mesos-20150825-162813-1248613158-5050-1-S0.4f525828-1217-4b3d-a169-bc0eb901eef1
    c2fb2e8bd482        mesos/dns:v0.1c              "/usr/bin/dataman_me   7 days ago          Up 7 days                                      mesos-20150825-162813-1248613158-5050-1-S0.82d500eb-c3f0-4a00-9f7b-767260d1ee9a
    df102527214d        mesos/zookeeper:omega.v0.1   "/data/run/dataman_z   8 days ago          Up 8 days                                      dataman-zookeeper
    b076a43693c1        mesos/slave:omega.v0.1       "/usr/sbin/mesos-sla   8 days ago          Up 8 days                                      dataman-slave
    e32e9fc9a788        mesos/master:omega.v0.1      "/usr/sbin/mesos-mas   8 days ago          Up 8 days                                      dataman-master
    c8454c90664e        shadowsocks_server           "/usr/local/bin/ssse   9 days ago          Up 9 days           0.0.0.0:57980->57980/tcp   shadowsocks
    6dcd5bd46348        registry:v0.1                "docker-registry"      9 days ago          Up 9 days           0.0.0.0:5000->5000/tcp     dataman-registry
对照系统挂载点

    grep aufs /proc/mounts
    
    /dev/mapper/ubuntu--vg-root /var/lib/docker/aufs ext4 rw,relatime,errors=remount-ro,data=ordered 0 0
    none /var/lib/docker/aufs/mnt/6dcd5bd463482edf33dc1b0324cf2ba4511c038350e745b195065522edbffb48 aufs rw,relatime,si=d9c018051ec07f56,dio,dirperm1 0 0
    none /var/lib/docker/aufs/mnt/c8454c90664e9a2a2abbccbe31a588a1f4a5835b5741a8913df68a9e27783170 aufs rw,relatime,si=d9c018051ba00f56,dio,dirperm1 0 0
    none /var/lib/docker/aufs/mnt/e32e9fc9a788e73fc7efc0111d7e02e538830234377d09b54ffc67363b408fca aufs rw,relatime,si=d9c018051b336f56,dio,dirperm1 0 0
    none /var/lib/docker/aufs/mnt/b076a43693c1d5899cda7ef8244f3d7bc1d102179bc6f5cd295f2d70307e2c24 aufs rw,relatime,si=d9c018051bfecf56,dio,dirperm1 0 0
    none /var/lib/docker/aufs/mnt/df102527214d5886505889b74c07fda5d10b10a4b46c6dab3669dcbf095b4154 aufs rw,relatime,si=d9c01807933e1f56,dio,dirperm1 0 0
    none /var/lib/docker/aufs/mnt/c2fb2e8bd4822234633d6fd813bf9b24f9658d8d97319b1180cb119ca5ba654c aufs rw,relatime,si=d9c01806c735ff56,dio,dirperm1 0 0
    none /var/lib/docker/aufs/mnt/0a01eb99c9e702ebf82f30ad351d5a5a283326388cd41978cab3f5c5b7528d94 aufs rw,relatime,si=d9c018051bfebf56,dio,dirperm1 0 0
    none /var/lib/docker/aufs/mnt/1b466b5ad049d6a1747d837482264e66a87871658c1738dfd8cac80b7ddcf146 aufs rw,relatime,si=d9c018052b2b1f56,dio,dirperm1 0 0
    none /var/lib/docker/aufs/mnt/dc9a7b000300a36c170e4e6ce77b5aac1069b2c38f424142045a5ae418164241 aufs rw,relatime,si=d9c01806d9ddff56,dio,dirperm1 0 0
    none /var/lib/docker/aufs/mnt/3f2e9de1d9d51919e1b6505fd7d3f11452c5f00f17816b61e6f6e97c6648b1ab aufs rw,relatime,si=d9c01806c708ff56,dio,dirperm1 0 0
### 分析
1.	虽然AUFS是Docker 第一版支持的存储方式，但到现在还没有加入内核主线( centos 无法直接使用)2.	从原理分析看，AUFS mount()方法很快，所以创建容器很快；读写访问都具有本机效率；顺序读写和随机读写的性能大于kvm；并且Docker的AUFS可以有效的使用存储和内存 。3.	AUFS性能稳定，并且有大量生产部署及丰富的社区支持4.	不支持rename系统调用，执行“copy”和“unlink”时，会导致失败。5.	当写入大文件的时候(比如日志或者数据库等)动态mount多目录路径的问题,导致branch越多，查找文件的性能也就越慢。(解决办法:重要数据直接使用 -v 参数挂载。)
## Device mapper
Device mapper是Linux内核2.6.9后支持的，提供的一种从逻辑设备到物理设备的映射框架机制，在该机制下，用户可以很方便的根据自己的需要制定实现存储资源的管理策略。Docker的Device mapper利用 Thin provisioning snapshot管理镜像和容器。
###Thin-provisioning SnapshotSnapshot是Lvm提供的一种特性，它可以在不中断服务运行的情况下为the origin（original device）创建一个虚拟快照(Snapshot)。Thin-Provisioning是一项利用虚拟化方法减少物理存储部署的技术。Thin-provisioning Snapshot是结合Thin-Provisioning和Snapshoting两种技术，允许多个虚拟设备同时挂载到一个数据卷以达到数据共享的目的。Thin-Provisioning Snapshot的特点如下：
1.	可以将不同的snaptshot挂载到同一个the origin上，节省了磁盘空间。2.	当多个Snapshot挂载到了同一个the origin上，并在the origin上发生写操作时，将会触发COW操作。这样不会降低效率。3.	Thin-Provisioning Snapshot支持递归操作，即一个Snapshot可以作为另一个Snapshot的the origin，且没有深度限制。
4. 在Snapshot上可以创建一个逻辑卷，这个逻辑卷在实际写操作（COW，Snapshot写操作）发生之前是不占用磁盘空间的。

相比AUFS和OverlayFS是文件级存储，Device mapper是块级存储，所有的操作都是直接对块进行操作，而不是文件。Device mapper驱动会先在块设备上创建一个资源池，然后在资源池上创建一个带有文件系统的基本设备，所有镜像都是这个基本设备的快照，而容器则是镜像的快照。所以在容器里看到文件系统是资源池上基本设备的文件系统的快照，并没有为容器分配空间。当要写入一个新文件时，在容器的镜像内为其分配新的块并写入数据，这个叫用时分配。当要修改已有文件时，再使用CoW为容器快照分配块空间，将要修改的数据复制到在容器快照中新的块里再进行修改。Device mapper 驱动默认会创建一个100G的文件包含镜像和容器。每一个容器被限制在10G大小的卷内，可以自己配置调整。结构如下图所示：
![image](https://github.com/fanfanbj/sharing/blob/master/dm_container.jpg)

可以通过"docker info"或通过dmsetup ls获取想要的更多信息。查看docker的Device mapper的信息：

![image](https://github.com/fanfanbj/sharing/blob/master/dm_docker_info.png)


###分析1.	Device mapper文件系统兼容性比较好，并且存储为一个文件，减少了inode消耗。2.	每次一个容器写数据都是一个新块，块必须从池中分配，真正写的时候是稀松文件,虽然它的利用率很高，但性能不好，因为额外增加了vfs开销。3.	每个容器都有自己的块设备时，它们是真正的磁盘存储，所以当启动N个容器时，它都会从磁盘加载N次到内存中，消耗内存大。 4. Docker的Device mapper默认模式是loop-lvm，性能达不到生产要求。在生产环境推荐direct-lvm模式直接写原块设备，性能好。 ## OverlayFS
Overlay是Linux内核3.18后支持的，也是一种Union FS，和AUFS的多层不同的是Overlay只有两层：一个upper文件系统和一个lower文件系统，分别代表Docker的镜像层和容器层。当需要修改一个文件时，使用CoW将文件从只读的lower复制到可写的upper进行修改，结果也保存在upper层。在Docker中，底下的只读层就是image，可写层就是Container。结构如下图所示：

![image](https://github.com/fanfanbj/sharing/blob/master/overlay_constructs.jpg)

###分析
1.	从kernel3.18进入主流Linux内核。设计简单，速度快，比AUFS和Device mapper速度快。在某些情况下，也比Btrfs速度快。是Docker存储方式选择的未来。因为OverlayFS只有两层，不是多层，所以OverlayFS “copy-up”操作快于AUFS。以此可以减少操作延时。2.	OverlayFS支持页缓存共享，多个容器访问同一个文件能共享一个页缓存，以此提高内存使用率。3.	OverlayFS消耗inode，随着镜像和容器增加，inode会遇到瓶颈。Overlay2能解决这个问题。4.	在Overlay下，为了解决inode问题，可以考虑将/var/lib/docker挂在单独的文件系统上，或者增加系统inode设置。5.	有兼容性问题。open(2)只完成部分POSIX标准，OverlayFS的某些操作不符合POSIX标准。例如： 调用fd1=open("foo", O_RDONLY) ，然后调用fd2=open("foo", O_RDWR) 应用期望fd1 和fd2是同一个文件。然后由于复制操作发生在第一个open(2)操作后，所以认为是两个不同的文件。6.	不支持rename系统调用，执行“copy”和“unlink”时，将导致失败。
## Btrfs
Btrfs被称为下一代写时复制文件系统，并入Linux内核，也是文件级级存储，但可以像Device mapper一直接操作底层设备。Btrfs把文件系统的一部分配置为一个完整的子文件系统，称之为subvolume 。那么采用 subvolume，一个大的文件系统可以被划分为多个子文件系统，这些子文件系统共享底层的设备空间，在需要磁盘空间时便从底层设备中分配，类似应用程序调用 malloc()分配内存一样。为了灵活利用设备空间，Btrfs 将磁盘空间划分为多个chunk 。每个chunk可以使用不同的磁盘空间分配策略。比如某些chunk只存放metadata，某些chunk只存放数据。这种模型有很多优点，比如Btrfs支持动态添加设备。用户在系统中增加新的磁盘之后，可以使用Btrfs的命令将该设备添加到文件系统中。Btrfs把一个大的文件系统当成一个资源池，配置成多个完整的子文件系统，还可以往资源池里加新的子文件系统，而基础镜像则是子文件系统的快照，每个子镜像和容器都有自己的快照，这些快照则都是subvolume的快照。

![image](https://github.com/fanfanbj/sharing/blob/master/btfs_container_layer.jpg)

###分析
1.	Btrfs是替换Device mapper的下一代文件系统， 很多功能还在开发阶段，还没有发布正式版本，相比EXT4或其它更成熟的文件系统，它在技术方面的优势包括丰富的特征，如：支持子卷、快照、文件系统内置压缩和内置RAID支持等。2.	不支持页缓存共享，N个容器访问相同的文件需要缓存N次。不适合高密度容器场景。3.	当前Btrfs版本使用“small writes”,导致性能问题，所以，使用Btrfs要使用Btrfs原生命令btrfs filesys show替代df4.	Btrfs使用“journaling”写数据到磁盘，这将影响顺序写的性能。5.	Btrfs文件系统会有碎片，导致性能问题。当前Btrfs版本，能通过mount时指定autodefrag 检测随机写和碎片整理。## ZFS
ZFS 文件系统是一个革命性的全新的文件系统，它从根本上改变了文件系统的管理方式，ZFS 完全抛弃了“卷管理”，不再创建虚拟的卷，而是把所有设备集中到一个存储池中来进行管理，用“存储池”的概念来管理物理存储空间。过去，文件系统都是构建在物理设备之上的。为了管理这些物理设备，并为数据提供冗余，“卷管理”的概念提供了一个单设备的映像。而ZFS创建在虚拟的，被称为“zpools”的存储池之上。每个存储池由若干虚拟设备（virtual devices，vdevs）组成。这些虚拟设备可以是原始磁盘，也可能是一个RAID1镜像设备，或是非标准RAID等级的多磁盘组。于是zpool上的文件系统可以使用这些虚拟设备的总存储容量。下面看一下在Docker里ZFS的使用。首先从zpool里分配一个ZFS文件系统给镜像的基础层，而其他镜像层则是这个ZFS文件系统快照的克隆，快照是只读的，而克隆是可写的，当容器启动时则在镜像的最顶层生成一个可写层。如下图所示：
![image](https://github.com/fanfanbj/sharing/blob/master/zfs_zpool.jpg)
当要写一个新文件时，使用按需分配，一个新的数据快从zpool里生成，新的数据写入这个块，而这个新空间存于容器（ZFS的克隆）里。

###分析1.	ZFS同 Btrfs类似是下一代文件系统。ZFS在Linux(ZoL)port是成熟的，但不推荐在生产环境上使用Docker的 ZFS存储方式，除非你大量ZFS文件系统的经验。2.	警惕ZFS内存问题，因为，ZFS最初是为了有大量内存的Sun Solaris服务器而设计 。3.	ZFS的“deduplication”特性，因为占用大量内存，推荐关掉。但如果食用SAN，NAS或者其他硬盘RAID技术，可以继续使用此特性。4.	ZFS caching特性适合高密度场景。5.	ZFS的128K块写，intent log及延迟写可以减少碎片产生。6. 和ZFS FUSE实现比较，推荐使用Linux原生ZFS驱动。 # 第四部分 总结
另外，下图列出Docker各种存储方式的优点缺点： 

![image](https://github.com/fanfanbj/sharing/blob/master/driver-pros-cons.png)
以上是五种Docker存储方式的介绍及分析，以此为理论依据，选择自己的Docker存储方式。同时可以做一些验证测试：如IO性能测试，以此确定适合自己应用场景的存储方式。同时，有两点值得提出：
1. 	使用SSD(Solid State Devices)存储，提高性能。2.	考虑使用数据卷挂载提高性能。# 参考1. [Docker storage drivers in Docker.com](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/)2. [Docker五种存储驱动原理及应用场景和性能测试对比](http://dockone.io/article/1513)3. [PPTV聚力传媒DCOS Docker存储方式的研究选型](http://dockone.io/article/1688)
4. [剖析Docker文件系统：Aufs与Devicemapper](http://www.infoq.com/cn/articles/analysis-of-docker-file-system-aufs-and-devicemapper/)5. [Linux 内核中的 Device Mapper 机制](https://www.ibm.com/developerworks/cn/linux/l-devmapper/)6. [Docker 环境 Storage Pool 用完解决方案：resize-device-mapper](http://segmentfault.com/a/1190000002931564)