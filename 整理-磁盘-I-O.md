---
title: '[整理] 磁盘 I/O'
date: 2019-11-03 20:21:13
tags: 
  - 高性能
  - System
---

注: 本文非原创, 只是对网上一些内容进行了整理总结.

# Linux 文件系统简介

### 影响硬盘性能的因素
影响磁盘的关键因素是磁盘服务时间, 即磁盘完成一个I/O请求所花费的时间, 它由寻道时间、旋转延迟和数据传输时间三部分构成. 

#### 1. 寻道时间

Tseek是指将读写磁头移动至正确的磁道上所需要的时间. 寻道时间越短, I/O操作越快, 目前磁盘的平均寻道时间一般在3-15ms. 

#### 2. 旋转延迟

Trotation是指盘片旋转将请求数据所在的扇区移动到读写磁盘下方所需要的时间. 旋转延迟取决于磁盘转速, 通常用磁盘旋转一周所需时间的1/2表示. 比如：7200 rpm 的磁盘平均旋转延迟大约为60*1000/7200/2 = 4.17ms, 而转速为15000rpm的磁盘其平均旋转延迟为2ms. 

#### 3. 数据传输时间

Ttransfer是指完成传输所请求的数据所需要的时间, 它取决于数据传输率, 其值等于数据大小除以数据传输率. 目前IDE/ATA能达到133MB/s, SATA II可达到300MB/s的接口数据传输率, 数据传输时间通常远小于前两部分消耗时间. 简单计算时可忽略. 

### 衡量性能的指标
机械硬盘的连续读写性能很好, 但随机读写性能很差, 这主要是因为磁头移动到正确的磁道上需要时间, 随机读写时, 磁头需要不停的移动, 时间都浪费在了磁头寻址上, 所以性能不高. 衡量磁盘的重要主要指标是IOPS和吞吐量. 

#### IOPS

IOPS（Input/Output Per Second）即每秒的输入输出量（或读写次数）, 即指每秒内系统能处理的I/O请求数量. 随机读写频繁的应用, 如小文件存储等, 关注随机读写性能, IOPS是关键衡量指标. 可以推算出磁盘的IOPS = 1000ms / (Tseek + Trotation + Transfer), 如果忽略数据传输时间, 理论上可以计算出随机读写最大的IOPS. 常见磁盘的随机读写最大IOPS为：

- 7200rpm的磁盘 IOPS = 76 IOPS
- 10000rpm的磁盘IOPS = 111 IOPS
- 15000rpm的磁盘IOPS = 166 IOPS

虽然15000rpm的磁盘计算出的理论最大IOPS仅为166, 但在实际运行环境中, 实际磁盘的IOPS往往能够突破200甚至更高. 这其实就是在系统调用过程中, 操作系统进行了一系列的优化(page cache, 预读). 

#### 吞吐量
吞吐量（Throughput）, 指单位时间内可以成功传输的数据数量. 顺序读写频繁的应用, 如视频点播, 关注连续读写性能、数据吞吐量是关键衡量指标. 它主要取决于磁盘阵列的架构, 通道的大小以及磁盘的个数. 不同的磁盘阵列存在不同的架构, 但他们都有自己的内部带宽, 一般情况下, 内部带宽都设计足够充足, 不会存在瓶颈. 磁盘阵列与服务器之间的数据通道对吞吐量影响很大, 比如一个2Gbps的光纤通道, 其所能支撑的最大流量仅为250MB/s. 最后, 当前面的瓶颈都不再存在时, 硬盘越多的情况下吞吐量越大. 

# read / write

在现代操作系统中, 一个 “真正的” 文件, 当调用 `read` / `write` 的时候, 数据当然不会简单地就直达硬盘. 对于 Linux 而言, 这个过程的**一部分**是这样的：

![](整理-磁盘-I-O/588476624-59fadc92da61c_articlex.jpeg)

在操作系统内核空间内, read / write 到硬件设备之间, 按顺序有这么几层：

