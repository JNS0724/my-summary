# lsm

适合写多读少的场景。比如日志系统。

lsm-tree的核心思想是将写入推迟转换为批量写。将数据缓存在内存，通过批量的方式顺序写入磁盘。

## lsm-tree

lsm在内存和磁盘中分别建立数据结构，内存中是AVL或红黑树这种有序结构，磁盘是B-tree这类适应磁盘io的数据结构。

内存memtable =》 磁盘SSTable

## leveldb

内存：

memtable => immutable mmtable

----------

磁盘：

level 0 : sstable sstable

level 1 : sstable sstable

level 2 : sstable sstable

memtable：存储在内存中的数据，使用skiplist实现。

immutable memtable：与memtable一样，准备落盘的数据

多层sstable：leveldb使用多个层次来存储sstable文件，这些文件分布在磁盘上，这些文件都是根据键值有序排列的，其中0级的sstable的键值可能会重叠，而level 1及以上的sstable文件不会重叠。

越靠上的数据越新，即同一个键值如果同时存在于memtable和immutable memtable中，则以memtable中的为准。

### 写入数据

1.append记录到log文件。追加写磁盘，写入快。

2.将数据写入memtable，如果遇到合并的时候，可能会阻塞。

3.删除数据只做删除标记。

### 合并流程

合并是从memtable到sstable的过程。
