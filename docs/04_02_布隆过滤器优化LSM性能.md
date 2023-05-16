LSM-Tree（Log Structured Merge Tree）是数据库领域内较高效的key-value存储结构，被广泛应用于工业界数据库系统，如经典的单机kv数据库LevelDB、RocksDB，以及被诸多分布式NewSQL作为底层存储引擎。

本期将由腾讯云数据库高级工程师韩硕来为大家分享基于LSM-Tree存储的数据库性能改进，重点介绍近年来学术界对LSM-Tree的性能改进工作，并探讨这些改进措施在工业界数据库产品中的应用情况以及落地的可能性。以下是分享实录：



LSM-Tree基本结构
LSM-Tree全称为“Log Structured Merge Tree”，是一种基于磁盘存储的数据结构。1996年Patrick O’Neil等人在信息系统期刊上发表了一篇题名为“Log Structured Merge Tree”的论文，首次提出LSM-Tree结构。相比于传统的B+树，LSM-Tree具有更好的写性能，可以将离散的随机写请求转换成批量的顺序写操作，无论是在RAM、HDD还是在SSD中，LSM-Tree的写性能都更加优秀。      

![图片](..\images\LSM_Tree_Concept_Apps.png) 

作为高效的key-value存储结构，LSM-Tree已被广泛应用到工业界数据库系统中，如经典的单机kv数据库LevelDB、RocksDB，以及被诸多分布式NewSQL作为底层存储引擎，近日发布的TDSQL新敏态引擎存储模块也运用了LSM-Tree结构。LSM-Tree结构有较多优点：写性能强大、空间利用率高、较高的可调参特性、并发控制简单、有完备的修复方案等。       

![图片](..\images\LSM-Tree-Advantages.png)

LSM-Tree的本质是基于out-place update即不在原地更新。下图展示了in-place update即原地更新与out-place update即不在原地更新的区别。



在in-place update中，数据更新操作是将数据的新值直接覆盖旧值，但会带来随机读写问题。在out-place update中，数据更新操作则是将新值按时间顺序追加到文件末尾，通常会带有版本号用于标记新值或旧值，一般为顺序写，因此写性能更好，但缺点是会产生冗余数据，在读性能和空间利用率上也无法与in-place update相比。       

![图片](..\images\HIstory_Of_LSM-Tree.png)


当前主流的LSM-Tree基本结构如下图，分为内存和磁盘两部分。内存采用MemTable数据结构，可以通过B+树或跳表实现。MemTable用于接受最新的数据更新操作，维护内部数据in-tree逻辑上的有序。磁盘中的部分数据存储物理有序。数据按照层进行堆放，层次越往下，数据写入的时间就越早。每层内部按是否有序可划分为一个或多个sorted run。一个sorted run内部的数据必定有序（通常指物理上有序）。sorted run数据可进一步划分为不同的SSTable。当MemTable中的数据写满时，其会向L0进行落盘操作即Flush操作。如果LSM-Tree的某一层也达到容量阈值，就会向下合并，将同样的key的过期版本清除，即compaction操作。       

![图片](..\images\Today_LSM-Tree.png)

RocksDB是基于LSM-Tree的存储引擎，其基本结构如下图。它与LSM-Tree基本结构在主体上保持一致，不同点在于RocksDB的内存中增加了Immutable MemTable部分，这是用于Flush操作的写缓存机制。当MemTable中的数据写满时，会先暂存到Immutable MemTable中，再通过后台的异步线程将Immutable MemTable慢慢落盘到磁盘中的SST基本件。RocksDB中还增加了WAL日志，用于crash时的数据恢复。       

![图片](..\images\LSM_in_RocksDB-1.png)

LSM-Tree支持的查询可分为点查与范围查询两大类，对应的执行方式如下：

点查：先查MemTable，再从SST中的Level0、Level1…...一层层向下探查，找到数据就返回。由于上层数据必定比下层数据的版本新，因此返回的都是最新数据。

范围查询：每一层都会找到一个匹配数据项的范围，再将该范围进行多路归并，归并过程中同一key只会保留最新版本。

