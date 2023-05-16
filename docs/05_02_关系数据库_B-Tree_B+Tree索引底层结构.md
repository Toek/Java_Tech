# Oracle、Mysql及postgreSQL存储区别

## postgreSQL15官方文档

B树结构：https://www.postgresql.org/docs/15/btree-implementation.html#BTREE-STRUCTURE

数据页相关：https://www.postgresql.org/docs/15/storage-page-layout.html

> ### 67.4.1. B-Tree Structure
>
> PostgreSQL B-Tree indexes are multi-level tree structures, where each level of the tree can be used as a doubly-linked list of pages. A single metapage is stored in a fixed position at the start of the first segment file of the index. All other pages are either leaf pages or internal pages. Leaf pages are the pages on the lowest level of the tree. All other levels consist of internal pages. Each leaf page contains tuples that point to table rows. Each internal page contains tuples that point to the next level down in the tree. Typically, over 99% of all pages are leaf pages. Both internal pages and leaf pages use the standard page format described in [Section 73.6](https://www.postgresql.org/docs/15/storage-page-layout.html).

> **Table 73.2. Overall Page Layout**
>
> | Item           | Description                                                  |
> | -------------- | ------------------------------------------------------------ |
> | PageHeaderData | 24 bytes long. Contains general information about the page, including free space pointers. |
> | `ItemIdData`   | `Array of item identifiers pointing to the actual items. Each entry is an (offset,length) pair. 4 bytes per item.` |
> | Free space     | The unallocated space. New item identifiers are allocated from the start of this area, new items from the end. |
> | Items          | The actual items themselves.                                 |
> | Special space  | Index access method specific data. Different methods store different data. Empty in ordinary tables. |

## Oracle 21官方文档

oracle的存储结构：https://docs.oracle.com/en/database/oracle/oracle-database/21/cncpt/indexes-and-index-organized-tables.html

官方图片如下：

![Description of Figure 5-1 follows](..\images\05_02_关系数据库_常用索引分类_oracle_b+tree_structure-1.png)

## Mysql官方文档

https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_b_tree

https://dev.mysql.com/doc/dev/mysql-server/latest/structddl_1_1Builder.html#a030cf290c4e90d94d88d7100557e828a

> Load the sorted data into the B+Tree.



## 结论：

1. Oracle和Mysql默认使用的是B+树存储，postgreSQL默认使用的是B树存储
2. Oracle的叶子节点存储的是rowid，而Mysql叶子节点存储的是用户数据
