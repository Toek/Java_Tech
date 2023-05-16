[PostgreSQL B+Tree论文解读2 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/166398779)

## PostgreSQL B+Tree论文解读2 - 《A SYMMETRIC CONCURRENT B-TREE ALGORITHM》

## 1. 背景

LY算法解决了对B+Tree的高并发insert, update, search的问题。但是page的删除和回收并没有系统的解决，只提到了可以采用offline的方式来回收。

PostgreSQL的BTree中实现的delettion算法是基于《A SYMMETRIC CONCURRENT B-TREE ALGORITHM》论文，关键实现如下：

1. 不会对B+Tree的page直接删除，删除总是从叶子节点的leaf item开始，当一个叶子节点所有的item都被删除了，这个page也就成为了空的叶子page；

2. 空的叶子page会被删除回收，如果该page是父节点唯一的子节点，那么父节点也需要被删除回收，一直递归到一个至少拥有2个的祖先节点；

3. btbulkdelete算法：vacuum时不需要直接遍历B+Tree的数据结构，而是直接从page0开始遍历该B+Tree的索引文件，平铺式的遍历所有page，通过page上的标记识别出哪些page需要回收。好处是可以启动预读，加大读IO的带宽；

4. 叶子page delete的2阶段

5. 1. stage1：被删除的page从parent节点中摘除掉，同时标记该page为half-dead；
   2. stage2：被标记为half-dead的page从它的左右邻居节点中摘除掉；

6. 中间page delete的2阶段

7. 1. 如果叶子节点是父节点唯一的子节点，那么父节点需要删除；
   2. stage1: 递归的往上查找，至少拥有2个子节点的祖先节点，删除是从这个祖先节点往下逐个删除直到叶子节点。好处是，可以原子的在祖先节点中把这个subtree剔除掉，同时也满足了删除始终从叶子节点开始（如果先删除了叶子节点，就找不到这些中间待删除的节点了）；
   3. stage2：从topmost开始删除，每次删除一个topmost时，需要把下一个记录到叶子节点上，在宕机重启时可以从这个叶子节点中读取topmost，继续上次的删除；

## 2. Abstract

该论文主要解决LY算法没有涉及的BTree删除问题。通过增加一个类似rightlink的方法，给删除的page增加一个outlink，在发生了删除时，把当前page的内容合并到左边，并且通过outlink指向左边的节点。这样并发scan的进程可以通过outlink找到合并之前的key。

## 3. Btree算法演进

LY论文中提到过一次btree的算法演进。

## lock-coupling(读写隔离)

从父节点到子节点descengding操作需要：先对子节点上锁，再释放当前父节点的锁。 这里，需要同时对父子节点上锁，否则其他writer就可能有机会同时得到这两个锁并完成分裂过程。

![img](..\images\05_03_关系数据库_PostgreSQL_扩展_ B+Tree论文解读lock-coupling.png)



一个执行时序：

1. write1开始执行search(d)搜索；
2. writer1对父节点进行的搜索，发现需要进入‘abd’的子节点进行搜索；
3. writer1释放d的锁；
4. writer2执行insert(c)操作；
5. writer2上锁'd'节点成功；
6. writer2上锁‘abd'节点成功；
7. writer2对’abd‘进行了分裂，产生了’ab','cd'两个节点；
8. writer1再去原来'abd'页面上搜索'd'时就找不到了；

## lock-subtree(写写隔离)

writer操作需要对subtree上锁，理论上锁的最小粒度是在他们的公共祖先地方上锁，达到写写互斥。分裂的过程是自底向上进行的，writer1对一个节点n上锁并且更新了，此时writer2可能已经在这个子树下面它最终也要修改n，如果不在subtree地方上锁，当一个writer1修改了n，另外一个自底向上后发现n已经变化了，导致插入的地方不对。 lock-subtree本身是比较难以实现的，一个最直观的做法是对root节点上锁，但是这样就彻底阻止其他的操作了，尤其是并发的lock-coupling，更加剧了冲突。

## optimitic descending(乐观更新)

writer过程中对中间节点上读锁，对真正要更新的叶子节点上写锁，如果最终没有发生分裂或者合并，则更新成功，否则需要从root开始执行lock-subtree算法。

## update during decending(自上向下更新)

避免了lock-subtree，在更新操作的descending过程中，同时对节点重构。过程中仍然需要lock-coupling。

## B-link-tree算法

LY算法的思想是不通过锁来互斥btree的操作（lock-coupling和lock-subtree），而是允许btree操作并发的进行，通过其他手段来恢复出btree的结构。

### B-link-tree

这是LY的原创发明，B+tree每个节点都额外增加一个‘rightlink’指向它的右邻居节点。允许btree的操作并发执行，后续再根据rightlink来复原出完整的btree。

比如，分裂操作分成2个阶段：

1. 水平操作：将原节点n分成n和n'，将key也分成两拨，同时节点n指向节点n'；
2. 垂直操作：将新的节点n'插入父节点中；

在1和2之间允许其他进程从父节点进行搜索，进入节点n，如果：

1. 目标key在节点n中，则搜索n后返回；
2. 目标key在n'中，通过节点n跳到n'中搜索；

lock-coupling不需要了，因为通过父节点descend到子节点的过程中，如果有对子节点进行的分裂没有即使在插入父节点，导致错误的进入了左子节点，此时只需要通过rightlink往右搜索就能找到正确的子节点； lock-subtree不需要了，因为在ascend到父节点中进行插入新的n'时，此时如果父节点也发生了分裂，导致进入了错误的父节点，那么只需要通过rightlink往右搜索就能找到正确的父节点；

