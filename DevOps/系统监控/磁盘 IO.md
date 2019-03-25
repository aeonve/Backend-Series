# 磁盘 IOPS

I/O 的概念，从字义来理解就是输入输出。操作系统从上层到底层，各个层次之间均存在 I/O。比如，CPU 有 I/O，内存有 I/O, VMM 有 I/O, 底层磁盘上也有 I/O，这是广义上的 I/O。通常来讲，一个上层的 I/O 可能会产生针对磁盘的多个 I/O，也就是说，上层的 I/O 是稀疏的，下层的 I/O 是密集的。

磁盘的 I/O，顾名思义就是磁盘的输入输出。输入指的是对磁盘写入数据，输出指的是从磁盘读出数据。我们常见的磁盘类型有 ATA、SATA、FC、SCSI、SAS，如图 1 所示。这几种磁盘中，服务器常用的是 SAS 和 FC 磁盘，一些高端存储也使用 SSD 盘。每一种磁盘的性能是不一样的。

![image](https://user-images.githubusercontent.com/5803001/45203026-621db300-b2ad-11e8-93ad-1f5281466388.png)

# 磁盘的性能评价

SAN（Storage Area Network, 存储区域网络）和 NAS 存储（Network Attached Storage，网络附加存储)一般都具备 2 个评价指标：IOPS 和带宽（throughput），两个指标互相独立又相互关联。体现存储系统性能的最主要指标是 IOPS。下面，将介绍一下这两个参数的含义。

IOPS (Input/Output Per Second)即每秒的输入输出量(或读写次数)，是衡量磁盘性能的主要指标之一。IOPS 是指单位时间内系统能处理的 I/O 请求数量，I/O 请求通常为读或写数据操作请求。随机读写频繁的应用，如 OLTP(Online Transaction Processing)，IOPS 是关键衡量指标。另一个重要指标是数据吞吐量(Throughput)，指单位时间内可以成功传输的数据数量。对于大量顺序读写的应用，如 VOD(Video On Demand)，则更关注吞吐量指标。

每秒 I/O 吞吐量＝ IOPS\* 平均 I/O SIZE。从公式可以看出： I/O SIZE 越大，IOPS 越高，那么每秒 I/O 的吞吐量就越高。因此，我们会认为 IOPS 和吞吐量的数值越高越好。实际上，对于一个磁盘来讲，这两个参数均有其最大值，而且这两个参数也存在着一定的关系。

IOPS 可细分为如下几个指标：

- Toatal IOPS，混合读写和顺序随机 I/O 负载情况下的磁盘 IOPS，这个与实际 I/O 情况最为相符，大多数应用关注此指标。
- Random Read IOPS，100%随机读负载情况下的 IOPS。
- Random Write IOPS，100%随机写负载情况下的 IOPS。
- Sequential Read IOPS，100%顺序读负载情况下的 IOPS。
- Sequential Write IOPS，100%顺序写负载情况下的 IOPS。

# IOPS 计算公式

对于磁盘来说一个完整的 IO 操作是这样进行的：当控制器对磁盘发出一个 IO 操作命令的时候，磁盘的驱动臂(Actuator Arm)带读写磁头(Head)离开着陆区(Landing Zone，位于内圈没有数据的区域)，移动到要操作的初始数据块所在的磁道(Track)的正上方，这个过程被称为寻址(Seeking)，对应消耗的时间被称为寻址时间(Seek Time);但是找到对应磁道还不能马上读取数据，这时候磁头要等到磁盘盘片(Platter)旋转到初始数据块所在的扇区(Sector)落在读写磁头正上方的之后才能开始读取数据，在这个等待盘片旋转到可操作扇区的过程中消耗的时间称为旋转延时(Rotational Delay);接下来就随着盘片的旋转，磁头不断的读/写相应的数据块，直到完成这次 IO 所需要操作的全部数据，这个过程称为数据传送(Data Transfer)，对应的时间称为传送时间(Transfer Time)。完成这三个步骤之后一次 IO 操作也就完成了。

