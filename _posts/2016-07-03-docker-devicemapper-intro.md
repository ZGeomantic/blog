---
layout: post
title:  "Docker笔记——device mapper的原理"
date:   2016-07-03 20:30:15 +0800
categories: my article
---

Docker的三大技术基石namespace，cgroup，镜像技术各司其职，给应用的构建、交付、运行方面带来了巨大的便利，才有了今天在云计算时代的大行其道。其中借助Linux内核的namespace和cgroup等特性，实现了运行环境的资源隔离和限制。但真正让Docker能够全盘打包运行环境、构建容器的靠的是镜像技术(image)，这个是Docker文件管理系统的基础。

Docker的镜像的基础是分层(layer)，构建新的镜像是通过在父镜像上添加新的layer来实现的。前期Docker主要依赖于UnionMount技术，以aufs为代表。但是由于aufs未能加入Linux内核，为了寻求兼容性、扩展性，加入了Devicemapper的支持。目前，除少数版本如Ubuntu，Docker基本运行在Devicemapper基础上。


>注意: aufs 被作第一优先级（还包括其他 unionFS，如 overlay ），device mapper 作为第二优先级。实际上，[有很多人抱怨](https://forums.docker.com/t/rmi-not-freeing-disk-space-in-devicemapper-sparse-file-centos-6-6/1640/4) deviemapper 在存储上面会遇到一系列很麻烦的问题，比如当 metadata 和 data 空间被耗尽时，需要重启 docker 来扩充空间（如果这句话现在还不理解，请看完全文后在回来看一遍）。甚至连 docker 自己的 Twitter 都说没见过这家伙工作稳定。关于这些文件系统的优劣分析，可以[参考这里](http://www.projectatomic.io/docs/filesystems/)。


### 1  Aufs
网上关于aufs的定义和机制的阐述已经很多了，基本上只要是介绍docker结构的文章都会提到aufs，所以这些内容不再详述，只是通过命令行的方式展示一下如何手动实现aufs。非常简单，只有要一个mnt命令：

> 能这样使用mnt的前提当然是系统支持 aufs，如何验证当前系统是否支持呢？可以通过命令 modprobe来验证。如果执行命令  ```modeprobe aufs``` 后没有返回错误 ```modprobe: FATAL: Module devicemapper not found.``` 就说明当前系统支持aufs。实际上docker本身在源码里也是这么验证的。

```shell
root@Standard-PC:/tmp# tree
.
├── dir-union
├── dir1
│   └── file1
└── dir2
    └── file2
root@Standard-PC:/tmp# sudo mount -t aufs -o br=/tmp/dir1=ro:/tmp/dir2=rw none /tmp/dir-union
mount: warning: /tmp/aufs seems to be mounted read-only.
```
- -o 指定mount传递给文件系统的参数
- br 指定需要挂载的文件夹，这里包括dir1和dir2
- ro/rw 指定文件的权限只读和可读写
- none 这里没有设备，用none表示

这就完成了把dir1、dir2通过aufs方式挂载到dir-union目录下，非常简单。上面的命令执行完后，dir-union目录下会出现file1和file2，其中file1只读，file2可读写。在dir-union下修改文件和在dir2下修改文件的效果是等同的。


### 2 Device mapper

与于aufs相比，device mapper在网上的文章要少的多。虽然这个东西在docker中可能不是那么的可靠，但是了解原理总是没有错的。本节会分析devicemapper的结构、它与thin-provison snapshot的关系，以及如何使用。

#### 2.1 device mapper的结构 
Device mapper主要由三个组件单元：mapped device(逻辑设备)，mapping table（映射关系表），target device（物理设备，或已存在的设备）。

它们之间的相互关系用一句话概括就是： 在mapping table中注册一系列的映射规则，这些规则描述如何把一个或多个target device按照给定的起始地址和偏移量映射到对应的mapped device中，从而创建出一个虚拟的逻辑设备mapped device。结构如下图所示：

![image](http://www.ibm.com/developerworks/cn/linux/l-devmapper/images/image004.gif)

很明显这是一个树状结构，可以用一个已存在的mapped device作为另一个mapped device的target device。整个过程就像目录结构一样可以无限嵌套下去。

#### 2.2 device mapper的snapshot与docker的关系
在图上中，除了上面提到的三个组件外，还有一个与mapping table紧密相连的组件target driver。这是另一个很重要的模块化插件，device mapper正是通过target driver来实现对 IO 请求的过滤或者重新定向等工作。目前已经实现的target driver插件有很多种，比如 linear、mirror、snapshot、multipath等，在这诸多“插件”中，有一个东西叫Thin Provisioning Snapshot，这是Docker使用DeviceMapper中最重要的模块。

理解了上面这一层关系，我们对device mapper就可以做一个[比较完整的定义](http://www.ibm.com/developerworks/cn/linux/l-devmapper/index.html)了：

>Device Mapper 是 Linux2.6 内核中支持逻辑卷管理的通用设备映射机制，它为实现用于存储资源管理的块设备驱动提供了一个高度模块化的内核架构，
>
>Device mapper本质功能就是根据映射关系( mapping table )和 target driver描述的 IO处理规则，将IO请求从逻辑设备 mapped device转发相应的 target device上。IO请求在 device mapper的设备树中通过请求转发从上到下地进行处理。当一个 bio请求在设备树中的 mapped deivce向下层转发时，一个或者多个 bio的克隆被创建并发送给下层 target device。然后相同的过程在设备树的每一个层次上重复，只要设备树足够大理论上这种转发过程可以无限进行下去。在设备树上某个层次中，target driver结束某个 bio请求后，将表示结束该 bio请求的事件上报给它上层的 mapped device，该过程在各个层次上进行直到该事件最终上传到根 mapped device的为止，然后 device mapper结束根 mapped device上原始 bio请求，结束整个 IO请求过程。


#### 2.3 docker如何使用devicemapper

Docker是怎么使用Thin Provisioning这个技术做到像UnionFS那样的分层镜像的呢？答案是，Docker使用了Thin Provisioning的Snapshot的技术。下面我们[尝试自己搞一个](http://www.infoq.com/cn/articles/analysis-of-docker-file-system-aufs-and-devicemapper)：


---

- **第一步，我们需要创建两个文件**

```shell
[geomantic@coder ~]$ sudo dd if=/dev/zero of=/tmp/data.img bs=1K count=1 seek=10G
1+0 records in
1+0 records out
1024 bytes (1.0 kB) copied, 0.000621428 s, 1.6 MB/s
 
[geomantic@coder ~]$ sudo dd if=/dev/zero of=/tmp/meta.data.img bs=1K count=1 seek=1G
1+0 records in
1+0 records out
1024 bytes (1.0 kB) copied, 0.000140858 s, 7.3 MB/s
```

以上面的第一条命令为例，意思是以/dev/zero为输入文件，创建一个文件/tmp/data.img，略过10G空间后写入1K内容，只有一个block块。所以也就是10G的尺寸，但其实在硬盘上是没有占有空间的，占有空间只有1k的内容。当向其写入内容时，才会在硬盘上为其分配空间。

> /dev/zero,是一个输入设备，你可你用它来初始化文件。dd命令一般用来进行设备间的文件拷贝工作。



- **第二步，把文件挂载到loopback设备上，用上一步创建出的两个文件虚拟化成两个设备**

```shell 
[geomantic@coder ~]$ losetup /dev/loop2015 /tmp/meta.data.img
[geomantic@coder ~]$ losetup /dev/loop2016 /tmp/data.img
[geomantic@coder ~]$ losetup -a
/dev/loop2015: [0035]:183033 (/tmp/meta.data.img)
/dev/loop2016: [0035]:185651 (/tmp/data.img)
```

> losetup命令可把文件虚拟成块设备(Block Device),借以模拟整个文件系统，让用户得以将其视为硬盘、光驱或者软驱等设备，并加载当作目录来使用

为什么要创建loopback设备？因为device mapper是对block设备而言的，不是文件（UnionFS才是针对文件的，所以才那么方便）。



- **第三步，为设备创建一个thin-pool，也就是Device mapper的设备（以后的snapshot都是对它做）**

```shell
[geomantic@coder ~]$ dmsetup create geomantic-thin-pool --table "0 20971520 thin-pool 
/dev/loop2016 /dev/loop2015 128 32768 1 skip_block_zeroing"
```

此命令的意思是：创建一个名叫geomantic-thin-pool的pool，使用的target driver类型是thin-pool，将逻辑设备的0～20971520之间的sector映射到/dev/loop2015上，元数据信息存储到/dev/loop2016。这个设备最小分配128个sector，如果实际使用的sector数超过了32768就要扩容。最后的"1 skip_block_zeroing"意思是还有一个附加参数skip_block_zeroing，表示略过用0填充的块。

然后，我们就可以在/dev/mapper/目录下看到我们创建的Device mapper设备了：

```shell
[geomantic@coder ~]$ sudo ll /dev/mapper/geomantic-thin-pool
lrwxrwxrwx. 1 root root 1 July 25 21:00 /dev/mapper/geomantic-thin-pool -> ../dm-4
```



- **第四步，创建一个Thinly-Provisioned volume并格式化**

```shell
[geomantic@coder ~]$ dmsetup message /dev/mapper/geomantic-thin-pool 0 "create_thin 0"
[geomantic@coder ~]$ dmsetup create geomantic-thin-volume1 --table "0 2097152 thin /dev/mapper/geomantic-thin-pool 0"
[geomantic@coder ~]$ mkfs.ext4 -E discard,lazy_itable_init=0 /dev/mapper/geomantic-thin-volume1
```
- 第一个命令中的create_thin是关键字，后面的0表示这个Volume的device的id
- 第二个命令，是真正的为这个Volumn创建一个可以mount的设备，名字叫geomantic-thin-volume1。2097152个sector等于有1GB，id为0。
- 第三个命令用于格式化刚刚创建的volume


- **第五步，建立一个internal snapshot**

在thin volume的基础上建立一个快照

```shell
[geomantic@coder ~]$ dmsetup suspend /dev/mapper/geomantic-thin-volume1
[geomantic@coder ~]$ dmsetup message /dev/mapper/geomantic-thin-pool 0 "create_snap 1 0"
[geomantic@coder ~]$ dmsetup resume /dev/mapper/geomantic-thin-volume1
[geomantic@coder ~]$ dmsetup create mysnap --table "0 2097152 thin /dev/mapper/geomantic-thin-pool 1"
```

- 第二条命令是向geomantic-thin-pool发消息，后面跟着两个id，第一个是新的dev id，第二个是从哪个已有的dev id上做snapshot
- 第四条命令，是创建一个名为mysnap的device

从命令行上的相似也可以看出来，创建一个volume和创建一个snapshot其实是很接近的。实际上，docker的读写层中的变化，都体现在每一次新创建的snapshot上。我们可以
看一下此时的设备状态：

```shell
[geomantic@coder ~]$ dmsetup status
geomantic-thin-volume1: 0 2097152 thin 99840 2097151
mysnap: 0 2097152 thin 99840 2097151
geomantic-thin-pool: 0 20971520 thin-pool 0 280/4161600 780/163840 - rw discard_passdown queue_if_no_space 
```

---

### 3. 小结
至此，我们已经完成了devicemapper从创建设备到建立快照的每一步，如果还没有理解镜像与snapshot的关系，那么我们可以分别把/dev/mapper/geomantic-thin-volume1和/dev/mapper/mysnap挂载出来看：

```shell
# 挂载volume到/mnt/base，并创建文件id.txt
[geomantic@coder ~]$ sudo mkdir -p /mnt/base
[geomantic@coder ~]$ sudo mount /dev/mapper/geomantic-thin-volumn1 /mnt/base
[geomantic@coder ~]$ sudo echo "hello world, I am a base" > /mnt/base/id.txt

[geomantic@coder ~]$ sudo cat /mnt/base/id.txt
hello world, I am a base

# 挂载mysnap到/mnt/mysnap，可以看到里面有id.txt文件
[geomantic@coder ~]$ sudo mkdir -p /mnt/mysnap
[geomantic@coder ~]$ sudo mount /dev/mapper/mysnap /mnt/mysnap

[geomantic@coder ~]$ sudo ll /mnt/mysnap/
total 20
-rw-r--r--. 1 root root 25 Aug 25 23:46 id.txt
drwx------. 2 root root 16384 Aug 25 23:43 lost+found

[geomantic@coder ~]$ sudo cat /mnt/mysnap1/id.txt
hello world, I am a base
```
此时，如果我们对/mnt/mysnap目录下的文件做操作，是不会影响/mnt/base的任何内容的。这是不是很像image中最上层的读写layer呢？



> devicemapper并不属于 UnionFS，只能通过不断创建新快照的方式，来实现每层镜像的读写层。

> 而 aufs因为可以直接把目录挂载到一个目录下，所以每添加一个读写层就是创建一个新的文件夹，操作比较简单，完全由底层控制。



参考文章：

- http://www.infoq.com/cn/articles/analysis-of-docker-file-system-aufs-and-devicemapper
- http://coolshell.cn/articles/17200.html
- http://www.ibm.com/developerworks/cn/linux/l-devmapper/index.html
- http://www.projectatomic.io/docs/filesystems/


