#Docker存储方式选型建议
# 第一部分 问题诊断


WAS Server标准日志文件startServer.log和native_stderr.log都没有更加详细的错误信息。最后功夫不负有心人， 在configuration目录下找到可以定位的错误信息 =_=!：

![image](https://github.com/fanfanbj/share/blob/master/1/Picture2.jpg)

文件访问IO异常，查看相应目录文件的属性：

![image](https://github.com/fanfanbj/share/blob/master/1/Picture3.jpg)

到现在为止，可以初步判断是Docker存储方式(storage drive)在镜像容器分层管理上的问题。


![image](https://github.com/fanfanbj/share/blob/master/1/Picture4.jpg)

为了验证我们的推断，我们做了如下几方面的尝试：



![image](https://github.com/fanfanbj/share/blob/master/1/sharing-layers.jpg)

Docker存储方式提供管理分层镜像和容器的可读写层的具体实现。最初Docker仅能在支持AUFS文件系统的ubuntu发行版上运行，但是由于AUFS未能加入Linux内核，为了寻求兼容性、扩展性，Docker在内部通过graphdriver机制这种可扩展的方式来实现对不同文件系统的支持。







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






4. 在Snapshot上可以创建一个逻辑卷，这个逻辑卷在实际写操作（COW，Snapshot写操作）发生之前是不占用磁盘空间的。

相比AUFS和OverlayFS是文件级存储，Device mapper是块级存储，所有的操作都是直接对块进行操作，而不是文件。Device mapper驱动会先在块设备上创建一个资源池，然后在资源池上创建一个带有文件系统的基本设备，所有镜像都是这个基本设备的快照，而容器则是镜像的快照。所以在容器里看到文件系统是资源池上基本设备的文件系统的快照，并没有为容器分配空间。当要写入一个新文件时，在容器的镜像内为其分配新的块并写入数据，这个叫用时分配。当要修改已有文件时，再使用CoW为容器快照分配块空间，将要修改的数据复制到在容器快照中新的块里再进行修改。Device mapper 驱动默认会创建一个100G的文件包含镜像和容器。每一个容器被限制在10G大小的卷内，可以自己配置调整。结构如下图所示：
![image](https://github.com/fanfanbj/share/blob/master/1/dm_container.jpg)

可以通过"docker info"或通过dmsetup ls获取想要的更多信息。查看docker的Device mapper的信息：

![image](https://github.com/fanfanbj/share/blob/master/1/dm_docker_info.png)


###分析
Overlay是Linux内核3.18后支持的，也是一种Union FS，和AUFS的多层不同的是Overlay只有两层：一个upper文件系统和一个lower文件系统，分别代表Docker的镜像层和容器层。当需要修改一个文件时，使用CoW将文件从只读的lower复制到可写的upper进行修改，结果也保存在upper层。在Docker中，底下的只读层就是image，可写层就是Container。结构如下图所示：

![image](https://github.com/fanfanbj/share/blob/master/1/overlay_constructs.jpg)

###分析
1.	从kernel3.18进入主流Linux内核。设计简单，速度快，比AUFS和Device mapper速度快。在某些情况下，也比Btrfs速度快。是Docker存储方式选择的未来。因为OverlayFS只有两层，不是多层，所以OverlayFS “copy-up”操作快于AUFS。以此可以减少操作延时。



![image](https://github.com/fanfanbj/share/blob/master/1/btfs_container_layer.jpg)

###分析
1.	Btrfs是替换Device mapper的下一代文件系统， 很多功能还在开发阶段，还没有发布正式版本，相比EXT4或其它更成熟的文件系统，它在技术方面的优势包括丰富的特征，如：支持子卷、快照、文件系统内置压缩和内置RAID支持等。
ZFS 文件系统是一个革命性的全新的文件系统，它从根本上改变了文件系统的管理方式，ZFS 完全抛弃了“卷管理”，不再创建虚拟的卷，而是把所有设备集中到一个存储池中来进行管理，用“存储池”的概念来管理物理存储空间。过去，文件系统都是构建在物理设备之上的。为了管理这些物理设备，并为数据提供冗余，“卷管理”的概念提供了一个单设备的映像。而ZFS创建在虚拟的，被称为“zpools”的存储池之上。每个存储池由若干虚拟设备（virtual devices，vdevs）组成。这些虚拟设备可以是原始磁盘，也可能是一个RAID1镜像设备，或是非标准RAID等级的多磁盘组。于是zpool上的文件系统可以使用这些虚拟设备的总存储容量。Docker的ZFS利用snapshots和clones，它们是ZFS的实时拷贝，snapshots是只读的，clones是读写的，clones从snapshot创建。



###分析
另外，下图列出Docker各种存储方式的优点缺点： 

![image](https://github.com/fanfanbj/share/blob/master/1/driver-pros-cons.png)


4. [剖析Docker文件系统：Aufs与Devicemapper](http://www.infoq.com/cn/articles/analysis-of-docker-file-system-aufs-and-devicemapper/)