![图片](..\images\LSM_Point_Range_Lookup.png)

LSM-Tree性能的衡量主要考虑三类因素：空间放大、读放大和写放大。
第一类因素是空间放大。在LSM-Tree中所有写操作都是顺序追加写，数据的更新操作则是通过创建一个新的空间来存储新值，即out-place update。与此同时，因为旧值不会立即被删除，因此会占用部分空间。实际上这部分冗余数据占用空间的大小要远大于有效数据本身，这种现象被称为“空间放大”。LSM-Tree会利用compaction操作来清理旧数据从而降低空间放大。
第二类因素是读放大。在LSM-Tree、B+树等外存索引结构中，进行读操作时需要按从上到下的顺序一层层去读取存储节点，如果想读一条数据，就会触发多次操作，即一次读操作所读到的数据量实际上要远大于读目标数据本身，从而影响读性能，这种现象被称为“读放大”。    第三类因素是写放大。在LSM-Tree中，compaction操作会将多个SST文件反复读取，合并为新的SSTable文件后再次写入磁盘，因此导致一条kv数据的多次反复写入操作，由此带来的IO性能损失即写放大。       

![图片](..\images\LSM_Space_Read_Write_Amplification.png)

LSM-Tree的初衷是想通过空间放大和读放大来换取写放大的降低，从而达到极致的写性能，但也需要做好三方面因素的权衡。EDBT 2016的一篇论文首先提出RUM猜想（R为read，U为update，M为memory usage，RUM为三者缩写）。该论文认为，三者之间存在权衡关系，无法使得三个方面的性能都达到最优，因此必须在三者之间进行有效权衡。       

![图片](..\images\LSM_Space_Read_Write_Amplifications-1.png)


以compaction操作为例，其目的是保证数据的局部有序和清理数据旧值，即一个sorted run内部的多个SST文件中的数据是有序的，从而降低读放大。对一个查询而言，在一个sorted run中至多只需要读取一个SST中的一个数据块。

如果完全不做compaction操作，即一直顺序写，LSM-Tree就会退化为log文件，这时写性能达到最佳。因为只需要顺序写log即可，不需要做任何操作。但读性能将会处于最差状态，因为在没有任何索引、无法保证有序性的情况下，每次想读到固定的数据项，就需要扫描所有的SST件。

如果compaction操作做到极致，实现所有数据全局有序，此时读性能最优。查询时只需要通过索引和二分法即可迅速找到要读的键值的位置，一次IO操作即可完成，但代价是需要频繁进行compaction操作来维持全局有序状态，从而造成严重的写放大，即写性能变差。
这就延伸出两种compaction策略：

Tiering compaction：较少做compaction操作，有序性较弱，每一层允许有多个sorted run。

Leveling compaction：更频繁的compaction操作，尽可能增强有序性，限制每一层最多只有1个sorted run（L0层除外）。

![图片](..\images\LSM_Space_Read_Write_Amplifications-2.png)

Compaction优化策略与分析
Leveling compaction策略中，每层只有一个sorted run，sorted run内部的数据保持物理有序。具体实现上我们以RocksDB为例。一个sorted run可以由多个key不重叠且有序的SSTable files组成。当第L层满时，L层会选取部分数据即部分SSTable，与L+1层有重叠的SSTable进行合并，该合并操作即compaction操作。   

![图片](..\images\LSM_Compact_Strategy_twoBasic.png)    

Tiering compaction策略中，每层可以有至多T个sorted run，sorted run内部有序但每层不完全有序。当第L层满时，L层的T个sorted run会合并为L+1层的1个sorted run。因为每层允许有多个sorted run，因此SST文件间可能会存在数据范围的重叠，compaction操作的频率会更低，写性能也会更强。       

![图片](..\images\LSM_Compact_Strategy_twoBasic-1.png)


两种compaction策略各有优劣。Tiering compaction因为compation操作频率低，过期版本的数据未能得到及时清除，因此空间利用率低，由此带来的查询操作的代价比较高。在极端情况log file即完全不做compaction操作时，写入性能最优。Leveling compaction则会更频繁地做compaction操作，因此数据趋向更有序。极端情况sorted array即数据达到全局有序时，此时查询性能和空间利用率最优。       

