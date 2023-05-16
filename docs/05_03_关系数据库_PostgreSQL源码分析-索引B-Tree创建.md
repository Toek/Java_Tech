[PostgreSQL 创建B-Tree索引的过程 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/351302989)

Postgres支持B-tree, hash, GiST, and GIN，也支持用户通过Gist自定义索引方法，比如时空数据库的R-Tree索引。

为了支持索引框架，在创建索引时会查找和操作一系列Catalog元数据，另外为了加速B-Tree索引的构建，会先对待创建索引的数据进行排序，然后再按照B-Tree的页面格式直接写B-Tree的page，避免page的split。

## 例子

```text
create table t001(id int, i int, j int, k int, msg text);
create table t002(id int, i int, j int, k int, msg text);
insert into t001 select i, i+1, i+2, i+3, md5(random()::text) from generate_series(1,1000) as i;
insert into t002 select i, i+1, i+2, i+3, md5(random()::text) from generate_series(1,1000) as i;

create index t001_i_j on t001(i, j);
```

创建B-Tree索引分成2个部分:

1. catalog系统中生成新索引的相关元数据（索引文件也是一个表，因此相关的元数据需要建立和关联起来）；
2. 对索引列进行排序并生成BTree的page；

## 校验新索引的Catalog元数据

## 语法解析

```text
create index t001_i_j on t001 using btree (i, j);
```

通过parse.y将创建索引的sql解析成IndexStmt结构，其中：

```text
IndexStmt
{
    type = T_IndexStmt
    idxname = "t001_i_j"
    accessMethod = "btree"
}
```

## 校验B-Tree的handler

using btree执行了索引的类型是btree，因此需要校验内核是否支持该类型的索引。

### pg_am

```text
tuple = SearchSysCache1(AMNAME, PointerGetDatum("btree"));
```

在pg_am中查找"btree"对应的handler

```text
postgres=# select oid,* from pg_am;
 oid  | amname |  amhandler  | amtype
------+--------+-------------+--------
  403 | btree  | bthandler   | i
  405 | hash   | hashhandler | i
  783 | gist   | gisthandler | i
 2742 | gin    | ginhandler  | i
 4000 | spgist | spghandler  | i
 3580 | brin   | brinhandler | i
(6 行记录)
```

