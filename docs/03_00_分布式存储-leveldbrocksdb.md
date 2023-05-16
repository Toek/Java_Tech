本篇介绍典型的基于SStable的存储。适用于与SSD一起使用。更多存储相关见：[https://segmentfault.com/a/11...](https://segmentfault.com/a/1190000018729661)。涉及到leveldb,rocksdb。基本上分布式都要单独做，重点是单机架构，数据写入，合并，ACID等功能和性能相关的。
先对性能有个直观认识：
mysql写入千条/s，读万应该没问题。redis 写入 万条/s 7M/s(k+v 700bytes，双核)读是写入的1.4倍 mem 3gb 2核。这两个网上搜的，不保证正确，就看个大概吧。
SSD上 rocksdb随机和顺序的性能差不多，写要比读性能稍好。随机读写1.7万条/s 14M/s （32核）。batch_write/read下SSD单线程会好8倍。普通write只快1.2倍。
没有再一个机器上的对比。rocksdb在用SSD和batch-write/read下的读写性能还是可以的。

## 第一章 levelDb

架构图

![clipboard.png](..\images\03_00_分布式存储-leveldb架构图.png)

### 读取过程

数据的读取是按照 MemTable、Immutable MemTable 以及不同层级的 SSTable 的顺序进行的，前两者都是在内存中，后面不同层级的 SSTable 都是以 *.ldb 文件的形式持久存储在磁盘上

### 写入过程

1.调用 MakeRoomForWrite 方法为即将进行的写入提供足够的空间；
在这个过程中，由于 memtable 中空间的不足可能会冻结当前的 memtable，发生 Minor Compaction 并创建一个新的 MemTable 对象；不可变imm去minor C,新的memtable继续接收写
在某些条件满足时，也可能发生 Major Compaction，对数据库中的 SSTable 进行压缩；
2.通过 AddRecord 方法向日志中追加一条写操作的记录；
3.再向日志成功写入记录后，我们使用 InsertInto 直接插入 memtable 中，内存格式为跳表，完成整个写操作的流程；

### writebatch并发控制

全局的sequence(memcache中记录，这里指的就是内存)。读写事务都申请writebatch,过程如下程序。
虽然是批量，但是仍然串行，是选择一个leader(cas+memory_order)将多个writebatch合并一起写入WAL，再依次写入memtable再提交。
每一批writebatch完成后才更新sequence

```cobol
加锁，获取队列信息，释放锁，此次队列加入到weitebatch中处理，写日志成功后写入mem，此时其他线程可以继续加入队列，结束后加锁，更新seq,将处理过的队列移除。



Status DBImpl::Write(const WriteOptions &options, WriteBatch *my_batch){



    Writer w(&mutex_);



    w.batch = my_batch;



    w.sync = options.sync;



    w.done = false;



 



    MutexLock l(&mutex_);



    writers_.push_back(&w);



    while (!w.done && &w != writers_.front())



    {



        w.cv.Wait();



    }



    if (w.done)



    {



        return w.status;



    }



    // May temporarily unlock and wait.



    Status status = MakeRoomForWrite(my_batch == nullptr);



    uint64_t last_sequence = versions_->LastSequence();



    Writer *last_writer = &w;



    if (status.ok() && my_batch != nullptr)



    { // nullptr batch is for compactions



        WriteBatch *updates = BuildBatchGroup(&last_writer);



        WriteBatchInternal::SetSequence(updates, last_sequence + 1);



        last_sequence += WriteBatchInternal::Count(updates);



        // Add to log and apply to memtable.  We can release the lock



        // during this phase since &w is currently responsible for logging



        // and protects against concurrent loggers and concurrent writes



        // into mem_.



        {



            mutex_.Unlock();



            status = log_->AddRecord(WriteBatchInternal::Contents(updates));



            bool sync_error = false;



            if (status.ok() && options.sync)



            {



                status = logfile_->Sync();



                if (!status.ok())



                {



                    sync_error = true;



                }



            }



            if (status.ok())



            {



                status = WriteBatchInternal::InsertInto(updates, mem_);



            }



            mutex_.Lock();



            if (sync_error)



            {



                // The state of the log file is indeterminate: the log record we



                // just added may or may not show up when the DB is re-opened.



                // So we force the DB into a mode where all future writes fail.



                RecordBackgroundError(status);



            }



        }



        if (updates == tmp_batch_)



            tmp_batch_->Clear();



 



        versions_->SetLastSequence(last_sequence);



    }



    while (true) {



        Writer* ready = writers_.front();



        writers_.pop_front();



        if (ready != &w) {



          ready->status = status;



          ready->done = true;



          ready->cv.Signal();



        }



        if (ready == last_writer) break;



  }



 



  // Notify new head of write queue



  if (!writers_.empty()) {



    writers_.front()->cv.Signal();



  }



 



  return status;



}
```

### ACID

- 版本控制
  在每次读的时候用userkey+sequence生成。保证整个读过程都读到当前版本的，在写入时，写入成功后才更新sequnce保证最新写入的sequnce+1的内存不会被旧的读取读到。
  Compaction过程中：（更多见下面元数据的version）
  被引用的version不会删除。被version引用的file也不会删除
  每次获取current versionn的内容。更新后才会更改current的位置

### memtable

频繁插入查询，没有删除。需要无写状态下的遍历（dump过程）=》跳表
默认4M

### sstable

sstable（默认7个）【上层0，下层7】

- 内存索引结构filemetadata
  refs文件被不同version引用次数,allowed_Seeks允许查找次数,number文件序号,file_size，smallest,largest
  除了level0是内存满直接落盘key范围会有重叠，下层都是经过合并的，没重叠，可以直接通过filemetadata定位在一个文件后二分查找。level0会查找多个文件。
  上层到容量后触发向下一层归并，每一层数据量是比其上一层成倍增长
- 物理
  sstable=>blocks=>entrys

![clipboard.png](..\images\03_00_分布式存储-leveldb-sstable-数据结构.png)

- data index：每个datablock最后一个key+此地址.查找先在bloom（从内存的filemetadata只能判断范围，但是稀疏存储，不知道是否有值）中判断有再从data_index中二分查找（到重启点比较）再从data_block中二分查找
- meta index:目前只有bloom->meta_Data的地址
- Meta Block：比较特殊的Block，用来存储元信息，目前LevelDB使用的仅有对布隆过滤器的存储。写入Data Block的数据会同时更新对应Meta Block中的过滤器。读取数据时也会首先经过`布隆过滤器过滤`.
- bloom过滤器：key散列到hash%过滤器容量，1代表有0代表无，判断key在容量范围内是否存在。因为hash冲突有一定存在但并不存在的错误率 [http://www.eecs.harvard.edu/~...](http://www.eecs.harvard.edu/~michaelm/postscripts/im2005b.pdf)
  哈希函数的个数k；=>`double-hashing` i从0-k, gi(x) = h1(x) + ih2(x) + i^2 mod m,
  布隆过滤器位数组的容量m;布隆过滤器插入的数据数量n; 错误率e^(-kn)/m
- datablock:
  Key := UserKey + SequenceNum + Type
  Type := kDelete or kValue
  ![clipboard.png](..\images\03_00_分布式存储-leveldb-sstable-datablock数据结构.png)
  有相同的`Prefix`的特点来减少存储数据量,减少了数据存储，但同时也引入一个风险，如果最开头的Entry数据损坏，其后的所有Entry都将无法恢复。为了降低这个风险，leveldb引入了`重启点`，每隔固定条数Entry会强制加入一个重启点，这个位置的Entry会完整的记录自己的Key，并将其shared值设置为0。同时，Block会将这些重启点的偏移量及个数记录在所有Entry后边的Tailer中。
  - filemate的物理结构Manifest

每次进行更新操作就会把更新内容写入 Manifest 文件，同时它会更新版本号。

### 合并

- Minor C 内存超过限制 单独后台线 入level0
- Major C level0的SST个数超过限制，其他层SST文件总大小/allowed_Seeks次数。单独后台线程 （文件合并后还是大是否会拆分）
  当级别L的大小超过其限制时，我们在后台线程中压缩它。压缩从级别L中拾取文件，从下一级别L + 1中选择所有重叠文件。请注意，如果level-L文件仅与level-（L + 1）文件的一部分重叠，则level-（L + 1）处的整个文件将用作压缩的输入，并在压缩后将被丢弃。除此之外：因为level-0是特殊的（其中的文件可能相互重叠），我们特别处理从0级到1级的压缩：0级压缩可能会选择多个0级文件，以防其中一些文件相互重叠。
  压缩合并拾取文件的内容以生成一系列级别（L + 1）文件。在当前输出文件达到目标文件大小（2MB）后，我们切换到生成新的级别（L + 1）文件。当当前输出文件的键范围增长到足以重叠超过十个级别（L + 2）文件时，我们也会切换到新的输出文件。最后一条规则确保稍后压缩级别（L + 1）文件不会从级别（L + 2）中获取太多数据。
  旧文件将被丢弃，新文件将添加到服务状态。
  特定级别的压缩在密钥空间中循环。更详细地说，对于每个级别L，我们记住级别L处的最后一次压缩的结束键。级别L的下一个压缩将选择在该键之后开始的第一个文件（如果存在则包围到密钥空间的开头）没有这样的文件）。

### wal

32K。内存写入完成时，直接将缓冲区fflush到磁盘
日志的类型 first full, middle,last `若发现损坏的块直接跳过直到下一个first或者full`(不需要修复).重做时日志部分内容会嵌入到另一个日志文件中

记录
keysize | key | sequnce_number | type |value_size |value
type为插入或删除。排序按照key+sequence_number作为新的key

### 其他元信息文件

记录LogNumber，Sequence，下一个SST文件编号等状态信息；
维护SST文件索引信息及层次信息，为整个LevelDB的读、写、Compaction提供数据结构支持；
记录Compaction相关信息，使得Compaction过程能在需要的时候被触发；配置大小
以版本的方式维护元信息，使得Leveldb内部或外部用户可以以快照的方式使用文件和数据。
负责元信息数据的持久化，使得整个库可以从进程重启或机器宕机中恢复到正确的状态；
versionset链表
每个version引用的file（指向filemetadata的二维指针（每层包含哪些file）），如LogNumber，Sequence，下一个SST文件编号的状态信息
![clipboard.png](..\images\03_00_分布式存储-leveldb-sstable-其他元信息文件.png)

每个version之间的差异versionedit。每次计算versionedit,落盘Manifest文件（会存version0和每次变更），用versionedit构建新的version。manifest文件会有多个，current文件记录当前manifest文件，使启动变快
Manifest文件是versionset的物理结构。中记录SST文件在不同Level的分布，单个SST文件的最大最小key，以及其他一些LevelDB需要的元信息。
每当调用LogAndApply（compact）的时候，都会将VersionEdit作为一笔记录，追加写入到MANIFEST文件。并且生成新version加入到版本链表。
MANIFEST文件和LOG文件一样，只要DB不关闭，这个文件一直在增长。
早期的版本是没有意义的，我们没必要还原所有的版本的情况，我们只需要还原还活着的版本的信息。MANIFEST只有一个机会变小，抛弃早期过时的VersionEdit，给当前的VersionSet来个快照，然后从新的起点开始累加VerisonEdit。这个机会就是重新开启DB。
LevelDB的早期，只要Open DB必然会重新生成MANIFEST，哪怕MANIFEST文件大小比较小，这会给打开DB带来较大的延迟。后面判断小的manifest继续沿用。
如果不延用老的MANIFEST文件，会生成一个空的MANIFEST文件，同时调用WriteSnapShot将当前版本情况作为起点记录到MANIFEST文件。
dB打开的恢复用MANIFEST生成所有LIVE-version和当前version

### 分布式实现

google的bigtable是chubby(分布式锁)+单机lebeldb

## 第二章 rocksdb

[https://github.com/facebook/r...](https://github.com/facebook/rocksdb/wiki/RocksDB-Basics)

### 增加功能

range
merge（就是为了add这种多个rocksdb操作）
工具解析sst
压缩算法除了level的snappy还有zlib,bzip2（同时支持多文件）
支持增量备份和全量备份
支持单进程中启动多个实例
可以有多个memtable，解决put和compact的速度差异瓶颈。数据结构：跳表（只有这个支持并发）\hash+skiplist\hash+list等结构

```cobol
这里讲了memtable并发写入的过程，利用了InlineSkipList，它是支持多读多写的，节点插入的时候会使用 每层CAS 判断节点的 next域是否发生了改变，这个 CAS 操作使用默认的memory_order_seq_cst。



http://mysql.taobao.org/monthly/2017/07/05/



源码分析



https://youjiali1995.github.io/rocksdb/inlineskiplist/
```

### 合并

通用合并（有时亦称作tiered）与leveled合并（rocksdb的默认方式）。它们的最主要区别在于频度，后者会更积极的合并小的sorted run到大的，而前者更倾向于等到两者大小相当后再合并。遵循的一个规则是“合并结果放到可能最高的level”。是否触发合并是依据设置的空间比例参数。
size amplification ratio = (size(R1) + size(R2) + ... size(Rn-1)) / size(Rn)
低写入放大（合并次数少），高读放个大（扫描文件多），高临时空间占用（合并文件多）

### 压缩算法

RocksDB典型的做法是Level 0-2不压缩，最后一层使用zlib(慢，压缩比很高)，而其它各层采用snappy

### 副本

- 备份
  相关接口：CreateNewBackup（增量），GetBackupInfo获取备份ID，VerifyBackup(ID)，恢复：BackupEngineReadOnly::RestoreDBFromBackup(备份ID,目标数据库，目标位置)。备份引擎open时会扫描所有备份耗时间，常开启或删除文件。
  步骤：

  ```cobol
  禁用文件删除
  
  
  
  获取实时文件（包括表文件，当前，选项和清单文件）。
  
  
  
  将实时文件复制到备份目录。由于表文件是不可变的并且文件名是唯一的，因此我们不会复制备份目录中已存在的表文件。例如，如果00050.sst已备份并GetLiveFiles()返回文件00050.sst，则不会将该文件复制到备份目录。但是，无论是否需要复制文件，都会计算所有文件的校验和。如果文件已经存在，则将计算的校验和与先前计算的校验和进行比较，以确保备份之间没有发生任何疯狂。如果检测到不匹配，则中止备份并将系统恢复到之前的状态BackupEngine::CreateNewBackup()叫做。需要注意的一点是，备份中止可能意味着来自备份目录中的文件或当前数据库中相应的实时文件的损坏。选项，清单和当前文件始终复制到专用目录，因为它们不是不可变的。
  
  
  
  如果flush_before_backup设置为false，我们还需要将日志文件复制到备份目录。我们GetSortedWalFiles()将所有实时文件调用并复制到备份目录。
  
  
  
  重新启用文件删除
  ```

- 复制：
  1.1checkpoint
  1.2DisableFileDeletion，然后从中检索文件列表GetLiveFiles()，复制它们，最后调用2.EnableFileDeletion()。
  RocksDB支持一个API PutLogData，应用程序可以使用该API 来为每个Put添加元数据。此元数据仅存储在事务日志中，不存储在数据文件中。PutLogData可以通过GetUpdatesSinceAPI 检索插入的元数据。
  日志文件时，它将移动到存档目录。存档目录存在的原因是因为落后的复制流可能需要从过去的日志文件中检索事务
  或者调checkpoint保存

### iter

![clipboard.png](..\images\03_00_分布式存储-rocksdb数据结构.png)

![clipboard.png](..\images\03_00_分布式存储-rocksdb-LRU.png)
第一个图中的置换LRU，CLOCK。CLOCK介于FIFO和LRU之间，首次装入主存时和随后再被访问到时，该帧的使用位设置为1。循环，每当遇到一个使用位为1的帧时，操作系统就将该位重新置为0，遇到第一个0替换。concurrent_hash_map是CAS等并发安全

更多：
SST大时顶级索引：[https://github.com/facebook/r...](https://github.com/facebook/rocksdb/wiki/Partitioned-Index-Filters)
两阶段提交：[https://github.com/facebook/r...](https://github.com/facebook/rocksdb/wiki/Two-Phase-Commit-Implementation)