![图片](..\images\LSM_Compact_Strategy_twoBasic-2.png)

SIGMOD 2018的一篇论文对上述两种compaction策略进行了详细的理论分析，分析结果归纳为下表，其分析结果与前述一致。理论分析过程如下：      

![图片](..\images\LSM_Compact_Strategy_twoBasic-3.png) 

先从写入操作复杂度进行分析。I/O以block为基本单位，一次I/O平均读写B条kv，因此一条kv写一次磁盘的代价可以表示为1/B。在leveling compaction策略中，第i层向第i+1层最多合并T次，在第j层合并到第i层时，其可被后续的数据项反复进行t-j次合并。由此可得，每条KV在第i层即任意一层被平均IO的读写次数为t/2次，即O(T)。从MemTable一直向下合并到最后一层即第L层，会得到平均写磁盘数即“TL/B”。

![图片](..\images\LSM_Compact_Strategy_twoBasic-4.png)       


以下图为例，每合并一次，第i层的增量不会超过1/T。第i层通道从空到满，其最多向下合并t次，因此每条kv在每层平均被合并2/T次。              

![图片](..\images\LSM_Compact_Strategy_twoBasic-5.png)

在Tiering compaction策略中，每条KV在第i层时只会被合并一次，因此从MemTable一直向下合并到最后一层即第L层，会得到平均写磁盘数即L/B。       

![图片](..\images\LSM_Compact_Strategy_twoBasic-6.png)

以下图为例，在第i-1层向第i层合并时，会在第i层产生一个新的sorted run，平均每条kv在每层只会被合并一次。

![图片](..\images\LSM_Compact_Strategy_twoBasic-7.png)       

再从空间放大率进行分析。space-amplification定义为：过期版本的kv数与有效kv总数之比。在Leveling compaction策略中，最坏的情况为前L-1层的每条数据在最后一层即第L层中都各有一条未被清除的过期版本数据。每层容量为比值为T的等比数列，根据等比数列求和公式， 第L−1层的kv数量占总kv数的1/T，最后一层则占 (T−1)/T，因此空间放大为1/T，空间放大率较低。    

![图片](..\images\LSM_Compact_Strategy_twoBasic-8.png)   

在Tiering compaction策略中，允许每层最多有T个sorted run，最坏情况下最后一层的T个sorted run之间都是每个kv的不同版本，即一条kv在最后一层的每个sorted run都出现一次，共T个不同版本，因此空间放大为O(T)。   

![图片](..\images\LSM_Compact_Strategy_twoBasic-9.png)  

下图将两者进行对比。Leveling compaction的空间放大系数为1/T，因此空间利用率较高。Tiering compaction在最坏情况下，最后一层的每个sorted run都是冗余数据，空间利用率较差。      

![图片](..\images\LSM_Compact_Strategy_twoBasic-10.png) 


再从查询角度进行分析。点查使用Bloom filter进行过滤，将其分成两类：

读穿透，即查询数据不存在。

假设知道每次发生假阳性的概率，如果判定结果为假阳性，则需要读一次磁盘。


在Leveling compaction策略中，每层只有一个sorted run，实现完全有序，因此每层只需要查询一次Bloom过滤器。根据二项分布期望公式，推导出的期望读盘次数为L×e的-m/n次方。       

![img](..\images\LSM_Compact_Strategy_twoBasic-11.png)

在Tiering compaction策略中，每层最多有T个sorted run，最多可能查询T次Bloom filter，在前述结构基础上乘以T的系数，根据推导出的期望读磁盘次数，其查询性能落后于Leveling compaction。

![img](..\images\LSM_Compact_Strategy_twoBasic-12.png)

如果查询数据存在，因为发生假阳性的概率远小于1，因此大部分情况下，无论是Tiering compaction还是Leveling compaction，都只需要读一次盘。      

![img](..\images\LSM_Compact_Strategy_twoBasic-13.png)