在我们看硬盘厂商的宣传单的时候我们经常能看到 3 个参数，分别是平均寻址时间、盘片旋转速度以及最大传送速度，这三个参数就可以提供给我们计算上述三个步骤的时间。

第一个寻址时间，考虑到被读写的数据可能在磁盘的任意一个磁道，既有可能在磁盘的最内圈(寻址时间最短)，也可能在磁盘的最外圈(寻址时间最长)，所以在计算中我们只考虑平均寻址时间，也就是磁盘参数中标明的那个平均寻址时间，这里就采用当前最多的 10krmp 硬盘的 5ms。

第二个旋转延时，和寻址一样，当磁头定位到磁道之后有可能正好在要读写扇区之上，这时候是不需要额外额延时就可以立刻读写到数据，但是最坏的情况确实要磁盘旋转整整一圈之后磁头才能读取到数据，所以这里我们也考虑的是平均旋转延时，对于 10krpm 的磁盘就是(60s/10k)\*(1/2) = 2ms。

第三个传送时间，磁盘参数提供我们的最大的传输速度，当然要达到这种速度是很有难度的，但是这个速度却是磁盘纯读写磁盘的速度，因此只要给定了单次 IO 的大小，我们就知道磁盘需要花费多少时间在数据传送上，这个时间就是 IO Chunk Size / Max Transfer Rate。

现在我们就可以得出这样的计算单次 IO 时间的公式。

IO Time = Seek Time + 60 sec/Rotational Speed/2 + IO Chunk Size/Transfer Rate

于是我们可以这样计算出 IOPS。

IOPS = 1/IO Time = 1/(Seek Time + 60 sec/Rotational Speed/2 + IO Chunk Size/Transfer Rate)

对于给定不同的 IO 大小我们可以得出下面的一系列的数据

4K (1/7.1 ms = 140 IOPS)
　　 5ms + (60sec/15000RPM/2) + 4K/40MB = 5 + 2 + 0.1 = 7.1
　　 8k (1/7.2 ms = 139 IOPS)
　　 5ms + (60sec/15000RPM/2) + 8K/40MB = 5 + 2 + 0.2 = 7.2
　　 16K (1/7.4 ms = 135 IOPS)
　　 5ms + (60sec/15000RPM/2) + 16K/40MB = 5 + 2 + 0.4 = 7.4
　　 32K (1/7.8 ms = 128 IOPS)
　　 5ms + (60sec/15000RPM/2) + 32K/40MB = 5 + 2 + 0.8 = 7.8
　　 64K (1/8.6 ms = 116 IOPS)
　　 5ms + (60sec/15000RPM/2) + 64K/40MB = 5 + 2 + 1.6 = 8.6

从上面的数据可以看出，当单次 IO 越小的时候，单次 IO 所耗费的时间也越少，相应的 IOPS 也就越大。

上面我们的数据都是在一个比较理想的假设下得出来的，这里的理想的情况就是磁盘要花费平均大小的寻址时间和平均的旋转延时，这个假设其实是比较符合我们实际情况中的随机读写，在随机读写中，每次 IO 操作的寻址时间和旋转延时都不能忽略不计，有了这两个时间的存在也就限制了 IOPS 的大小。现在我们考虑一种相对极端的顺序读写操作，比如说在读取一个很大的存储连续分布在磁盘的的文件，因为文件的存储的分布是连续的，磁头在完成一个读 IO 操作之后，不需要从新的寻址，也不需要旋转延时，在这种情况下我们能到一个很大的 IOPS 值，如下。

4K (1/0.1 ms = 10000 IOPS)
　　 0ms + 0ms + 4K/40MB = 0.1
　　 8k (1/0.2 ms = 5000 IOPS)
　　 0ms + 0ms + 8K/40MB = 0.2
　　 16K (1/0.4 ms = 2500 IOPS)
　　 0ms + 0ms + 16K/40MB = 0.4
　　 32K (1/0.8 ms = 1250 IOPS)
　　 0ms + 0ms + 32K/40MB = 0.8
　　 64K (1/1.6 ms = 625 IOPS)
　　 0ms + 0ms + 64K/40MB = 1.6

