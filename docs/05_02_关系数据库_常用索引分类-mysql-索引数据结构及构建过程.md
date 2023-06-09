## 深入了解Mysql索引数据结构                 

[JAVA葵花宝典                      ](javascript:void(0);)                       **JAVA葵花宝典**                               ![img]()                               微信号                               Javakhbd功能介绍                               墙裂置顶，每天准时推送干货，SpringBoot/ SpringCloud/Dubbo/Docker /Linux /Mysql /Mybatis/MQ/大数据处理/中间件/中台技术！                                                                                 *今天*

收录于话题

以下文章来源于Java渣渣菜                                                                                                     ，作者豆豆和豆芽                                                             



### 目录

#### 1: 索引结构

```
    ** 哈希表
    ** 有序数组
    ** 二叉树
    ** 多叉树
```

#### 2: 多叉树索引维护

## 一：索引结构

提到数据库索引大家肯定不陌生，那到底什么是索引呢，索引是怎么工作的呢，今天就一起来聊聊这个话题
索引的出现就是为了解决数据库查询的效率问题，就像平时我们看书一样，想要找某个详细的内容，就先通过目录去找到大概的地方，再找具体的内容，索引就是数据库中的“目录”
下面我们进入今天的正题，索引数据结构的类型

#### 哈希表

哈希表是一种以（Key-value）的形式存储的数据结构，这种数据结构我们接触的最多的就是HashMap的数据结构了。通过Key找到Value，哈希表的数据存储很简单，把value存在数组中，这个value存的的位置就是key通过哈希函数转换得到的，不同的key通过哈希函数得到的值可能是一样的，针对这种情况会拉出一个链表，相同位置的value存放在链表上，然后遍历链表的值。哈希表的特点就是查询等值的场景，做范围查询性能都是不好的。

#### 有序数组

有序数组在等值查询和范围查询的场景下查询效率都特别好，但是索引有更新的时候有序数组就遇到大问题了，所以有序数组适合静态数据的存储。

#### 二叉树

二叉搜索树的特点是：每个节点的左儿子小于父节点，父节点又小于右儿子。树可以有二叉，也可以有多叉。多叉树就是每个节点有多个儿子，儿子之间的大小保证从左到右递增。二叉树是搜索效率最高的，但是实际上大多数的数据库存储却并不使用二叉树。其原因是，索引不止存在内存中，还要写到磁盘上。为了让一个查询尽量少地读磁盘，就必须让查询过程访问尽量少的数据块。那么，我们就不应该使用二叉树，而是要使用“N 叉”树。这里，“N 叉”树中的“N”取决于数据块的大小。

#### Innodb引擎的索引

Innodb使用的是B+树的数据结构存储索引数据，每一个索引在引擎里面都有一颗对应的B+树
假设，有个表主键为Id，有个字段X，在X上有索引。往表中插入一些值
（1，10）（2，20）（3，30）（4，40）（7，70）（8，80），下面是索引的组织结构

![img](..\images\05_02_关系数据库_常用索引分类-mysql-InnoDB-索引结构.png)从图中不难看出，主键索引（聚簇索引）存的是对应的数据，非主键索引（二级索引）存的是主键id

##### 主键索引和非主键索引的区别

select id from table_name where id = ? 这种查询只要访问主键索引就能拿到结果
select x from table_name where x = ? 这种要先访问非主键索引找到id,再访问主键索引拿到值，这个过程叫做回表。这样就会多扫码一颗索引树。

## 索引维护

### 页分裂和页合并

如上图，如果在id=4后面加入id=5的记录，需要在逻辑上移动后面的数据，如果在id=7后面加入新数据，直接在后面插入就好，但是如果R7所在的数据页满了，根据B+树的算法，需要重新申请一个新的数据页，把数据移动到新的数据页，这个过程叫页分裂，性能受到影响，还会降低空间的利用率。合并其实就是一个分裂的逆过程。

### 自增主键和非自增主键的区别

自增主键的数据插入，是一个递增插入的场景。每次插入一条新记录，都是追加操作，都不涉及到挪动其他记录，也不会触发叶子节点的分裂。而非递增主键都相反。
主键的长度越小，非主键索引存的是主键的值，那么非主键索引占用的空间就越小。

## END

今天的内容就先讲到这里，不知道大家有没有收获，下篇文章会给继续给大家带来索引相关的知识，比如说覆盖索引，最左匹配原则等等，下次再见。