再从范围查询角度进行分析。该论文将范围查询分为短范围查询和长范围查询。短范围查询指返回block数量的大小不超过最大层数范围的两倍；反之则为长范围查询。  

![img](..\images\LSM_Compact_Strategy_twoBasic-14.png)


通过统计公式发现，短范围查询在归并前的各版本数据趋向于均匀分布在各层，因此Leveling compaction和Tiering compaction的查询复杂度分别为O(L)和O(T×L)。长范围查询在归并前各版本数据大部分来自最后一层，因此上述两种策略的代价分别为返回block数的大小、返回block数的大小再乘以T，因此Leveling  compaction比Tiering  compaction的读性能更好。      

![img](..\images\LSM_Compact_Strategy_twoBasic-15.png)


通过上述理论分析，可以得到如下结论：

Tiering compaction的写放大低，compaction频率低，其缺陷为空间放大高、查询效率低，更利于update频繁的workload；Leveling compaction的写放大高，compaction操作更频繁，但空间放大低，查询效率高。

尽管Tiering  compaction和Leveling  compaction的空间放大不同，但导致空间放大的主要原因相同，即受最下层的过期版本数据影响。

越往下的层，做一次compaction的I/O代价越高，但发生的频率也更低，不同层之间做compaction的期望代价大致相同。

点查、空间放大、长范围查询的性能瓶颈在LST-tree的最下层，而更新操作则更加均匀地分布在每一层。因此，减少非最后一层的compaction频率可以有效降低更新操作的代价，且对点查、空间放大、长范围查询的性能影响较小。



降低写放大

基于上述理论分析，该论文提出混合compaction策略即Lazy Leveling。它将Leveling与Tiering进行结合，在最后一层使用Leveling策略，其他层使用Tiering策略，即最后一层只能存在唯一的sorted run，其他层允许存在多个sorted run，从而有效降低非最后一层做compaction的频率。

下表是采取Lazy Leveling策略后的性能汇总表，其中，绿色部分表示效果较好，红色部分表示较差，黄色部分代表适中。从下表可以看出，Lazy Leveling的更新操作（update）性能优于Leveling，接近于Tiering。这是由于在前L-1层维持Tiering策略，做compaction的频率更低，写放大低。但Lazy Leveling的空间放大接近于Leveling，远好于Tiering。这相当于结合了两种策略的优势。

对于点查（point lookup），论文中分别分析了查找不存在kv和kv在最后一层两种情况，并基于论文Monkey的思路对每层的bloom filter bit进行了优化，可以达到与Leveling+Monkey策略相匹配的点查性能。对于长范围查询，Lazy Leveling可以做到与Leveling一致的性能，而短范围查询则退化至接近Tiering的水平。

论文对此进行总结：使用一种单一compaction策略，不可能在上述所有操作中做到性能最优。Lazy Leveling本质上是Tiering与Leveling策略的折衷加调优，在侧重于更新操作、点查和长范围查询的workload上性能较好；Leveling适用于查询为主的workload；Tiering则适用于更新操作为主的workload。  

![img](..\images\LSM_Compact_Strategy_twoBasic-降低写放大.png)  


论文通过建立代价模型将Lazy Leveling进行推广，将其称为Fluid LSM-Tree。假设有两个参数，分别为Z和K。Z表示最后一层允许有最多Z个sorted run；其他层则允许有多个sorted run。K为限制参数，表示每层最多有K个sorted run。当每个sorted run的值达到t/k时会向下做compaction。通过这样的方式，将Lazy Leveling进行泛化推广，通过不同的workloads来调节参数Z和K。下表是将参数Z和K添加进去后的Fluid LSM-Tree的代价利用分析结果。

但这在实际操作中会遇到问题，即如何根据workloads来选取参数Z和K。论文采取搜索方式去找到针对某个workload代价模型最小的参数K和Z。在真正实现时，我们对workload的认知相对有限，想要找到最优的参数K和Z会较为困难，且该模型总体比较复杂，实践性有限。

![img](..\images\LSM_Compact_Strategy_twoBasic-降低空间放大.png)