相比第一组数据来说差距是非常的大的，因此当我们要用 IOPS 来衡量一个 IO 系统的系能的时候我们一定要说清楚是在什么情况的 IOPS，也就是要说明读写的方式以及单次 IO 的大小，当然在实际当中，特别是在 OLTP 的系统的，随机的小 IO 的读写是最有说服力的。

另外，对于同一个磁盘（或者 LUN），随着每次 I/O 读写数据的大小不通，IOPS 的数值也不是固定不变的。例如，每次 I/O 写入或者读出的都是连续的大数据块，此时 IOPS 相对会低一些；在不频繁换道的情况下，每次写入或者读出的数据块小，相对来讲 IOPS 就会高一些。也就是说，IOPS 也取决与 I/O 块的大小，采用不同 I/O 块的大小测出的 IOPS 值是不同的。 对一个具体的 IOPS, 可以了解它当时测试的 I/O 块的尺寸。并且 IOPS 都具有极限值，表 1 列出了各种磁盘的 IOPS 极限值。

```sh
# 查看磁盘剩余空间
$ df -ah
$ df --block-size=GB/-k/-m

# 查看当前目录下的目录空间占用
$ du -h --max-depth=1 /var/ | sort
# 查看 tmp 目录的磁盘占用
$ du -sh /tmp
# 查看当前目录包含子目录的大小
$ du -sm .

# 查看目录下文件尺寸
$ ls -l --sort=size --block-size=M
```

![default](https://user-images.githubusercontent.com/5803001/39466197-45bac832-4d5a-11e8-9c90-1cbdc0762b49.png)

1 处表示系统负载，它表示当前正在等待被 cpu 调度的进程数量，这个值小于系统 vcpu 数(超线程数)的时候是比较正常的，一旦大于 vcpu 数，则说明并发运行的进程太多了，有进程迟迟得不到 cpu 时间。这种情况给用户的直观感受就是敲任何命令都卡。

2 处表示当前系统的总进程数，通常该值过大的时候就会导致 load average 过大。

3 处表示 cpu 的空闲时间，可以反应 cpu 的繁忙程度，该值较高时表示系统 cpu 处于比较清闲的状态，如果该值较低，则说明系统的 cpu 比较繁忙。需要注意的是，有些时候该值比较高，表示 cpu 比较清闲，但是 load average 依然比较高，这种情况很可能就是因为进程数太多，进程切换占用了大量的 cpu 时间，从而挤占了业务运行需要使用的 cpu 时间。

4 处表示进程 IO 等待的时间，该值较高时表示系统的瓶颈可能出现在磁盘和网络。

5 处表示系统的剩余内存，反应了系统的内存使用情况。

6 处表示单个进程的 cpu 和内存使用情况。关于 top 命令中各个指标含义的进一步描述可以参见：

```sh
$ iostat -x -d 2

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
vda               0.00     0.25    0.04    0.53     0.56     4.88    19.25     0.00    6.85    3.09    7.14   0.25   0.01
```

# 文件备份

在每月第一天备份并压缩/etc 目录的所有内容，存放在/root/bak 目录里，且文件名为如下形式 yymmdd_etc，yy 为年，mm 为月，dd 为日。

```sh
#!/bin/sh
DIRNAME=`ls /root | grep bak`
if [ -z "$DIRNAME" ] ; then
mkdir /root/bak
cd /root/bak
fi
YY=`date +%y`
MM=`date +%m`
DD=`date +%d`
BACKETC=$YY$MM$DD_etc.tar.gz
tar zcvf $BACKETC /etc
echo “fileback finished!”
```

编写任务定时器：

```sh
echo “0 0 1 * * /bin/sh /usr/bin/fileback” >; /root/etcbakcron
crontab /root/etcbakcron
或使用crontab -e 命令添加定时任务：
0 1 * * * /bin/sh /usr/bin/fileback
```
