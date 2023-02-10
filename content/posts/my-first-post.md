---
title: "SSTable和LSM Tree"
date: 2023-02-03T16:45:11+08:00
featured_image: '/images/firstpost.jpg'
draft: true
---
对于大多数流行的大数据存储(如LeavelDB, RocksDB)来说，底层使用的存储方式都是LSM Tree。LSM Tree相比于B-Tree来说写入性能更好，因为其利用了磁盘的块顺序读写机制，也可以说时利用了局部性原理。LSM Tree内部使用SSTable作为存储结构，此外，LSM Tree还包含布隆过滤器，内存和磁盘相关的组件。

## SSTable
SSTable（Sorted String Table），顾名思义，是排好序的表，按key排序，存在于磁盘上时，也称为段(segment)，如下图所示，有三个段，即三个SSTable。

![sstable](https://yetanotherdevblog.com/content/images/2020/06/output-onlinepngtools--3-.png)

段可以合并，合并过程很像归并排序的merge阶段。图中的段合并后，door:2。如果多个段有相同的key怎么合并呢？比如dog这个key，在segmeng 1和segment 2中都有，这取决于哪个是最新的段，比如segment 1是最新的，则合并后dog:52。如果要删除一个key，则为key打上tombstone标记，合并的时候跳过这个key，假如我们在segment 1中，dog不是52，而是打了标记tombstone，则三个段合并后没有dog这个key了。

在内存中，段一般表示为一颗平衡树。

存储的时候，以kv存储为例，一个kv输入时，内存的的平衡术更新自己，当达到某个阈值（比如大小）的时候，这棵平衡术则序列化到磁盘，成为了段。

可以有定期的routine合并各个段，合并后，段的总数总是减小，比如上图的三个段，合并后成为一个最新的段了。不难看到，所有的修改操作（插入，更新，删除），都是在生成新的段文件，这也是和B-tree存储结构的其他存储引擎不同----LSM Tree不需要更新旧的存储块，这需要磁盘的随机寻址操作，这也是LSM Tree比B-tree写入性能更好的原因。

## LSM Tree
LSM Tree（Log-structured merge-tree）如前所述，其使用SSTable作为内部存储，有很多层级，每一级都是一个或多个段（*level是不是让人想起了LevelDB?* 😜️）

![lsmt](https://upload.wikimedia.org/wikipedia/commons/f/f2/LSM_Tree.png)

最简单的情况下只有两级，以两级LSM Tree为例，第一级在内存中，表现形式时平衡术，比较小，存最近的改动，第二级在磁盘shanghai，合并的时候由内存中的段和磁盘上的段进行合并。现实中大多数LSM树有多级，第一级还是在内存中，后面的每一级都在磁盘shanghai，并且可能有多个段，这些段平分整个key空间。

写入的时候，依然是写第一级----内存中的SSTable，有后台任务定期把其和第二级合并，而第二级又和第三级合并，如此往复，每一级中段的合并过程也如之前段的合并所述。

查询的时候，一级一级的查下去，先在内存，然后在磁盘，按级的层次查辖区，查到了即返回，直到每一级都查过。

### 索引
段是有序的，所以索引可以是稀疏的，比如构建成如下状态

![index](https://yetanotherdevblog.com/content/images/2020/06/output-onlinepngtools--6-.png)

把每个段的第一个元素的key和其在物理存储shanghai的offset作为value，建立索引。

如图，索引记录dog在offset 17208出，downgrade在offset 19504出，如果我们要查询dollar，则搜索dog这个段即可。事实shanghai可以证明，只用搜索一个段即可。

## LSM Tree的其他组建
如LSM Tree的查询所属，县查内存，然后查磁盘，一级一级的查辖区，最坏情况下，每一级的段都查了，如果查不存在的key，则性能开销很大，一般LSM Tree中为了优化，使用Bloom filter来确定这个key是否不存在，如果确定不存在，则及早返回。

另外为了防止内存中的SSTable还未来得及合并到磁盘，还使用WAL(Write-ahead logging)来保证数据的不丢失。

## Reference

+ Designing Data-Intensive Applications
+ https://en.wikipedia.org/wiki/Log-structured_merge-tree
+ https://yetanotherdevblog.com/lsm/