RocksDB在某种意义上也采取了混合compaction策略。RocksDB的L0层的SST之间，K允许互相重叠，即允许多个sorted run。这实际上是Tiering策略，因为其落盘的Flush的性能更优。在L1-Ln层中采取Leveling策略，能够更及时地清理过期数据，提高空间利用率。  

![img](..\images\LSM_Compact_Strategy_twoBasic-混合.png)

RocksDB将每个sorted run切割为多个SST文件，每个SST文件都有大小阈值。其优点在于每次发生compaction时，只需要选取一个或者少量的SSTable与下层有KV重叠的SSTable进行合并，涉及到合并的有效数据量处于可控范围。 
![img](..\images\LSM_Compact_Strategy_twoBasic-sstFiles.png)

 SOSP 2017中有一篇名为PebblesDB的论文，提出了compaction Guard概念。每层分割为多个分片，分片间的分割称之为compaction Guard，即一次compaction操作不会跨越该分界线，每个分界线内部SST之间允许重叠，可理解为Tiering compaction。这种做法最大的好处是可以保证数据项从第i层向第L+1层进行compaction时，写入只有一次。因为它将原生的LSM-Tree进行分隔，形成FLSM结构。其坏处是读性能会变差。因为找到对应分片后，分片内部如果存在多个SST，我们就不知道数据的真正存放位置，这时需要借助Bloom过滤器来对每个SST进行探查，且即使使用Bloom过滤器，其发生假阳性的期望次数也会增加。   

![img](..\images\LSM_Compact_Strategy_twoBasic-Fragment.png)

在RocksDB实践过程中，我们发现它实际上提供了一个SST Partitioner的类，通过类可以在代码实现上保证分片，但与PebblesDB不同的是，其在分片内部保持SST不重叠，仍然采取Leveling策略。其好处是读性能不变。虽然写性能没有得到提升，但更加适配于扩容场景，便于迁移及迁移后数据的物理删除操作。 

![img](..\images\LSM_Compact_Strategy_twoBasic-SST_Partitioner.png)      


以TDSQL新敏态引擎架构为例。每层的存储节点称为TDstore，一个TDstore真实的数据存储相当于维护一个LSM-Tree结构来存储KV数据，再将KV数据按照区间划分成不同Region。需要扩容时，我们会增加若干个存储节点，再将原来节点上的部分Region迁移过去。Region迁移涉及到Region数据的迁移。如果采用前述的分割策略，将LSM-Tree的每一层基于Region边界进行分割，将Region从相对完整的SST文件中捞取出来，并插入到新增的TDstore存储节点中。因为SST根据边界进行分割，我们可以相对完整地将Region内部的数据迁移或删除，因此Region数据迁移的性能会得到提升。       


​        ![img](..\images\LSM_Compact_Strategy_twoBasic-数据分片迁移.png)

降低读放大

降低读放大必须结合布隆过滤器。具体实现为：每层设置一个布隆过滤器，通过布隆过滤器进行过滤，减少无效读磁盘block的次数。

下图为前述的结论表。当数据查询不存在即发生读穿透时，发生假阳性的概率为e的-m/n次方。只要发生假阳性，就必须去读数据块，进行一次读磁盘操作。在Leveling中，每层只有一个sorted run，查一次Bloom filter的概率为e的-m/n次方，根据期望公式推导出的期望读盘次数为L乘以e的-m/n次方。Tiering中则是T乘以L再乘以e的-m/n次方。假设点查时使用布隆过滤器进行过滤，每层都建立一个布隆过滤器即固定的bits团，读性能就可得到提升。       

![img](..\images\LSM_Compact_Strategy_twoBasic-降低读放大-Bloom_filter.png)

2017年一篇名为Monkey的论文对布隆过滤器进行优化。在LSM-Tree不同层设置不同的布隆过滤器的bits，可以有效降低IO开销，从而减少读穿透的期望次数。其结论分析如下：第i层设置（a+b）×（L-i）的bits。由于最后一层数据量最大，只用一个bit来hash到布隆过滤器中。相比于前述公式，这可以减少一个L系数。因为Bloom filter要在内存中持有，而每层的bits不同，因此总体上降低了布隆过滤器的内存开销。其点查的读性能和整体Bloom filter的空间利用率都会得到提升。

