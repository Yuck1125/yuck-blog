---
title: InnoDB 内存和磁盘结构介绍
date: 2019-07-20 21:11:47
tags: ["InnoDB","Mysql"]
categories: "技术"
---
### 前言
&emsp;&emsp;本来只是想了解下redo、undo log的机制，但发现好像牵扯挺多知识点，就写了这篇文章记录下。。。
### InnoDB 架构
**本文分析的mysql版本为8.0**  
![](/img/innodb-architecture.png)
### 一  InnoDB 内存结构
####  1.1 Buffer pool  
&emsp;&emsp;**Buffer   pool(下文简称BP)** 是在主内存中的一块区域，用于在访问时缓存表和索引数据。它可以直接从内存处理数据,因此处理速度非常快。  
&emsp;&emsp;为了提高大容量读取操作的效率，BP被分成可以容纳多行的**Page**（默认16K）。BP的底层数据结构是链表，以此管理Page。  
![](/img/innodb-buffer-pool-list.png) 
##### 1.1.1 Buffer Pool LRU 
&emsp;&emsp;BP的LRU是一种变体。BP插入数据时，采用了**中间策略（midpoint insertion strategy）**。中间策略将BP视为两个子列表：
```
new sublist : 存放最近访问的子列表
old sublist : 存放最近访问较少的子列表 
```
LRU算法主要的操作如下：
* 默认将3/8的BP空间分配给old sublist
* 规定**midpoint**为 `new sublist`的tail 与 `old sublist`的head交界处
* InnoDB读取Page到BP时，插入到**midpoint（old sublist的头部）**。Page被读取有两种情况:
	* 用户的sql查询
	* InnoDB的预读操作
* 当访问old sublist的Page时，会将其移到new sublist的head,实际上几乎所有的read Page操作都会将其移动到new sublist的head。除了预读的Page的情况,预读后如果一直没其他的读操作，该Page最终会从tail剔除
* 随着数据库的运行,一直没有被访问的page会被移向tail，最终被剔除