B-link树允许插入和搜索只对一个page上（这里有个假设，所有的page直接从磁盘上读取到自己进程的local buffer中。PostgreSQL中B-link树的page读取到shared buffer中被所有进程共享，因此它需要锁住更多的页面。这是由具体的工程实现导致的，并非算法本身的要求）。

## 4. Symmetric Deletion Approach

## 算法思想

扩展LY算法的思想（允许并发的合并节点，通过额外的指针提供一致性的BTREE语义），额外增加outlink指针，其他进程能够通过该指针完整的遍历BTREE。

1. 向左邻居节点合并；
2. 增加outlink指向左邻居节点；
3. 并发的search时可以通过outlink找到合并之前的key；

## 算法细节

### 该论文中的BTree状态图

![img](..\images\05_03_关系数据库_PostgreSQL_扩展_ B+Tree论文解读论文中的BTree状态图.png)



1. top pointer;
2. fast pointer;
3. downlink;
4. empty node;
5. outlink;
6. rightlinik;
7. leaft's separator(highkey);
8. rightsep(d)，d是一个donwlink：返回d右边的key，这个key始终和该downlink在一个node中；
9. rightsep(n)，n是一个non-empty节点：返回节点n的最大highkey；
10. leftsep(d)，d是节点n的一个donwlink：返回d左边的key，如果d已经是最左侧的downlink了，那么返回左侧邻居的highkey；
11. leftsep(n)，n是一个non-empty节点，d是最左侧downlink，leftsep(n) = leftsep(d)：返回这个node的下界；

### locate phase

删除v时，需要先从root节点search到v属于的叶子节点。 search的过程中只对单个节点上锁，而不需要lock-coupling的方式。在descend过程中通过**downlink，rightlink，outlink**向下查找（并发insert/delete导致BTree处于一个中间的状态）。 在找到叶子节点后，对叶子节点上读锁或者写锁。

### reconstruct phase

### Two-Phase split

![img](..\images\05_03_关系数据库_PostgreSQL_扩展_ B+Tree论文解读-Two-Phase split.png)



1. half-split;
2. add-link;

### Two-Phase merge

![img](..\images\05_03_关系数据库_PostgreSQL_扩展_ B+Tree论文解读-Two-Phase merge.png)



1. half-merge;
2. remove-link;

无论是split还是merge，在第一个phase进行中，父节点也可能发生过操作，那么ascend时真正的父节点是：

1. split：分裂之后左边节点的rightmost，可以通过父节点的rightlink进行查找；
2. merge：合并之前右边的lefttmost，在父节点的outlink查找；

### Height of the Structure

在对父节点进行ascend分裂时，如果一直分裂到了top level（根节点），那么树的高度就增长了1。 安全的删除top level是非常困难的（多个进程内存有该节点page的缓存；无论如何删除，到root节点的路径上都会有一条路径，这条路径上每个节点都有一个子节点）。

fast level：拥有至少2个子节点的最高level，大于该level的都有只有一个节点；

anchor节点维护指向top level节点，同时也指向最新的fast leve节点，后续的搜索可以从fast leve直接开始，跳过从root到该fast层，因为上面的路径只有一条。

每次插入、删除时都需要维护该fast leve的正确性。

### Freeing Empty Nodes

当btree中所有指向这个page的指针都为空时，这个page就从btree结构中彻底解除了。 并不能立即re-use回收再利用，因为可能有进程通过它的左右节点的指针（并发搜索进程之前读取到了它的邻居节点）访问到这个page。如果该page被回收重用了，那么相同的page号内容就不同了。

延迟删除：等待所有在locate phase引用了该page的进程都退出； 或者每个节点额外记录height和leftsep信息，可以让进程识别出是否即将扫描到了一个删除节点；

PostgreSQL做法是：等待所有活跃事务退出；

1. 这是一个充分不必要条件；
2. 当page被标记为dead时，page上记录next-transaction counter value；
3. vacuum可以回收该page，如果该事务ID比RecentGlobalXmin小（说明page被标记为dead时的活跃事物都已经退出，新的事务也不会引用到该deadpage）；
4. 副作用
5. 等待没有Snapshot的事务XID；
6. 必须等待到有一个新的事务被分配，而且commit了；

## 5. 正确性

该论文提到的证明方法对如何证明一个数据结构和算法的正确性具有代表性。 思路更像是形式化的内涵：

1. 通过集合定义数据结构可能有的合法的状态，比如BTree结构的状态；
2. 定义作用在集合上的一系列的action，比如search, inser, delete；
3. 定义action之间的偏序关系，比如操作的并发和顺序关系（Lamport序）；

使用集合描述BTree的特性，所有的search状态从s转换到s'，对于所有属于s的节点n： 如果s中满足leftsep(n)=l, s'中满足leftsep(n)=l', 那么 l' <= l；

最终通过证明该算法产生的所有action序列都等价于serializable。

## 6. 总结

该论文主要讨论如何系统的删除回收BTree中的节点，是对LY算法的一个补充，PostgreSQL的工程实践中还需要考虑的是：

1. vacum加速：vacuum直接顺序的扫描btree物理文件，需要处理在并发的split时可能错过部分标记deleted tuples。因为是顺序扫描因此可以预读；
2. 多阶段删除；
3. 何时可以reuse一个page；
4. WAL如何组织；