![img](..\images\LSM_Compact_Strategy_twoBasic-降低读放大.png)


SIGMOD 2021一篇名为Cuckoo的论文，采用布谷鸟过滤器代替布隆过滤器，从而优化读性能。布隆过滤器的优化方式为：LSM-Tree的每层甚至每个SST文件都会维护一个Bloom filter，查询时需要从MemTable L0到Ln一层层向下探查，每次探查时先走一遍相应的布隆过滤器。该论文提出替代方案，不需要每层都维护一个布隆过滤器，而是选择维护一个全局的布谷鸟过滤器。先查全局的布谷鸟过滤器，如果反馈不存在则表示数据不存在，如果反馈存在，考虑到可能发生假阳性，我们再去查数据文件。

实现原理为：全局的布谷鸟过滤器里记录了Level ID。如果在布谷鸟过滤器中有mash，则不需要继续向下探查，可以直接找到其对应的Level ID，找到对应层、对应的sorted run去读磁盘，看数据是否存在。布谷鸟过滤器对接两部分：其hash map或签名，再加上位置ID即level ID。

![图片](..\images\LSM_Compact_Strategy_twoBasic-降低读放大1.png)       


布谷鸟过滤器的工作原理如下图，相当于维护下图右上角中的两个桶。我们通过两次哈希算出key所要插入的具体位置，所对应的两个桶称为b1和b2。b1直接拿key进行擦洗，b2不需要用key的原值，在算出签名指纹后再做哈希，因此每个key会有两个可以存放的位置。

布谷鸟过滤器哈希的数据插入过程如下：假设要插入item X，通过第一个哈希计算出其位置是2，即b1等于2，第二个哈希算出其位置是6。我们先将其放在第一个位置，如果可以放下则放下，如果放不下再看第二个位置即b2，如果b2没有被占则直接放下。如果b2也被占则进行relocate，将原来的对象剔除，新插入的值再放到第二个位置。被剔除的值会放到它的两个bucket中的另一个。如果另一个bucket也被占据，被占的元素会继续被踢走，直到所有元素都有一个新的空白位置存放，则该过程结束。

Cuckoo论文中还需要记录其签名，用于确认key。如果签名冲突则会造成假阳性。除此之外，还需要记录其Level ID如下图中的1、4、5，Level ID的作用是记录该值位于哪个sorted run。通过Level ID可以在具体的sorted run里查询或更新。       


​        ![图片](..\images\LSM_Compact_Strategy_twoBasic-降低读放大2.png)

降低空间放大

RocksDB默认采用Leveling compaction，但也提供类似Tiering compaction的选项。如果选择Leveling compaction，它会提供dynamic level size adaptation机制。理论分析表明，在最后一层数据充满时，Leveling compaction的空间放大率不高，等于1/T。但在实践中，最后一层的实际数据通常不会达到阈值。

假设每层容量放大比例为T，每层容量是T系数的等比数列。根据等比数列进行求和，我们发现最后一层的数据量占绝大多数，等于总大小的（T-1）/T，前面所有层只占总大小的1/T，此时冗余数据量为前面所有层数据量与最后一层数据量之比，等于1/（T-1）。假设放大比率为10，阈值为1G，L1到L4的阈值分别为10G、100G、1000G。如果L4为最后一层，通过计算可知冗余率为1.1。

实际情况下，最后一层的数据量一般不会达到阈值。在上述例子中的L4层，如果以静态阈值为参照，当前L4的数据并没有写很多，可能只有200G数据，其放大率会比较差，前面存在大量冗余数据。       

![图片](..\images\LSM_Compact_Strategy_twoBasic-降低空间放大-1.png)

因此RocksDB提出了动态调整每层容量阈值的机制。以下图为例，当L0充满时，其数据先不往L1写，而是先往L5进行合并，这时L1至L4为空。       

![图片](..\images\LSM_Compact_Strategy_twoBasic-降低空间放大-2.png)

