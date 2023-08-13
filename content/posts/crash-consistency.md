---
title: "FS Crash Consistency: 文件系统崩溃一致性"
date: 2023-08-13
featured_image: '/images/crash-consistency-1.jpg'
draft: false
---

我们熟知linux系统有inode和data block，当写入一个文件的时候不仅要更新data block，还要更新inode，由于是两个操作，如果第一个操作完执行第二个操作的时候系统崩溃（或者直接断电）了怎么办呢？重启后如何保证数据的一致性？

这个问题伴随文件系统诞生之初就有了，以前使用fsck，目前主流使用journaling（也称WAL日志），另外也有一些其他解决方案。下面一一讨论。

## 写入问题
考虑一次更新，我们写入数据，需要更新inode信息（下图中I），inode bitmap（下图中B），还有真正的数据data block（下图中Db），我们在写入过程的任何时候都可能崩溃。

![pic](/images/crash-consistency-2.png)

## fsck
fsck对崩溃没有任何处理，在重启后，文件系统mount前，进行一致性检查并尝试修复。这个检查主要检查inode，bitmap，superblock等，但它不能修复所有问题。并且还有两个缺点：

1. 太慢了，在文件系统挂在之前，扫描整个文件系统，随着现代磁盘的增大，这个操作相当慢。
2. 效率较低，如果仅仅为了修复一处不一致而扫描整个文件系统，效率较低。所以fsck并不是文件系统崩溃一致性的首选。

## journaling/WAL
日志系统在**etx3**引入，日志系统(**journaling**)也称为(**WAL**, Write Ahead Log)，简单的说就是在执行写入之前，先记录下自己将要做什么，然后如果写入阶段崩溃了，重启的时候可以根据之前的日志做重放，重新写入数据。

如下是一个日志格式例子

![pic](/images/crash-consistency-3.png)

图中TxB(Transcation Begin)是日志开始标志，I是inode数据，B是Bitmap数据，D是真实数据，TxE(Transcation End)是日志结束标志。一旦这条transcation日志持久化，我们就可以毫无负担的执行真正的数据写入，这个写入过程也称为**chekpointing**。

### 写入日志时崩溃
这里引出一个问题，如果写入日志的时候崩溃，我们又陷入了先有鸡还是先有蛋的问题？那么如何保证写日志不受崩溃影响呢？如果我们一次提交上图五个写入，并且利用barrier**等待他们完成**，然后再执行checkpointing，由于任何时候都可能崩溃

1. 假设在checkpointing之前崩溃了，并且重启后发现没有TxE，我们知道这条写入事务无效，Data也肯定没有写入，系统是一致的
2. 假设在checkpointing之前崩溃了，重启后发现有TxE，但是由于数据写入顺序无保证（磁盘对于块写入无顺序保证），见下图，我们不知道中间的数据是否真的先于TxE写入，于是无法处理
3. 假设在checkpointing阶段崩溃了，重启后发现有TxE，但是也无法和情况2区分开

![pic](/images/crash-consistency-4.png)

为了保证我们在看到TxE的时候前面的数据真的已经写入了，我们将一次性提交分为两段提交，第一段提前面的4个，第二段提TxE。第一段提后等待完成，完成后提TxE。这里不得不提下磁盘保证任意512bytes写入是原子的。保证TxE原子写入。

执行过程：

1. 写日志（前面4段数据），等待完成
2. 提交事务（写TxE）,等待完成
3. 执行checkpointing

假设崩溃：

1. 假设在写日志时候崩溃，重启后没有TxE，知道日志并不完整，数据也未写入，文件系统依然保持一致性
2. 假设在提交事务的时候崩溃，重启后没有TxE，日志不完整，同1
3. 假设在执行checkpoingting阶段崩溃，重启后有TxE，于是对日志进行replay

### 一些变体
除了以上介绍的日志崩溃一致性，还有一些变体：

**元数据日志(metadata journaling)** 此种日志不记录D，因为D在整个写入过程中，除了日志阶段，还在checkpointing阶段写数据，总共写了两次，于是日志只记录除了D之外的数据，而D直接写data block。元数据日志被NTFS和xfs使用, ext3则提供了选项，可以选元数据日志或者经典的数据日志[^datalog]。

**ext4的一次性提交** 相比于ext3的两次提交，ext4只执行一次提交——即全部提交。其在TxB和TxE中带上了数据的校验和，崩溃恢复阶段执行校验和对比，计算出来的校验和与TxB和TxE携带上的校验和对比，一致则表示日志完整，可以执行replay；不一致，则表示此条日志数据损坏。

## Reference

+ *Operating Systems: Three Easy Pieces* https://pages.cs.wisc.edu/~remzi/OSTEP/

[^datalog]: 带数据D的日志称为**data log**, 不带数据D的日志称为**metadata log**，也可以叫**ordered log**