BTree的handler是bthandler，是一个regproc类型，对应pg_proc一个函数。对于内置的BTree索引，相关的函数是在bootstrap过程中插入(pg_proc.dat文件）。

## pg_proc

pg_proc.dat文件（下面代码会被重新格式化成postgres.bki文件，并且在inintdb时被bootstrap.y解析插入）：

```text
# Index access method handlers
{ oid => '330', descr => 'btree index access method handler',
  proname => 'bthandler', provolatile => 'v', prorettype => 'index_am_handler',
  proargtypes => 'internal', prosrc => 'bthandler' },
postgres=# select oid,* from pg_proc where proname='bthandler';
-[ RECORD 1 ]---+----------
oid             | 330
proname         | bthandler
pronamespace    | 11
proowner        | 10
prolang         | 12
procost         | 1
prorows         | 0
provariadic     | 0
protransform    | -
prokind         | f
prosecdef       | f
proleakproof    | f
proisstrict     | t
proretset       | f
provolatile     | v
proparallel     | s
pronargs        | 1
pronargdefaults | 0
prorettype      | 325
proargtypes     | 2281
proallargtypes  |
proargmodes     |
proargnames     |
proargdefaults  |
protrftypes     |
prosrc          | bthandler
probin          |
proconfig       |
proacl          |
```

## 校验索引列及比较函数

查找pg_attribute，校验create index中指定的索引列是否存在，如果存在记录attno，并且根据atttypid查找对应的比较函数；

### 校验pg_attribute

```text
postgres=# select a.*  from pg_class c, pg_attribute a  where c.relname='t001' and c.oid = a.attrelid;
-[ RECORD 8 ]-+---------
attrelid      | 16384
attname       | i
atttypid      | 23
attstattarget | -1
attlen        | 4
attnum        | 2
attndims      | 0
attcacheoff   | -1
atttypmod     | -1
attbyval      | t
attstorage    | p
attalign      | i
attnotnull    | f
atthasdef     | f
atthasmissing | f
attidentity   |
attisdropped  | f
attislocal    | t
attinhcount   | 0
attcollation  | 0
attacl        |
attoptions    |
attfdwoptions |
attmissingval |
-[ RECORD 9 ]-+---------
attrelid      | 16384
attname       | j
atttypid      | 23
attstattarget | -1
attlen        | 4
attnum        | 3
attndims      | 0
attcacheoff   | -1
atttypmod     | -1
attbyval      | t
attstorage    | p
attalign      | i
attnotnull    | f
atthasdef     | f
atthasmissing | f
attidentity   |
attisdropped  | f
attislocal    | t
attinhcount   | 0
attcollation  | 0
attacl        |
attoptions    |
attfdwoptions |
attmissingval |
```

### 查找比较函数

其中403是pg_am中btree的handler的oid

```text
postgres=# select * from pg_opclass where  opcmethod = 403 and opcintype  = 23;
-[ RECORD 1 ]+---------
opcmethod    | 403
opcname      | int4_ops
opcnamespace | 11
opcowner     | 10
opcfamily    | 1976
opcintype    | 23
opcdefault   | t
opckeytype   | 0
```

在经过上述元数据校验之后，可以开始为新的索引文件生成相关元数据了。

## 创建新索引的元数据

## 在文件系统中创建索引文件

### 生成Oid

```text
DefineIndex -> index_create -> GetNewRelFileNode
```

为新的索引文件生成唯一oid，过程是：生成一个新的oid，然后查找pg_class的索引，如果不存在就返回这个oid。

### 跟新本地的relcache

把正在创建的relation添加到relcache系统中

```text
DefineIndex -> index_create -> heap_create -> RelationBuildLocalRelation
```

### 创建文件

```text
DefineIndex -> index_create -> heap_create -> RelationCreateStorage
```

1. 文件系统中生成新的文件；

```text
路径：
 "/home/postgres/xxx/base/13881/16391"
```

1. 生成wal日志，类型是

```text
XLogInsert(RM_SMGR_ID, XLOG_SMGR_CREATE | XLR_SPECIAL_REL_UPDATE)
```

## 创建Catalog相关元数据

### pg_class

新的索引文件也是一个表文件，因此需要插入到pg_class中：

```text
DefineIndex -> index_create -> InsertPgClassTuple
postgres=# select * from pg_class where relname = 't001_i_j';
-[ RECORD 1 ]-------+---------
relname             | t001_i_j
relnamespace        | 2200
reltype             | 0
reloftype           | 0
relowner            | 10
relam               | 403
relfilenode         | 16396
reltablespace       | 0
relpages            | 6
reltuples           | 1100
relallvisible       | 0
reltoastrelid       | 0
relhasindex         | f
relisshared         | f
relpersistence      | p
relkind             | i
relnatts            | 2
relchecks           | 0
relhasoids          | f
relhasrules         | f
relhastriggers      | f
relhassubclass      | f
relrowsecurity      | f
relforcerowsecurity | f
relispopulated      | t
relreplident        | n
relispartition      | f
relrewrite          | 0
relfrozenxid        | 0
relminmxid          | 0
relacl              |
reloptions          |
```

### pg_attribute

把索引文件引用的列插入pg_attr中：

```text
DefineIndex -> index_create -> AppendAttributeTuples
postgres=# select a.*  from pg_class c, pg_attribute a  where c.relname='t001_i_j' and c.oid = a.attrelid;
-[ RECORD 1 ]-+------
attrelid      | 16396
attname       | i
atttypid      | 23
attstattarget | -1
attlen        | 4
attnum        | 1
attndims      | 0
attcacheoff   | -1
atttypmod     | -1
attbyval      | t
attstorage    | p
attalign      | i
attnotnull    | f
atthasdef     | f
atthasmissing | f
attidentity   |
attisdropped  | f
attislocal    | t
attinhcount   | 0
attcollation  | 0
attacl        |
attoptions    |
attfdwoptions |
attmissingval |
-[ RECORD 2 ]-+------
attrelid      | 16396
attname       | j
atttypid      | 23
attstattarget | -1
attlen        | 4
attnum        | 2
attndims      | 0
attcacheoff   | -1
atttypmod     | -1
attbyval      | t
attstorage    | p
attalign      | i
attnotnull    | f
atthasdef     | f
atthasmissing | f
attidentity   |
attisdropped  | f
attislocal    | t
attinhcount   | 0
attcollation  | 0
attacl        |
attoptions    |
attfdwoptions |
attmissingval |

postgres=#
```

### pg_index

把索引本身相关信息插入pg_index中，如果索引包含了表达式，则把表达式通过nodeToString序列化成字符串：

```text
DefineIndex -> index_create -> UpdateIndexRelation
postgres=# select * from pg_indexes where indexname = 't001_i_j';
-[ RECORD 1 ]-------------------------------------------------------
schemaname | public
tablename  | t001
indexname  | t001_i_j
tablespace |
indexdef   | CREATE INDEX t001_i_j ON public.t001 USING btree (i, j)

postgres=#
```

### relcache生效

为了使catalog元数据的变更对所有进程生效，把该heap相关的元数据invalid掉，此处仅仅是注册失效函数，在CommandCounterIncrement，开启执行下一个command时才真正执行invalid。

```text
DefineIndex -> index_create -> CacheInvalidateRelcache
```

### pg_depend

记录该索引对heap的依赖，对collations的依赖，对opclass的依赖，

```text
比如：
t001_i_j依赖表 t001的i；
t001_i_j依赖表 t001的j；
postgres=# select * from pg_depend where objid = 16396;
-[ RECORD 1 ]------
classid     | 1259
objid       | 16396
objsubid    | 0
refclassid  | 1259
refobjid    | 16384
refobjsubid | 2
deptype     | a
-[ RECORD 2 ]------
classid     | 1259
objid       | 16396
objsubid    | 0
refclassid  | 1259
refobjid    | 16384
refobjsubid | 3
deptype     | a
```

## CommandCounterIncrement

使得新索引文件相关的relcache生效

## 构建B-Tree索引

create index时通过"btree" ，找到bthandler函数，找到btbuild

```text
DefineIndex -> index_create -> index_build -> btbuild
```

## 排序

### 构建sortkey

通过index的索引列构建排序时需要用到的sortkey：

```text
btbuild -> _bt_spools_heapscan -> tuplesort_begin_index_btree
```

### 扫描heap表

在非concurrent模式下create index时，为了扫描所有的tpule，snapshot使用SnapshotAny，

```text
btbuild -> _bt_spools_heapscan -> IndexBuildHeapScan -> IndexBuildHeapRangeScan -> heap_getnext
```

## 自下向上构建B-Tree索引page

逐个读取排好序的tuple，填充到B-Tree的叶子节点上，自下向上插入B-Tree，插入叶子节点可能会递归的触发起父节点也插入。

### page layout

索引页面的内存布局整体上是遵循堆表的布局：pageheader，行指针数组构成。 不同点是： 1. 页面的尾巴上多了BTree自定义的BTPageOpaqueData结构，和左右邻居页面组织成链表； 2. 每个行指针指向的内容由IndexTupleData和key组成；

![img](..\images\05_03_PostgreSQL_B-Tree_pageLayout.png)

填充率

向刚刚创建出来的索引插入新的数据时，为了避免split，在第一次构建索引时，每个页面上都预留了一些空间： 1. 叶子节点90%； 2. 中间节点70%；

### 自下向上构建

1. 填充叶子page：leaf-0;
2. 当leaf-0满了，leaf-0落盘前需要把它的邻居和父节点相关指针更新；
3. 分配一个paret0，leaf-0插入parent0中；
4. 分配一个新的页面leaf-1，leaf-0的右指针指向leaf-1；
5. leaft-0可以落盘；
6. leaft-1也满了后，过程和2到6一样；
7. 当leaf-2满了后，要把leaf-2插入父节点，此时父节点也满了，因此要递归的把父节点落盘，过程和2到6一样；

### leaf-0

![img](..\images\05_03_PostgreSQL_B-Tree_pageLayout-leaf-0.png)



### leaf-0满了

![img](..\images\05_03_PostgreSQL_B-Tree_pageLayout-leaf-0-满了.png)

### leaf-1满了

![img](..\images\05_03_PostgreSQL_B-Tree_pageLayout-leaf-1-满了.png)



### leaf-2满了，先递归处理parent0满的情况

![img](..\images\05_03_PostgreSQL_B-Tree_pageLayout-leaf-2-满了-先处理p0满.png)



### leaf-2满了，再处理leaf-2自身满的情况

![img](..\images\05_03_PostgreSQL_B-Tree_pageLayout-leaf-2-满了-再处理leaf2自身满.png)



### indextuple和scankey的语义

1. 对于叶子节点，indextuple指向heap表的行指针；
2. 对于中间节点，indextuple指向大于scankey的页面，是一个左闭右开区间[scankey, )；

索引页面中真正存放的是indextuple和scankey的组合，在page这个层面中把这两个当做一个item，长度是两者之和；

在实际查找时，同时linp定位到indextuple，由于indextuple是定长的，只需要再往前移动sizeof(indextuple)就能找到scankey的datum和null区域；

根据index的attribute desc来解析datum和null；

### key space

![img](..\images\05_03_PostgreSQL_B-Tree_pageLayout-keySpace.png)



对于-Tree中间节点的任意一层，它的所有的indextuple和scankey并集在一起是[负无穷，正无穷]。 由于一对indextuple和scankey组合表达的是左闭右开区间[scankey, )： 1. 对于最右侧的scankey (k4)，它的语义本身就是[scankey，正无穷)； 2. 对于最左侧的scankey (k1)，这个key本身代表的是负无穷的语义： 使用该层出现的第一个key内容(记录在pagesate->minkey)； indextuple中只记录blkno，使用offset来记录attr为0； 不存储scankey的datum和null；

```text
if (last_off == P_HIKEY)
    {
        Assert(state->btps_minkey == NULL);
        state->btps_minkey = CopyIndexTuple(itup);
        BTreeTupleSetNAtts(state->btps_minkey, 0);
    }

    ...

    if (!P_ISLEAF(opaque) && itup_off == P_FIRSTKEY)
    {
        trunctuple = *itup;
        trunctuple.t_info = sizeof(IndexTupleData);
        BTreeTupleSetNAtts(&trunctuple, 0);
        itup = &trunctuple;
        itemsize = sizeof(IndexTupleData); //大小仅仅是indextuple，去除了datum和null
    }
```

### high key和first key

![img](..\images\05_03_PostgreSQL_B-Tree_pageLayout-keySpace-highKey_firstKey.png)



每个非最右侧页面都有highkey，highkey的语义是改页面所有key的上界（不包含），因此最右侧页面没有highkey，它的上界是正无穷。

页面1的highkey是在页面1满了之后，生成页面2时确定下来的： 1. 把页面1的highkey内容拷贝到页面2，做为页面2的FIRSTKEY； 2. 设置页面1的P_HIHEY指向最后一个行指针last_off，并且标记last_off为无效指针；

因此，页面1的highkey是页面2的FIRSTKEY。

### 未满页面的落盘

![img](..\images\05_03_PostgreSQL_B-Tree_pageLayout-未满页的落盘.png)

当所有的heap表都被插入了之后，还没有达到满的页面需要落盘，在自下向上的过程中，插入父节点时是一个递归的过程，为了维护每次递归的状态信息，每层的页面用一个BTPageState结构来描述：

1. 记录level；
2. 记录当前层的最右侧的page；
3. 记录minkey；

BTPageState结构是一个单向链表，因此在插入了所有heap表之后，只需要遍历这个链表，从叶子节点开始落盘。 注意：过程中会往往父节点插入(indextuple,scankey)，可能会触发父节点满了而落盘，因而产生一个新的父节点，所属层的BTPageState会被更新，沿着BTPageState链表会最终把这个新产生的页面也落盘了。

### metapage

metapage记录B-Tree：level，root，fastroot等信息。 metapge始终在索引文件的第0个page，root的位置是不固定的。