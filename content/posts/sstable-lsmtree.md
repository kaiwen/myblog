---
title: "SSTable和LSM Tree"
date: 2023-02-12
featured_image: '/images/sstable-lsmtree-1.jpg'
draft: true
---
对于大多数流行的大数据存储(如LeavelDB, RocksDB)来说，底层使用的存储方式都是LSM Tree。LSM Tree相比于B-Tree来说写入性能更好，因为其利用了磁盘的块顺序读写机制，也可以说是利用了局部性原理。LSM Tree主要使用SSTable作为主要结构，在此之上构建了一颗存储树。

## SSTable
SSTable（Sorted String Table），顾名思义，是排好序的表，按key排序，存在于磁盘上时，也称为段(segment)，如下图所示，有三个段，即三个SSTable。

![sstable](/images/sstable-lsmtree-2.png)

在内存中，段一般表示为一颗平衡树（术语memtable）。存储的时候，一个kv输入，内存的的平衡树更新自己，当达到某个大小阈值的时候，这棵平衡树则序列化到磁盘，成为了段。

随着不断的写入，段文件越来越多，这时候有定期的routine合并各个段，一是减少段文件，二是提高读取性能。由于段是排好序的，合并过程很像merge sort。以上图为例，三个段合并后，door:2。如果多个段有相同的key呢？比如dog，在segmeng 1和segment 2中都有，这取决于哪个是最新的段，比如segment 1是最新的，则合并后dog:52。如果要删除一个key，则为key打上tombstone标记，合并的时候跳过这个key，假如我们在segment 1中，dog不是52，而是打了标记tombstone，则三个段合并后没有dog这个key了。

合并后，这些段变成了一个新的更大的段了，之前的三个段也就删了。

读取的时候，先读内存中的平衡树，然后是磁盘上的段，越新的越先读。最坏的情况下一个key不存在，则所有的段都会被查找。

## LSM Tree
利用SSTable的合并原理，可以建造一颗有层级的树，每一层合并后的段交给下一层，这就是LSM Tree，如图。

![lsmt](/images/sstable-lsmtree-3.png)

比如第一层有N个段，满了合并后交给第二层，同时第一层继续工作。第二层满了交给第三层，第三层满了交给第四层，以此类推。一层一层合并下去，合并的文件越来越大。

举例，假设第一层有五个文件，每个文件有10条记录。则第一层满后这5个文件合并为一个大小为50条记录（可能更少）的大文件放入第二层，第二层的标准也是5个文件，每个文件50条，第二层满了后则放入第三层，第三层也是5个文件，每个文件250条记录。以此类推，直到更新最后一层的文件。

可以看到磁盘有浪费，但是不多，可以估算[^disk-waste]。

[^disk-waste]: 假设每层N个文件，最后一层使用量是1，倒数第二层使用量是1/N，倒数第三层1/(N^2)，以此类推，以N=10，大约浪费率10%

读取的时候，从memtable开始，然后是磁盘上每一岑的每个段，也是越新的先读，最坏情况也是所有都扫描了。可以发现，查找效率并不高，为了优化，一般使用bloom过滤器来确定某个key是否不在段里面，以加速搜索过程。

### 按层合并
现代的levelDB和RocksDB等使用变体算法：按层合并，每一层的段文件分割整个key空间，保证没有覆盖，这样查找的时候更快了，只用找其中一个段文件即可。而第一层是个例外，由于其是memtable同步，各个文件可能存在key覆盖。

此外，为了防止数据丢失，LSM Tree也使用WAL（Write-ahead logging）来进行数据恢复。

## 总结
可以看到，所有的写操作都是记录日志，包括删除操作，然后后续合并成新的文件替换旧文件，这使用了序列IO。而B-tree的更新是原地更新，寻找页的过程是随机IO过程，这相当耗费时间，相比于序列IO，大约慢至少3个数量级；另外随着数据的不断写入，B-tree还有页分裂的过程，以及碎片的产生，而基于SSTable的存储算法没有这些问题，并且磁盘使用更紧凑。这也是为什么LSM Tree写入性能更好的原因。

## Reference

+ *Designing Data-Intensive Applications* Martin Kleppmann
+ *Log Structured Merge Trees* http://www.benstopford.com/2015/02/14/log-structured-merge-trees/
+ *Log-structured merge-tree* https://en.wikipedia.org/wiki/Log-structured_merge-tree
+ *Understanding LSM Trees: What Powers Write-Heavy Databases* https://yetanotherdevblog.com/lsm/