当L5的实际数据量达到阈值10M时，compaction L0往下落的目标层切到L4，并让L4的阈值保持最初的阈值即10M，再将L5的阈值乘以放大系数T，L4与L5之间的容量阈值仍保持T倍的关系。假设L5的实际数据量达到11M，则L4的阈值为1.1M。       

![图片](..\images\LSM_Compact_Strategy_twoBasic-降低空间放大-3.png)


数据不断往L4写，L4再向L5进行compaction，L4会逐渐达到阈值。       

​         ![图片](..\images\LSM_Compact_Strategy_twoBasic-降低读放大-4.png)
L4达到阈值后，以此类推，当L3达到10M后，开启L2作为L0的合并目标层。L3初始阈值为10M，这时L4扩大为100M，L5扩充为1000M。       

![图片](..\images\LSM_Compact_Strategy_twoBasic-降低读放大-5.png)

在这种动态调整每层空间大小阈值的策略下，可以将数据冗余量始终保持在1/（T-1）。在T=10时，其空间放大率不会超过1.1，这是比较理想的空间放大率。       

![图片](..\images\LSM_Compact_Strategy_twoBasic-降低空间放大-6.png)


降低空间放大还有其他的实践措施。目前RocksDB已经实现key prefix encoding，可以让相邻的key共享一次，只存一次，从而节省10%的空间开销。sequence ID garbage collection也是较好的优化措施。当一个key的sequence number小于最老的snapshot的sequence number，则它的sequence number可以改写为0，而不影响数据的正确性。sequence number压缩率的提高带来了空间利用率的提高。

加压缩也可以优化空间放大。目前RocksDB支持多种压缩方法。第三方压缩方法有LZ、Snappy等方法。我们还可以以SST文件作为单元来设置不同的压缩算法，从而对KV数据进行压缩。降低空间消耗的同时会增加读开销，压缩过程中也会消耗一定的CPU资源，读的过程中还要进行解压，这就要求我们必须进行权衡。因此实践的优化策略通常为：L0至L2层不压缩，L3及以下使用较为轻量级的压缩算法，如Snappy、LZ，可以综合考虑压缩带来的空间开销受益和读性能受益。最后一层采用压缩率更大的压缩算法，一方面是因为数据较老，读到的概率较低，另一方面是因为越往下的层的容量阈值越大，存储的数据也越多，压缩率高可以节省较大的空间开销。       

![图片](..\images\LSM_Compact_Strategy_twoBasic-降低空间放大-7.png)

RocksDB也在Bloom filter方面进行优化实践。Bloom filter本身会有相应的内存开销，在内存有限的情况下，可以减少Bloom filter对内存的占用。方法之一是最后一层数据不再建立bloom filter。因为最后一层的数据量较大，Bloom filter本身占用的空间也比较大，会消耗较多内存，且考虑到最后一层的数据较老，被访问的概率较小，因此在最后一层或最后几层不再建立Bloom filter，在节省内存消耗的同时对读性能影响小。    

![图片](..\images\LSM_Compact_Strategy_twoBasic-其他实践.png) 


还有其他的优化方法，比如compaction job 并发。在RocksDB中，compaction在后台异步进行执行，执行过程中分为不同compaction job，如果两个compaction job之间的SST集合互相不重叠，则这两个compaction job可以实现现场并发。L0层的SST文件通常互相重叠，RocksDB对此进行优化，提出subcompaction概念，将L0层有SST重叠的情况也做并发。IPDPS 2014的一篇论文还提出，compaction之间可以做流水线。

结语
相比于传统的B+树，LSM-Tree结构具有更好的写性能，可以将离散的随机写请求转换成批量的顺序写操作。其性能衡量主要考虑三类因素：空间放大、读放大和写放大。LSM-Tree最初的设计理念是想通过空间放大和读放大来换取写放大的降低，从而达到极致的写性能，但也需要做好三方面因素的权衡，可以从降低写放大、降低读放大、降低空间放大三方面来对LSM-Tree结构进行优化，从而提升数据库性能。

 作者：腾讯云数据库 https://www.bilibili.com/read/cv18223855 出处：bilibili