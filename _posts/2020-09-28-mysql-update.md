---
layout: post
title: "MySQL 更新语句如何执行的"
subtitle: 'MySQL update'
author: "cslqm"
header-style: text
tags:
  - MySQL
---

# 更新语句如何执行的

更新和查找是差不多的

## redo log 和 binlog
两中日志的不同
1. redo log 是 InnDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。
2. redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1”。
3. redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换下一个，并不会覆盖以前的日志。

## 执行器和 InnoDB 的 update 流程


1. 执行器先找引擎取 ID=2 这一行。ID 是主键，引擎直接用树搜索找到这一行。如果 ID=2 这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁 盘读入内存，然后再返回。 
2. 执行器拿到引擎给的行数据，把这个值加上 1，比如原来是 N，现在就是 N+1，得到 新的一行数据，再调用引擎接口写入这行新数据。 
3. 引擎将这行新数据更新到内存中，同时将这个更新操作记录到 redo log 里面，此时 redo log 处于 prepare 状态。然后告知执行器执行完成了，随时可以提交事务。 
4. 执行器生成这个操作的 binlog，并把 binlog 写入磁盘。 
5. 执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成提交（commit）状 态，更新完成。


redo log 的写入拆成了两个步骤： prepare 和 commit，这就是"两阶段提交"。

## 两阶段提交
反证法，证明为何需要两阶段提交

如果不用两阶段提交，要么就是先写完 redo log 再写 binlog，或者采用反过来的顺序。我们看看这两种方式会有什么问题。


仍然用前面的 update 语句来做例子。假设当前 ID=2 的行，字段 c 的值是 0，再假设执 行 update 语句过程中在写完第一个日志后，第二个日志还没有写完期间发生了 crash， 会出现什么情况呢？

1. 先写 redo log 后写 binlog。假设 redgo log 写完，binlog 还没写完时，MySQL 进程异常重启。因为 redo log 写完了，可以将数据恢复，恢复后这一行 c 的值是 1。
由于 binlog 没写完就 crash 了，这时候 binlog 里边没有记录这个语句。
如果需要用这个 binlog 来恢复临时库的话，由于这个语句的 binlog 丢失，这时临时库就会少一次更新，恢复出来的这一行 c 的值是 0，与原库的值不同。
2. 先写 binlog 后写 redo log。如果 binlog 写完之后 crash，由于 redo log 还没写，崩溃恢复以后这个事务无效，所以这一行 c 的值是 0。但是 binlog 里边有记录“把 c 从 0 改成 1”这个日志。所以，在之后用 binlog 来恢复的时候就多了一个事务处理，恢复出来的这一行 c 的值是 1，与原库的值不同。
