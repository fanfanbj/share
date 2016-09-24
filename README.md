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
尝试3: 在其他的宿主机上，启动原镜像，这个问题是无法复现的。综上所述，我们得到了一个结论，这个问题的根本原因是overlayFS在xfs上出现了兼容性的问题。同时，我们从Docker社区找到相关问题的issue报告：https://github.com/docker/docker/issues/9572这个问题的修改在内核 4.4.6以上。事情的起因到此为止，下面让我们深入的看看Docker的存储方式及其比较，并给出一些技术选型的建议。# 第二部分 概述Docker在启动容器的时候，需要创建文件系统，为rootfs提供挂载点。最底层的引导文件系统bootfs主要包含 bootloader和kernel，bootloader主要是引导加载kernel，当kernel被加载到内存中后 bootfs就被umount了。 rootfs包含的就是典型 Linux 系统中的/dev，/proc，/bin，/etc等标准目录和文件。Docker 模型的核心部分是有效利用分层镜像机制，镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像。Docker1.10引入新的可寻址存储模型，使用安全内容哈希代替随机的UUID管理镜像。同时，Docker提供了迁移工具，将已经存在的镜像迁移到新模型上。不同 Docker 容器就可以共享一些基础的文件系统层，同时再加上自己独有的可读写层，大大提高了存储的效率。其中主要的机制就是分层模型和将不同目录挂载到同一个虚拟文件系统。

![image](https://github.com/fanfanbj/sharing/blob/master/sharing-layers.jpg)

Docker存储方式提供管理分层镜像和Docker容器自己的可读写层的具体实现。最初Docker仅能在支持AUFS文件系统的ubuntu发行版上运行，但是由于AUFS未能加入Linux内核，为了寻求兼容性、扩展性，Docker在内部通过graphdriver机制这种可扩展的方式来实现对不同文件系统的支持。Docker有如下几种不同的drivers：
* AUFS* Device mapper* Btrfs* OverlayFS* ZFS
# 第三部分 方案选择
## AUFS
AUFS（AnotherUnionFS）是一种联合文件系统。所谓 UnionFS 就是把不同物理位置的目录合并 mount 到同一个目录中。UnionFS 的一个最主要的应用是，把一张 CD/DVD 和一个硬盘目录给联合 mount 在一起，然后就可以对这个只读的 CD/DVD 上的文件进行修改（当然，修改的文件存于硬盘上的目录里）。 AUFS 支持为每一个成员目录（类似 Git 的分支）设定只读（readonly）、读写（readwrite）和写出（whiteout-able）权限, 同时 AUFS 里有一个类似分层的概念, 对只读权限的分支可以逻辑上进行增量地修改(不影响只读部分的)。
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
### 性能分析
#### 优点： 
* Docker 第一版支持, 性能稳定，并且有大量生产部署及丰富的社区支持
* AUFS mount() 方法很快，所以创建容器也很快。
* 读写访问都具有本机效率(一旦找到后)
* 顺序读写和随机读写的性能大于kvm
* 有效的使用存储和内存#### 缺点：
* AUFS 到现在还没有加入内核主线( centos 无法直接使用)
* AUFS不支持rename系统调用，将失败当执行“copy”和“unlink”
* 当写入大文件的时候(比如日志或者数据库..)动态mount多目录路径的问题,导致branch越多，查找文件的性能也就越慢。(解决办法:重要数据直接使用 -v 参数挂载到系统盘。同时启动1000个一样的容器，数据只从磁盘加载一次，缓存也只从内存加载一次。)

## Device mapper
Device mapper是Linux内核2.6.9后支持的，提供的一种从逻辑设备到物理设备的映射框架机制，在该机制下，用户可以很方便的根据自己的需要制定实现存储资源的管理策略。Docker的Device mapper利用 thin provisioning and snapshotting管理镜像和容器。AUFS和OverlayFS都是文件级存储，而Device mapper是块级存储，所有的操作都是直接对块进行操作，而不是文件。Device mapper驱动会先在块设备上创建一个资源池，然后在资源池上创建一个带有文件系统的基本设备，所有镜像都是这个基本设备的快照，而容器则是镜像的快照。所以在容器里看到文件系统是资源池上基本设备的文件系统的快照，并不有为容器分配空间。当要写入一个新文件时，在容器的镜像内为其分配新的块并写入数据，这个叫用时分配。当要修改已有文件时，再使用CoW为容器快照分配块空间，将要修改的数据复制到在容器快照中新的块里再进行修改。Device mapper 驱动默认会创建一个100G的文件包含镜像和容器。每一个容器被限制在10G大小的卷内，可以自己配置调整。结构如下图所示：
![image](https://github.com/fanfanbj/sharing/blob/master/dm_container.jpg)

###性能分析####优点：1.	Docker的Device mapper默认模式是loop-lvm，性能达不到生产级别要求。在生产级别推荐direct-lvm模式。Direct-lvm模式直接写原块设备，性能好2.	兼容性比较好3.	因为存储为1个文件，减少了inond消耗4.	为了更好的性能，Data和Metadata文件使用高速存储 如：SSD。
####缺点：1.	每次一个容器写数据都是一个新块，块必须从池中分配，真正写的时候是稀松文件,虽然它的利用率很高，但性能不好，因为额外增加了vfs开销2.	每个容器都有自己的块设备时，它们是真正的磁盘存储，所以当启动N个容器时，它都会从磁盘加载N次到内存，消耗内存大。
4. 默认存储池只有100GB5. 是所有空间是静态值 ## OverlayFS


# 参考1.	[Docker storage drivers in Docker.com](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/)2.	[剖析Docker文件系统：Aufs与Devicemapper](http://www.infoq.com/cn/articles/analysis-of-docker-file-system-aufs-and-devicemapper/)3.	 [Linux 内核中的 Device Mapper 机制](https://www.ibm.com/developerworks/cn/linux/l-devmapper/)4.	[Docker 环境 Storage Pool 用完解决方案：resize-device-mapper](http://segmentfault.com/a/1190000002931564)
5. 	[Docker五种存储驱动原理及应用场景和性能测试对比](http://dockone.io/article/1513)