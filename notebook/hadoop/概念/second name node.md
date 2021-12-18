原文章[Secondary Namenode - What it really do?](http://computegeeken.blogspot.com/2012/06/secondary-namenode-what-it-really-do.html) 

### NameNode

NameNode主要是用来保存HDFS的元数据信息，比如命名空间信息，块信息等。当它运行的时候，这些信息是存在内存中的。但是这些信息也可以持久化到磁盘上。

![img](http://www.processon.com/chart_image/53536bcc0cf2bb589c5e16d0.png)

上面的这张图片展示了NameNode怎么把元数据保存到磁盘上的。这里有两个不同的文件：

1. fsimage - 它是在NameNode启动时对整个文件系统的快照
2. edit logs - 它是在NameNode启动后，对文件系统的改动序列

只有在NameNode重启时，edit logs才会合并到fsimage文件中，从而得到一个文件系统的最新快照。但是在产品集群中NameNode是很少重启的，这也意味着当NameNode运行了很长时间后，edit logs文件会变得很大。在这种情况下就会出现下面一些问题：

1. edit logs文件会变的很大，怎么去管理这个文件是一个挑战。
2. NameNode的重启会花费很长时间，因为有很多改动[笔者注:在edit logs中]要合并到fsimage文件上。
3. 如果NameNode挂掉了，那我们就丢失了很多改动因为此时的fsimage文件非常旧。[笔者注: 笔者认为在这个情况下丢失的改动不会很多, 因为丢失的改动应该是还在内存中但是没有写到edit logs的这部分。]

因此为了克服这个问题，我们需要一个易于管理的机制来帮助我们减小edit logs文件的大小和得到一个最新的fsimage文件，这样也会减小在NameNode上的压力。这跟Windows的恢复点是非常像的，Windows的恢复点机制允许我们对OS进行快照，这样当系统发生问题时，我们能够回滚到最新的一次恢复点上。

现在我们明白了NameNode的功能和所面临的挑战 - 保持文件系统最新的元数据。那么，这些跟Secondary NameNode又有什么关系呢？

### Secondary NameNode

SecondaryNameNode就是来帮助解决上述问题的，它的职责是合并NameNode的edit logs到fsimage文件中。

![img](http://www.processon.com/chart_image/535371590cf2bb589c5e2391.png)

上面的图片展示了Secondary NameNode是怎么工作的。

1. 首先，它定时到NameNode去获取edit logs，并更新到fsimage上。[笔者注：Secondary NameNode自己的fsimage]
2. 一旦它有了新的fsimage文件，它将其拷贝回NameNode中。
3. NameNode在下次重启时会使用这个新的fsimage文件，从而减少重启的时间。

Secondary NameNode的整个目的是在HDFS中提供一个检查点。它只是NameNode的一个助手节点。这也是它在社区内被认为是检查点节点的原因。

现在，我们明白了Secondary NameNode所做的不过是在文件系统中设置一个检查点来帮助NameNode更好的工作。它不是要取代掉NameNode也不是NameNode的备份。所以从现在起，让我们养成一个习惯，称呼它为检查点节点吧。



## 后记

这篇文章基本上已经清楚的介绍了Secondary NameNode的工作以及为什么要这么做。最后补充一点细节，是关于NameNode是什么时候将改动写到edit logs中的？这个操作实际上是由DataNode的写操作触发的，当我们往DataNode写文件时，DataNode会跟NameNode通信，告诉NameNode什么文件的第几个block放在它那里，NameNode这个时候会将这些元数据信息写到edit logs文件中。





++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++



# Hadoop Namenode和Secondary Namenode



### **Secondarynamenode作用**

SecondaryNameNode有两个作用，一是镜像备份，二是日志与镜像的定期合并。两个过程同时进行，称为checkpoint. 镜像备份的作用:备份fsimage(fsimage是元数据发送检查点时写入文件);日志与镜像的定期合并的作用:将Namenode中edits日志和fsimage合并,防止(如果Namenode节点故障，namenode下次启动的时候，会把fsimage加载到内存中，应用edit log,edit log往往很大，导致操作往往很耗时。)

### **Secondarynamenode工作原理**

日志与镜像的定期合并总共分五步：

1. SecondaryNameNode通知NameNode准备提交edits文件，此时主节点产生edits.new
2. SecondaryNameNode通过http get方式获取NameNode的fsimage与edits文件（在SecondaryNameNode的current同级目录下可见到 temp.check-point或者previous-checkpoint目录，这些目录中存储着从namenode拷贝来的镜像文件）
3. SecondaryNameNode开始合并获取的上述两个文件，产生一个新的fsimage文件fsimage.ckpt
4. SecondaryNameNode用http post方式发送fsimage.ckpt至NameNode
5. NameNode将fsimage.ckpt与edits.new文件分别重命名为fsimage与edits，然后更新fstime，整个checkpoint过程到此结束。 在新版本的hadoop中（hadoop0.21.0）,SecondaryNameNode两个作用被两个节点替换， checkpoint node与backup node. SecondaryNameNode备份由三个参数控制fs.checkpoint.period控制周期，fs.checkpoint.size控制日志文件超过多少大小时合并， dfs.http.address表示http地址，这个参数在SecondaryNameNode为单独节点时需要设置。

### **相关配置文件**

core-site.xml：这里有2个参数可配置，但一般来说我们不做修改。fs.checkpoint.period表示多长时间记录一次hdfs的镜像。默认是1小时。fs.checkpoint.size表示一次记录多大的size，默认64M。

```xml
<property>
  <name>fs.checkpoint.period</name>
  <value>3600</value>
  <description>The number of seconds between two periodic checkpoints.</description>
</property> 
<property>
  <name>fs.checkpoint.size</name>
  <value>67108864</value>
  <description>The size of the current edit log (in bytes) that triggersa periodic checkpoint even if the fs.checkpoint.period hasn’t expired.</description>
</property>
```

镜像备份的周期时间是可以修改的，如果不想一个小时备份一次，可以改的时间短点。core-site.xml中的fs.checkpoint.period值

这也解释了下面的问题：

(1)、为什么namenode和Secondary namenode需要同样大内存

(2)、大集群中namenode和Secondary namenode需要是各自独立的两个节点。

**Checkpoint的日志信息**

2011-07-19 23:59:28,435 INFO org.apache.hadoop.hdfs.server.namenode.FSNamesystem: Number of transactions: 0 Total time for transactions(ms): 0Number of transactions batched in Syncs: 0 Number of syncs: 0 SyncTimes(ms): 02011-07-19 23:59:28,472 INFO org.apache.hadoop.hdfs.server.namenode.SecondaryNameNode: Downloaded file fsimage size 548 bytes.2011-07-19 23:59:28,473 INFO org.apache.hadoop.hdfs.server.namenode.SecondaryNameNode: Downloaded file edits size 631 bytes.2011-07-19 23:59:28,486 INFO org.apache.hadoop.hdfs.server.namenode.FSNamesystem: fsOwner=hadadm,hadgrp2011-07-19 23:59:28,486 INFO org.apache.hadoop.hdfs.server.namenode.FSNamesystem: supergroup=supergroup2011-07-19 23:59:28,486 INFO org.apache.hadoop.hdfs.server.namenode.FSNamesystem: isPermissionEnabled=true2011-07-19 23:59:28,488 INFO org.apache.hadoop.hdfs.server.common.Storage: Number of files = 62011-07-19 23:59:28,489 INFO org.apache.hadoop.hdfs.server.common.Storage: Number of files under construction = 02011-07-19 23:59:28,490 INFO org.apache.hadoop.hdfs.server.common.Storage: Edits file /home/hadadm/clusterdir/tmp/dfs/namesecondary/current/edits of size 631 edits # 6 loaded in 0 seconds.2011-07-19 23:59:28,493 INFO org.apache.hadoop.hdfs.server.common.Storage: Image file of size 831 saved in 0 seconds.2011-07-19 23:59:28,513 INFO org.apache.hadoop.hdfs.server.namenode.FSNamesystem: Number of transactions: 0 Total time for transactions(ms): 0Number of transactions batched in Syncs: 0 Number of syncs: 0 SyncTimes(ms): 02011-07-19 23:59:28,543 INFO org.apache.hadoop.hdfs.server.namenode.SecondaryNameNode: Posted URL master:50070putimage=1&port=50090&machine=10.253.74.234&token=-18:1766583108:0:1311091168000:13110875677972011-07-19 23:59:28,561 WARN org.apache.hadoop.hdfs.server.namenode.SecondaryNameNode: Checkpoint done. New Image Size: 831

 **Namenode/Secondarynamenode文件结构**

| [hadadm@slave /home/hadadm/clusterdir/tmp/dfs/namesecondary/current]$ ll总用量 24drwxr-xr-x 2 hadadm hadgrp 4096 7月 19 22:59 ./drwxr-xr-x 5 hadadm hadgrp 4096 7月 19 23:59 ../-rw-r–r– 1 hadadm hadgrp  4 7月 19 23:59 edits-rw-r–r– 1 hadadm hadgrp 548 7月 19 22:59 fsimage-rw-r–r– 1 hadadm hadgrp  8 7月 19 22:59 fstime-rw-r–r– 1 hadadm hadgrp 101 7月 19 22:59 VERSION [hadadm@slave /home/hadadm/clusterdir/tmp/dfs/namesecondary/current]$ cat VERSION#Tue Jul 19 22:59:27 CST 2011namespaceID=1766583108cTime=0storageType=NAME_NODElayoutVersion=-18推这里VERSION表示的是secondarynamenode中的fsimage版本是22:59时的;加上edits应用的日志就可以到23:59 |
| ------------------------------------------------------------ |
| [hadadm@master /home/hadadm/clusterdir/dfs/name/current]$ ls -l总用量 16-rw-r–r– 1 hadadm hadgrp  4 7月 19 23:59 edits-rw-r–r– 1 hadadm hadgrp 831 7月 19 23:59 fsimage-rw-r–r– 1 hadadm hadgrp  8 7月 19 23:59 fstime-rw-r–r– 1 hadadm hadgrp 101 7月 19 23:59 VERSION [hadadm@master /home/hadadm/clusterdir/dfs/name/current]$ cat VERSION#Tue Jul 19 23:59:28 CST 2011namespaceID=1766583108cTime=0storageType=NAME_NODElayoutVersion=-18这里VERSION表示的是namenode中的fsimage版本是23:59时的; edits应用没有变更这里的fsimage相当于secondarynamenode里面的fsimage+edits |
| [hadadm@slave /home/hadadm/clusterdir/tmp/dfs/namesecondary]$ ls -l总用量 12drwxr-xr-x 2 hadadm hadgrp 4096 7月 19 23:59 currentdrwxr-xr-x 2 hadadm hadgrp 4096 7月 19 22:59 image-rw-r–r– 1 hadadm hadgrp  0 7月 19 23:59 in_use.lockdrwxr-xr-x 2 hadadm hadgrp 4096 7月 19 22:59 previous.checkpoint [hadadm@slavea /home/hadadm/clusterdir/tmp/dfs/namesecondary]$ ls -l previous.checkpoint/总用量 16-rw-r–r– 1 hadadm hadgrp  4 7月 19 23:59 edits-rw-r–r– 1 hadadm hadgrp 548 7月 19 22:59 fsimage-rw-r–r– 1 hadadm hadgrp  8 7月 19 22:59 fstime-rw-r–r– 1 hadadm hadgrp 101 7月 19 22:59 VERSION这里上一个检查点的数据是可以用来恢复数据的 |

**Import Checkpoint（恢复数据）**

如果主节点namenode挂掉了，硬盘数据需要时间恢复或者不能恢复了，现在又想立刻恢复HDFS，这个时候就可以import checkpoint。步骤如下：

1. 准备原来机器一样的机器，包括配置和文件
2. 创建一个空的文件夹，该文件夹就是配置文件中dfs.name.dir所指向的文件夹。
3. 拷贝你的secondary NameNode checkpoint出来的文件，到某个文件夹，该文件夹为fs.checkpoint.dir指向的文件夹（例如：/home/hadadm/clusterdir/tmp/dfs/namesecondary）
4. 执行命令bin/hadoop namenode –importCheckpoint
5. 这样NameNode会读取checkpoint文件，保存到dfs.name.dir。但是如果你的dfs.name.dir包含合法的 fsimage，是会执行失败的。因为NameNode会检查fs.checkpoint.dir目录下镜像的一致性，但是不会去改动它。

一般建议给maste配置多台机器，让namesecondary与namenode不在同一台机器上值得推荐的是，你要注意备份你的dfs.name.dir和 ${hadoop.tmp.dir}/dfs/namesecondary。

**后续版本中的backupnode**

Checkpoint Node 和 Backup Node在后续版本中hadoop-0.21.0，还提供了另外的方法来做checkpoint：Checkpoint Node 和 Backup Node。则两种方式要比secondary NameNode好很多。所以 The Secondary NameNode has been deprecated. Instead, consider using the Checkpoint Node or Backup Node. Checkpoint Node像是secondary NameNode的改进替代版，Backup Node提供更大的便利，这里就不再介绍了。

BackupNode ： 备份结点。这个结点的模式有点像 mysql 中的主从结点复制功能， NN 可以实时的将日志传送给 BN ，而 SNN 是每隔一段时间去 NN 下载 fsimage 和 edits 文件，而 BN 是实时的得到操作日志，然后将操作合并到 fsimage 里。在 NN 里提供了二个日志流接口： EditLogOutputStream 和 EditLogInputStream 。即当 NN 有日志时，不仅会写一份到本地 edits 的日志文件，同时会向 BN 的网络流中写一份，当流缓冲达到阀值时，将会写入到 BN 结点上， BN 收到后就会进行合并操作，这样来完成低延迟的日志复制功能。总结：当前的备份结点都是冷备份，所以还需要实现热备份，使得 NN 挂了后，从结点自动的升为主结点来提供服务。主 NN 的效率问题： NN 的文件过多导致内存消耗问题， NN 中文件锁问题， NN 的启动时间。

因为Secondarynamenaode不是实施备份和同步,所以SNN会丢掉当前namenode的edit log数据,应该来说backupnode可以解决这个问题