---
title: 分布式事务（XA、2PC、3PC介绍）
date: 2019-07-15 11:18:13
tags: "Distributed Transaction"
categories: "技术"
---
###  XA简单介绍
XA是由X / Open发布的规范，用于DTP（分布式事务处理）。
DTP分布式模型主要含有
* AP： 应用程序
* TM: 事务管理器
* RM: 资源管理器(如数据库)
* CRM: 通讯资源管理器（如消息队列）

XA主要就是TM和RM之间的通讯桥梁。

### 2PC
两阶段提交协议（The two-phase commit protocol，2PC）是 XA 用于在全局事务中协调多个资源的机制。
2PC将事务的提交过程分为两个阶段来进行处理
* **准备阶段**
    1. TM向所有RM发送事务内容，询问是否可以提交事务，并等待所有RM答复。
    2. RMs执行事务操作，将操作信息记入事务日志中,但不提交事务
    3. 如RM执行成功，给TM反馈YES；如执行失败，给TM反馈NO。
* **提交阶段**
    1. 所有RM均反馈YES时，即提交事务
    2. 任何一个RM反馈NO时，即中断事务并进行回滚。
### 3PC
3PC是2PC的改进版，其将二阶段提交协议的“准备阶段”一份为二，形成了cancommit，precommit，docommit三个阶段。
* **CanCommit阶段**
    1. TM向RMs发送CanCommit请求。询问是否可以执行事务提交操作。然后开始等待RM的响应。
    2. RMs接到CanCommit请求之后，正常情况下，如果可以顺利执行事务，则返回Yes,并进入预备状态。否则反馈No
* **PreCommit阶段**
    * 正常情况：所有RM均反馈YES时，即提交事务
        1. TM向RM发送PreCommit请求，并进入Prepared阶段。
        2. RM接收到PreCommit请求后，会执行事务操作，并将undo和redo信息记录到事务日志中。
        3. 如果RM成功的执行了事务操作，则返回ACK响应，同时开始等待最终指令。
    * 有任何一个RM向TM发送了No响应，或者等待超时之后，TM都没有接到RM的响应，那么就执行事务的中断。
        1. TM向所有RM发送abort请求。
        2. RM收到来自TM的abort请求之后（或超时之后，仍未收到TM的请求），执行事务的中断。
* **doCommit阶段**
    * 正常
        1. TM接收到RM发送的ACK响应，那么他将从预提交状态进入到提交状态。并向所有RM发送doCommit请求。
        2. RM接收到doCommit请求之后，执行正式的事务提交。**并在完成事务提交之后释放所有事务资源**。
        3. 事务提交完之后，向TM发送Ack响应。
        4. TM接收到所有RM的ack响应之后，完成事务。
    * TM没有接收到RM发送的ACK响应（可能是TM发送的不是ACK响应，也可能响应超时），那么就会执行中断事务。
        1. TM向所有RM发送abort请求。
        2. RM接收到abort请求之后，利用其在阶段二记录的undo信息来执行事务的回滚操作，**并在完成回滚之后释放所有的事务资源**。
        3. RM完成事务回滚之后，向TM发送ACK消息。
        4. TM接收到RM反馈的ACK消息之后，执行事务的中断。

### 小结
2PC的主要问题有三点：
* 两个阶段事务处于阻塞状态
* TM出现问题，一直修复不了的话，RM会一直阻塞
* 由于网络问题或者TM发送一半挂了 ，只有部分RM收到commmit请求，会导致数据的不一致

3PC在二、三阶段引入**超时自动提交事务的机制**和**RM完成事务后释放资源**，有效的防止2PC的前两种情况。
3PC数据一致性的问题还是存在，doCommit时，TM发送abort请求，但由于网络问题，部分RM没有接受到。这样就会出现部分RM执行commit,另外一部分执行abort,从而导致数据不一致的问题。
### refer
[https://www.infoq.cn/article/xa-transactions-handle/](https://www.infoq.cn/article/xa-transactions-handle/)
[https://blog.csdn.net/u013679744/article/details/79188945](https://blog.csdn.net/u013679744/article/details/79188945)

