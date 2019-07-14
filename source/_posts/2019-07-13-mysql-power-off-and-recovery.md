---
title: mysql的断电恢复能力
date: 2019-07-13 15:14:15
categories: 数据库
tags: 
     - mysql
     - acid
---
这段时间对mysql数据库事务的acid做了一些深入的研究，理解了如何在异常情况下保证事务的acid。而最严重的异常情况莫过于断电，随时都有可能发生且发生时不会有任何保存的机会，所以就想以事务处理的各个阶段发生断电的情况下mysql如何保证acid为切入点总结一下这段时间所学到的东西。

<!-- more -->
# 概述
这段时间对mysql数据库事务的acid做了一些深入的研究，理解了如何在异常情况下保证事务的acid。而最严重的异常情况莫过于断电，随时都有可能发生且发生时不会有任何保存的机会，所以就想以事务处理的各个阶段发生断电的情况下mysql如何保证acid为切入点总结一下这段时间所学到的东西。  
本文将重点介绍commit命令执行的各个阶段，mysql是如何进行断电恢复的。

# mysql的page、日志和buffer介绍
## page
mysql的page与文件系统的block类似，是mysql自己定义的读取和写入磁盘的最小单位，其目的是为了充分的利用磁盘的顺序访问速度快的优势。一般是磁盘扇区的4~16倍。
## logs
 * undolog。即 mvcc 历史快照，记录的是某行过去的某个版本，用于帮助事务回滚及 MVCC 的功能。存储在表空间，以 transaction 为维度记录，且记录了 transaction 状态 ，可用于 crash 恢复时提取未提交的事务。可以理解成一张特殊的表，也会用到 buffer pool 。
 * redolog。可以理解为内存数据的WAL（write-ahead-log），用来恢复未flush到disk的page数据。记录的是对页的二进制级别的修改操作，比如某个偏移写入'aa'记录。** 一般存储在单独的redo log文件中，数据库全局的。另外要注意的是， undolog 的写入也会生成对应的 redo log 。**
 * binlog。即我们常说的二进制日志，主要用于主从复制。

## buffer
 * buffer pool。即 mysql page 的 buffer 。使用 lru 来管理。
 * doublewrite buffer。在 flush page 到磁盘之前，会先将 page 写入这里，为了防止 page flush 到一半时断电造成的磁盘 page 数据不完整的问题。存储介质虽为文件，但充分利用顺序写，所以很快。
 * log buffer。即 redolog 的内存 buffer 。 用于降低 redolog 对于写入速度的影响。  

除了这些还有 change buffer 等，不影响接下来的内容介绍，故不在此处展开讨论。

# mysql写入过程简介
##  commit 之前
先说说 commit 命令之前，写入操作对应的过程。 此时所有的写入操作都会通过 undolog 的支持，同时保留写入之前的版本和写入之后的版本，而这里对表数据和 undolog 的修改并不会直接写入到磁盘中，而是写入 buffer pool 中，同时生成相应的 redolog 。而此时的 redolog 存放在 log buffer 中，这样做的目的是为了减少 redolog 对写入性能的影响。想象一下，如果每次写入产生的 redolog 都写入磁盘，即使是顺序写入，也会造成不小的影响。所以，如果一个 transaction 中包含大量的写入操作，最好将 log buffer 调大一些，避免不必要的磁盘 io 。
下面说说 commit 的执行过程。由于在生产环境中，绝大多数 mysql 实例都会开启 binlog ，因此我们主要介绍开启 binlog 时 commit 的实行过程。
##  commit 执行过程
在开启 binlog 的情况下， mysql 使用 2PC 来执行 commit 过程，具体步骤如下： 
 * 阶段1：将 undolog 中的 transaction 状态改为 prepared ，并生成相应的生成 redolog ，然后 flush log buffer 。
 * 阶段2：将 transaction 写入 binlog ，然后告诉底层存储引擎将 undolog 中的 transaction 状态改为 commited 并生成 redo log 。只要写入binlog成功，就可以认为事务写入成功。  

另外需要注意的是，此时 mysql 使用 xid 来标识该事务。

# 断电恢复过程  
## 完成了 prepare 阶段，写入 binlog 之前断电
这种情况下， mysql 重新启动时，会从 redolog 中读出未 flush 到磁盘中的 page —— buffer pool 。然后从 redolog 重建这些内存中的 page ，以恢复断电之前内存的状态。之后，mysql检测到该事务并未提交，因此主动执行事务的回滚操作。

## flush buffer pool 时断电
即 flush page 的时候。此时造成 page 一部分数据写入磁盘，另一部分数据未写入磁盘。这时无法从 redo log 恢复异常 page 。原因是：之前提到 redo log 保存的page的修改记录，即从原 page 到新 page 的 patch ，而此时原 page 已经丢失（因为 page 的一部分数据已经写入磁盘），所以无法从 redo log 恢复。这时就要用到 double write buffer 了。每次 flush page 之前，都会先将 page flush 到一个 double write buffer ，如果在真正 flush page 的时候出现异常，则可以从 double write buffer 恢复这个 page 。

## flush log buffer 时断电
即 redolog 的flush。由于 redolog 的 flush 是以扇区为单位进行 flush 的，而扇区的写入是原子性的（当代的大部分硬盘都有这个能力），因此不会造成部分写入、部分未写入的情况。

## 写完 binlog 后断电
上面说到，写入binlog时，便可认为事务提交成功，就可以保证acid中的d。此时如果断电，mysql在启动会执行如下过程：
1. 先从binlog中筛选出所有xid并创建hash_list
2. 然后从存储引擎（如innodb）中选出prepared状态的transaction的xid列表list。一般是从rollback segments(rseg) —— undo log ——中提取xid列表，然后在选出prepared状态的。
3. 遍历list的xid，如果xid在hash_list中则commit，否则rollback。

## 写入 binlog 时断电  
binlog 的写入也是没有 double write buffer 保护的，那么在写入 binlog 的过程中如果发生了断电， mysql 是如何恢复的呢。如上面介绍的写入 binlog 后断电恢复过程所示， mysql 会从 binlog 中选出所有的 xid 并创建恢复列表，不过在遍历 binlog 的过程中， mysql 同时也会寻找 last valid position ，并将这个 position 之后的内容都截取掉。因此，如果某个 transaction 写入 binlog 的过程中断电，并且有一部分未成功写入，则该 transaction 在 mysql 重启时会被回滚。

# 相关配置
这些配置是mysql5.7之后的默认配置：  
 * sync_binlog=1
 * innodb_support_xa=ON
 * innodb_flush_log_at_trx_commit=1。用于控制 log buffer 的刷新时机。

# 参考资料
《Mysql技术内幕》  
https://dev.mysql.com/doc/refman/5.7/en/glossary.html  
mysql5.7源码——sql/binlog.cc  