*   **VFS**：虚拟文件系统, 可以大致理解为 `read` / `write` / `ioctl` 之类的系统调用就在这一层. 当调用 `open` 之后, 内核会为每一个 file descriptor 创建一个 `file_operations` 结构体实例. 这个结构体里包含了 open、write、seek 等的实例（回调函数）. 这一层其实是 Linux 文件和设备体系的精华之一, 很多东西都隐藏或暴露在这一层. 不过本文不研究这一块
*   **文件系统**： 这一层是实际的文件系统实现层, 向上隐藏了实现细节. 当然, 实际上除了文件系统之外, 还包含其他的虚拟文件, 包括设备节点、`/proc` 文件等等
*   **buffer cache**：这就是本文所说的 “缓存”. 后文再讲. 
*   **设备驱动**：这是具体硬件设备的设备驱动了, 比如 SSD 的读写驱动、磁盘的读写驱动、字符设备的读写驱动等等. 
*   **硬件设备**：这没什么好讲的了, 就是实际的硬件设备接口. 参见[上一篇文章](https://segmentfault.com/a/1190000011743916)

#### 从系统调用角度

![](整理-磁盘-I-O/20191103212251.png)

1. 进程向内核发起一个系统调用, 
2. 内核接收到系统调用, 知道是对文件的请求, 于是告诉磁盘, 把文件读取出来
3. 磁盘接收到来着内核的命令后, 把文件载入到内核的内存空间里面 (第一次 copy)
4. 内核的内存空间接收到数据之后, 把数据 copy 到用户进程的内存空间(第二次 copy)
5. 进程内存空间得到数据后, 给内核发送通知
6. 内核把接收到的通知回复给进程, 此过程为唤醒进程, 然后进程得到数据, 进行下一步操作

下面举例说明一个阻塞的文件 IO: 从 OS 角度看, 进程A执行过程中, 进行IO操作, IO操作引起进程A阻塞, 等待一个外部事件的发生（外部事件发生后, 该进程A才可以由阻塞态转为就绪态）, 同时, 上下文切换, 另一个进程B获得时间片执行. 在此期间, 阻塞态的进程A执行底层的IO处理（设备控制器和设备交换数据）, 进程B执行CPU操作. 当进程A完成IO处理后, 相应的外部事件（设备控制器发出中断请求信号）发生, 当前运行的进程B被中断, 进行外部事件处理, CPU和设备控制器之间传输数据, 完成后, 系统修改阻塞进程A的状态为就绪态. 然后, 依据剥夺或非剥夺调度算法, 选择被中断的进程B, 或刚解除阻塞的进程A, 或其它就绪进程C执行. 

虽然 CPU 的时间片没有浪费, 但是进程/线程的上下文切换代价高.

##### MMAP

mmap 把文件映射到用户空间里的虚拟内存, 省去了从内核缓冲区复制到用户空间的过程, 文件中的位置在虚拟内存中有了对应的地址, 可以像操作内存一样操作这个文件, 相当于已经把整个文件放入内存, 但在真正使用到这些数据前却不会消耗物理内存, 也不会有读写磁盘的操作, 只有真正使用这些数据时, 虚拟内存管理系统 VMS 才根据缺页加载的机制从磁盘加载对应的数据块到物理内存进行渲染. 这样的文件读写文件方式少了数据从内核缓存到用户空间的拷贝, 效率很高.

比较适用小文件随机 IO, 比如说索引

1. 要提前确定大小
2. Java 中的 mmap 回收比较蛋疼
3. MMAP 使用的是虚拟内存, 和 PageCache 一样是由操作系统来控制刷盘的, 可以通过 force 强刷

![](整理-磁盘-I-O/20191103215919.png)

###### 性能测试(Java)
- 文件大小 1.5G
- 读取方式:顺序读取
- 读取时间

    冷数据(排除 page cache):
    
        1. FileChannel:10153ms
        2. FileChannel(use direct memory):8881ms
        3. Mmeory mapping:8111ms
        4. BufferedInputStream:8252ms
    
    热数据(有 page cache):
    
        1. FileChannel:2262ms
        2. FileChannel(use direct memory):1354ms
        3. Mmeory mapping:1305ms
        4. BufferedInputStream:1961ms

##### Zero copy
用于网络传输减少上下文切换和 copy.

###### 普通流程(read/send)

1. OS 将数据从磁盘复制到操作系统内核的页缓存中 (磁盘(DMA) -> kernel buffer, U -> K)
2. 应用将数据从内核空间复制到应用的用户空间 (kernel buffer -> user space buffer, K -> U)
3. 应用将数据复制回内核态的 Socket 缓存中 (user space buffer -> kernel buffer, U -> K)
4. OS 将Socket缓存复制到网卡发出 (kernel buffer -> 网卡(DMA))

1,2 步是 read, 3,4 步是 send, 整个过程总共发生了四次拷贝和四次的用户态和内核态的切换. 

![](整理-磁盘-I-O/20191103221636.png)

###### Zero copy (sendfile)

1. OS 将数据从磁盘复制到操作系统内核的页缓存中 (磁盘(DMA) -> kernel buffer, U -> K)
2. 内核的页缓存复制到内核态的 Socket 缓存中 (kernel buffer -> Socket kernel buffer)
3. OS 将Socket缓存复制到网卡发出 (kernel buffer -> 网卡(DMA))

![](整理-磁盘-I-O/20191103221734.png)

如果网卡支持 gather operations 内核就可以进一步减少数据拷贝:

1. OS 将数据从磁盘复制到操作系统内核的页缓存中 (磁盘(DMA) -> kernel buffer, U -> K)
2. 无数据被复制到 Socket buffer. 只是描述了需要被写入的数据的位置和长度. DMA直接把数据从 kernel buffer 复制到网卡(DMA gather copy)

#### 延伸
DMA（直接内存访问, Direct Memory Access）可以不经过CPU而直接进行磁盘和内存的数据交换. 在DMA模式下, CPU只需要向DMA控制器下达指令, 让DMA控制器来处理数据的传送即可, DMA控制器通过系统总线来传输数据, 传送完毕再通知CPU, 这样就在很大程度上降低了CPU占有率, 大大节省了系统资源

没有 DMA, 磁盘数据 copy 到内核需要 CPU 参与.

# 工作机制

### 通用块层
硬盘中每个扇区的大小固定为512字节, 扇区是最小的可寻址单元.

对于VFS和具体的文件系统来说, 数据是以块(Block)为单位进行管理的, 当内核访问文件的数据时, 它首先从磁盘上读取一个块, 由于扇区是磁盘的最小可寻址单元, 所以块不能比扇区还小, 只能整数倍于扇区大小, 即一个块对应磁盘上的一个或多个扇区. 而且由于Page Cache层的最小单元是页（Page）, 所以块大小不能超过一页的长度. 每块一般设定为4Kb, 当OS需要获取磁盘某个数据时, 将产生一个I/O中断, 本次I/O中断将带上具体的块号去驱动磁盘进行块数据查找和读取操作, 这些操作都是以某块作为起点, 每次读取数据的最小单位也是4Kb.

通用块层是粘合所有上层和底层的部分, 一个页的磁盘数据布局如下图所示：

![](整理-磁盘-I-O/20191103210628.png)

[IBM 的资料](https://www.ibm.com/developerworks/cn/linux/l-cache/index.html) 摘抄：

当应用程序需要读取文件中的数据时, 操作系统先分配一些内存, 将数据从存储设备读入到这些内存中, 然后再将数据分发给应用程序；当需要往文件中写数据时, 操作系统先分配内存接收用户数据, 然后再将数据从内存写到磁盘上. 

对于每个文件的**第一个**读请求, 系统读入所请求的页面并读入紧随其后的少数几个页面(不少于一个页面, 通常是三个页面), 这时的预读称为**同步预读**. 

如果应用程序接下来是顺序读取的话, 那么文件 cache 命中, OS 会加大同步预读的范围, 增强缓存效率, 此时的预读被称为**异步预读**

如果接下来 cache 没命中, 那么 OS 会继续使用同步预读. 

### PAGE CACHE层

引入Cache层的目的是为了提高Linux操作系统对磁盘访问的性能. Cache层在内存中缓存了磁盘上的部分数据. 当数据的请求到达时, 如果在Cache中存在该数据且是最新的, 则直接将数据传递给用户程序, 免除了对底层磁盘的操作, 提高了性能. Cache层也正是磁盘IOPS为什么能突破200的主要原因之一. 

在Linux的实现中, 文件Cache分为两个层面, 一是Page Cache, 另一个Buffer Cache, 每一个Page Cache包含若干Buffer Cache. Page Cache主要用来作为文件系统上的文件数据的缓存来用, 尤其是针对当进程对文件有read/write操作的时候. Buffer Cache则主要是设计用来在系统对块设备进行读写的时候, 对块进行数据缓存的系统来使用. 

磁盘Cache有两大功能：预读和回写. 预读其实就是利用了局部性原理, 具体过程是：对于每个文件的第一个读请求, 系统读入所请求的页面并读入紧随其后的少数几个页面（通常是三个页面）, 这时的预读称为同步预读. 对于第二次读请求, 如果所读页面不在Cache中, 即不在前次预读的页中, 则表明文件访问不是顺序访问, 系统继续采用同步预读；如果所读页面在Cache中, 则表明前次预读命中, 操作系统把预读页的大小扩大一倍, 此时预读过程是异步的, 应用程序可以不等预读完成即可返回, 只要后台慢慢读页面即可, 这时的预读称为异步预读. 任何接下来的读请求都会处于两种情况之一：第一种情况是所请求的页面处于预读的页面中, 这时继续进行异步预读；第二种情况是所请求的页面处于预读页面之外, 这时系统就要进行同步预读. 

回写是通过暂时将数据存在Cache里, 然后统一异步写到磁盘中. 通过这种异步的数据I/O模式解决了程序中的计算速度和数据存储速度不匹配的鸿沟, 减少了访问底层存储介质的次数, 使存储系统的性能大大提高. Linux 2.6.32内核之前, 采用pdflush机制来将脏页真正写到磁盘中, 什么时候开始回写呢？下面两种情况下, 脏页会被写回到磁盘：

在空闲内存低于一个特定的阈值时, 内核必须将脏页写回磁盘, 以便释放内存. 
当脏页在内存中驻留超过一定的阈值时, 内核必须将超时的脏页写会磁盘, 以确保脏页不会无限期地驻留在内存中. 
回写开始后, pdflush会持续写数据, 直到满足以下两个条件：

1. 已经有指定的最小数目的页被写回到磁盘. 
2. 空闲内存页已经回升, 超过了阈值. 

Linux 2.6.32内核之后, 放弃了原有的pdflush机制, 改成了bdi_writeback机制. bdi_writeback机制主要解决了原有fdflush机制存在的一个问题：在多磁盘的系统中, pdflush管理了所有磁盘的Cache, 从而导致一定程度的I/O瓶颈. bdi_writeback机制为每个磁盘都创建了一个线程, 专门负责这个磁盘的Page Cache的刷新工作, 从而实现了每个磁盘的数据刷新在线程级的分离, 提高了I/O性能. 

回写机制存在的问题是回写不及时引发数据丢失（可由sync|fsync解决）, 回写期间读I/O性能很差. 

#### 清除 Cache
```shell
#清理 pagecache (页缓存)
sysctl -w vm.drop_caches=1

#清理 dentries（目录缓存）和 inodes
sysctl -w vm.drop_caches=2

#清理 pagecache、dentries 和 inodes
sysctl -w vm.drop_caches=3
```

## 高性能硬盘 I/O 优化方案

### 基本思路

从缓存的工作机制来看, 很简单, 如果要充分利用 Linux 的文件缓存机制, 那么最好的方法就是：每一个文件都尽可能地采用**顺序读写**, 避免大量的 `seek` 调用. 

### 尽可能顺序地读写一个文件

从文件缓存角度, 如果频繁地随机读取一个文件不同的位置, 很可能导致缓存命中率下降. 那么 OS 就不得不频繁地往硬盘上预读, 进一步导致硬盘利用率低下. 所以在读写文件的时候, 尽可能的只是简单写入或者简单读取文件, 而不要使用 `seek`. 

这条原则非常适用于 log 文件的写入：当写入 log 的时候, 写就好了, 不要经常翻回去查看以前的内容. 

### 单进程读写硬盘

整个系统, 最好只有一个进程进行磁盘的读写. 而不是多个进程进行文件存取. 这个思路, 一方面和上一条 “顺序写” 原则的理由其实是一致的. 当多个进程进行磁盘读写的时候, 随机度瞬间飙升. 特别是多个进程操作多个文件的时候, 磁盘的磁头很可能需要频繁大范围地移动. 

如果确实有必要多个进程分别读取多个不同文件的话, 可以考虑下面的替代方案：

*   这多个进程是否功能上是独立的？能不能分开放在几个不同的服务器之中？
*   如果这几个进程确实需要放在同一台服务器上, 那么能不能考虑为每个频繁读写的文件, 单独分配一个磁盘？
*   如果成本允许, 并且文件大小不大的话, 能否将磁盘更换为 SSD ？因为 SSD 没有磁头和磁盘的物理寻址动作, 响应会快很多. 

如果是多个进程同时写入一个文件（比如 log）, 那就更好办了. 这种情况下, 可以在这几个进程和文件中间加入一个内部文件服务器, 将所有进程的存取文件需求汇总到该文件服务器中进行统一处理. 

```
ProcessA   ProcessB   ProcessC
   |          |          |
   |          V          |
   *---->  The File  <---*
```

改为

```
ProcessA   ProcessB   ProcessC
   |          |          |
   |          V          |
   *---->  ProcessD  <---*
              |
              V
           The File
```

顺便还可以在这个服务进程中实现一些自己的缓存机制, 配合 Linux 自身的文件缓存进一步优化磁盘 I/O 效率. 

#### 以 4kB 为单位写文件

这里可以看看下面这个伪代码：

```
const int WRITE_BLOCK_SIZE = 4096

for (int i = 0 to 999) {
    write(fd, buff, WRITE_BLOCK_SIZE)
}
```

其实这个问题, 就是我在上一篇文章的 ["硬盘文件存取速度的考量"](https://segmentfault.com/a/1190000011743916#articleHeader2) 小节中所说的内容了. 
这里有一个常量 `WRITE_BLOCK_SIZE`, 这并不是可以随意取的值, 比较合适的是 4096 或者其倍数, 理由是文件系统往往以 4kB 为页, 如果没有写够 4kB 的话, 将导致文件需要多余的读出动作. 虽然文件缓存在一定程度上能够帮你缓解, 但总会有一部分操作会最落地到底层 I/O 的. 所以实际操作中, 要尽量以 4kB 为边界操作大文件. 

## 大目录的寻址效率

有一个问题被提了出来：我们都知道, 当我们面对一个大目录（目录中有很多很多文件）的时候, 这个目录刷出来需要很长的时间. 那么我们在开发的时候是不是要避免经常在这个大目录中读写文件呢？

实际上, 当你第一次操作这个大目录的时候, 可能延时确实会比较大. 但是实测只要进入了这个目录之后, 再后续操作的时候, 却一点都不慢, 和其他的普通目录相当. 

这个问题的原因, 我个人猜测（求权威人士指正）是这样的：
　　**目录**在文件系统中, 是以一个 **inode** 的方式存在的, 那么载入目录, 实际上就是载入这个 inode. 从存储的角度, inode 也只是一个普通的文件, 那么载入 inode 的动作和载入其他文件一样, 也会经过文件缓存策略. 载入了一次之后, 只要你持续地访问它, 那么操作系统就会将这个 inode 保持在缓存中. 因此后续的操作, 就是直接读写 RAM 了, 并不会受到硬盘 I/O 瓶颈的影响. 

## 参考资料

[高性能磁盘 I/O 开发学习笔记 -- 软件手段篇](https://segmentfault.com/a/1190000011830405)
[Linux 内核的文件 Cache 管理机制介绍](https://www.ibm.com/developerworks/cn/linux/l-cache/index.html)
[磁盘I/O那些事](https://tech.meituan.com/about-desk-io.html)
[Linux系统结构 详解](http://blog.csdn.net/hguisu/article/details/6122513)
[【Linux编程基础】文件与目录--知识总结](http://blog.leanote.com/post/xiangfei/4#title-4)