##### 1.1.2 Making the Buffer Pool Scan Resistant
默认情况下，会出现以下两种case
* 因表扫描，大量数据读入BP，但这数据之后不再被使用 
* 由预读加载然后仅访问一次的Page移动到新列表的头部  
 这两种情况可以将经常使用的page移向到旧子列表中，最终导致被剔除。  
  因此InnoDB做了`Making the Buffer Pool Scan Resistant`这个优化。简单的说就是读取Page插入到BP的时候不是插入到**midpoint**而是先插入到**old sublist的tail**   ，第一次被读取的时候，再移到**midpoint**。  
  另外的一些参数优化可参考官方文档：
   > [Making the Buffer Pool Scan Resistant](https://dev.mysql.com/doc/refman/8.0/en/innodb-performance-midpoint_insertion.html)  
    >[Configuring InnoDB Buffer Pool Prefetching (Read-Ahead)](https://dev.mysql.com/doc/refman/8.0/en/innodb-performance-read_ahead.html)  

####  1.2 Change Buffer
&emsp;&emsp;**Change Buffer(以下简称CB)** 存在于内存，当对辅助索引（secondary index） 进行DML操作时，BP没有其相应的Page，会将这些变更缓存到CP中。当CB里的Page被read的时候，会被合并到BP中。**当脏页超过一定比例时，会将其flush磁盘中**。CB的内存默认占BP的25%。  
个人认为刷新页到磁盘时没有将CB的数据刷到磁盘，因为CB中的数据只有在被读到的时候才会和BP合并，因此数据库对应的旧数据也不会被读，所以也不急着和flush到磁盘。。  
![](/img/innodb-change-buffer.png)  
##### 1.2.1 CB带来的提升
&emsp;&emsp;原本修改BP不存在的Page需要先从磁盘读取（一次IO操作）到内存，然后写redo log。
&emsp;&emsp;引入CB后，会先缓存在CB，然后写redo log。  
**由于CB的存在，避免了从磁盘读取辅助索引到缓冲池所需的大量随机访问I / O。**  
辅助索引不支持Change Buffer的情况：
* 如辅助索引包含降序索引列
* 主键包含降序索引  
**PS. 降序索引在8.0以上版本才支持。**

#### 1.3. Adaptive Hash Index
&emsp;&emsp;**Adaptive Hash Index**对InnoDB在BP的查询有很大的优化。针对BP中**热点页数据**，构建索引（一般使用索引键的前缀构建哈希索引）。因为HASH索引的等值查询效率远高于B+ tree，所以当查询命中hash，就能很快返回结果，不用再去遍历B+ tree。
####  1.4. Log Buffer
&emsp;&emsp;**Log Buffer**是保存要**写入磁盘上**日志文件的数据的内存区域。默认大小16MB。相关参数设置如下
* innodb_log_buffer_size : 设置大小
*  innodb_flush_log_at_trx_commit : 刷新行为。默认值：1， 取值有0，1，2三种。
	* 0： 日志每秒刷新到磁盘。 未刷新日志的事务会在mysql崩溃中丢失
	* 1： 每次提交事务时，写入并刷新日志到磁盘
	* 2： 每次提交事务后写入日志，并每秒刷新一次磁盘。 未刷新日志的事务可能会在mysql崩溃中丢失。
* innodb_flush_log_at_timeout：每几秒刷新日志，取值范围 [1,2700] (second)

### 二 InnoDB 磁盘结构
####  2.1 Tablespaces
##### 2.1.1 The System Tablespace
**The System Tablespace** 是Doublewrite Buffer和Change buffer的储存区域，也有用户创建的表和索引数据。该空间的数据文件通过参数`innodb_data_file_path`控制，默认值是`ibdata1:12M:autoextend`(文件名为ibdata1，大小略大于12MB，自动扩展)。   
**8.0之后InnoDB将元数据（以前的.frm文件，存表结构）存在该区域的数据字典中（data dictionary）**。
##### 2.1.2 File-Per-Table Tablespaces
**File-Per-Table Tablespaces** 默认开启,为每个表都独立建一个.ibd文件。 通过参数` innodb_file_per_tabl`  可以设置关闭，这样的话所有表数据是都存在The System Tablespace的ibdata。
##### 2.1.3 General Tablespaces
**General Tablespaces** 是通过`CREATE TABLESPACE`创建的共享表空间。
##### 2.1.4 Undo Tablespaces
**Undo Tablespaces**保存的是undo log ，用于回滚事务。  
该表空间有rollback segments,**rollback segments**是用于存 **undo log segments**, 而**undo log segments**存的就是undo logs。
mysql启动的时候，默认初始两个undo tablespace。因为sql执行前必须要有rollback segments。而两个undo tablespace才支持**automated truncation of undo**。
>[truncating  undo tablespaces](https://dev.mysql.com/doc/refman/8.0/en/innodb-undo-tablespaces.html#truncate-undo-tablespace)
##### 2.1.5  Temporary Tablespaces
InnoDB把 **Temporary Tablespaces**分为两种，**session temporary tablespaces** 和**global temporary tablespace**。
**session temporary tablespaces**存储的是用户创建的临时表和内部的临时表，一个session最多有两个表空间（用户临时表和内部临时表）。  
**global temporary tablespace**储存用户临时表的回滚段（rollback segments ）。
####  2.2 Doublewrite Buffer 
**Doublewrite Buffer**位于The System Tablespace。在BP的页数据刷到磁盘真正的位置前，会先将数据存在doublewrite buffer。 这步操作是直接将数据作为顺序块，调用OS的fsync()方法写入到doublewrite buffer。  
虽然数据写了两次，但是性能还是比两次IO低的。
此外fsync保证了BP中的数据写到磁盘中，即使数据库挂了，还可以从doublewrite buffer中还原数据。  
还可以解决页断裂的问题
>https://www.cnblogs.com/cchust/p/3961260.html
#### 2.3 Redo log 
redo log记录的DML操作的日志，可以用来宕机后的数据前滚。（在log buffer的redo log日志会在宕机中丢失）
#### 2.4 undo log
**undo log**记录数据更改前的快照（感觉就是备份），在数据需要回滚就可以根据undo log恢复。  
那些undo log 记录关于在global temporary tablespace 的用户临时表的回滚信息，不会在回滚中恢复。
### 总结
个人记录，并不一定都是对的，还是得好好看看官方文档,不断琢磨。。。
>https://dev.mysql.com/doc/refman/8.0/en/innodb-in-memory-structures.html
https://dev.mysql.com/doc/refman/8.0/en/innodb-on-disk-structures.html
