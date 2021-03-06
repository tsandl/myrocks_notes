# 1.整体架构
RocksDB 整个 LSM 树的信息需要常驻内存，以让 RocksDB 快速进行 kv 查找或者进行 compaction 任务，RocksDB 会用文件把这些信息固化下来，这个文件就是 Manifest 文件。

Manifest 文件作为事务性日志文件，只要数据库有变化，Manifest都会记录。其内容 size 超过设定值后会被 VersionSet::WriteSnapShot 重写。
RocksDB 进程 Crash 后 Reboot 的过程中，会首先读取 Manifest 文件在内存中重建 LSM 树，然后根据 WAL 日志文件恢复 memtable 内容。


Memtable：KV在内存中的存储结构，有序的，可读写，默认用SkipList实现。
Immutable Memtable：与Memtable结构一样，区别是只读。
SSTable：Immutable Memtable写入到文件后的结构，有多层，只读。
WAL：写操作日志记录，用于重启后恢复内存中尚未持久化的数据。
MANIFEST：记录当前有效的SSTable。
CURRENT：记录当前有效的MANIFEST

# 2.MAINFEST功能
在RocksDB中MANIFEST保存了存储引擎（sstable）的内部的一些状态元数据，简单来说当系统重启，或者程序异常被退出之后，RocksDB需要有一种机制能够恢复到一个一致性的状态， 而这个一致性的状态就是靠MANIFEST来保证的.
MANIFEST = { CURRENT, MANIFEST-<seq-no>* } 
CURRENT = File pointer to the latest manifest log 

MANIFEST-<seq no> = Contains snapshot of RocksDB state and subsequent modifications 

----
manifest-log-file = { version, version-edit* } = { version-edit* }
version-edit = Any RocksDB state change 
version = { version-edit* } 
当MANIFEST-<seq no>超过指定大小之后，MANIFEST会刷新一个新文件,当新的文件刷新到磁盘(并且文件名更新)之后，老的文件会被删除掉.这里可以认为每一次MANIFEST的更新都代表一次snapshot.


# 3.mainfest和compaction的关系
RocksDB 整个 LSM 树的信息需要常驻内存，以让 RocksDB 快速进行 kv 查找或者进行 compaction 任务，RocksDB 会用文件把这些信息固化下来，这个文件就是 Manifest 文件。

# 4.Version
version里边保存了各个level下每个sstable的fileMetaData， fileMetaData里存放了filenumber, filesize, smallestkey, largestkey等信息

# 5.VersionEdit
versionEdit里边保存了此次compact新生成的sstable所处level和MetaData
同时保存了需要被删除的sstable，（即被compact的sstable），所处level和filenumber

# 6.VersionSet
versionSet里边维护了一个双向的环状的version链表
读操作或compact操作会增加version的引用计数，当其引用计数减少为0时，会在链表中删除。

Version挂到VersionSet中，并初始化VersionSet的manifestfilenumber， nextfilenumber，lastsequence，lognumber，prevlognumber_ 信息
