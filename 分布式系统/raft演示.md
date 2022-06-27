# raft 演示

## 什么是分布式一致性

1. 单节点是一致性的
2. 三个节点则需要一定的算法支持。

## 一、raft概述

1. raft 是实现分布式一致性的协议
2. 一个节点有三种状态：follower、candidate、leader
3. 如果follower一段时间没有听到leader的心跳，就会成为candidate。candidate会向其他节点请求选票，如果candidate收到了半数以上的选票，就会成为leader。这个过程称为**Leader Election**
4. 在存在leader后，系统所有的改变都将通过leader 
5. 例如客户端请求设置5，leader节点收到后，先写入本地log, 之后将set 5命令发送给所有节点；此时leader会等待，直到半数以上节点写入该entry；之后leader提交该entry，并将状态设置为5；之后leader提醒follow节点该entry被提交。这个过程被称为***Log Replication***

## 二、Leader Election

1. follower在一段时间内没有收到来自leader的心跳，将进入election timeout 状态，直到成为candidate。election timeout在150ms-300ms之间随机设置。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       

2. 最先结束timeout的节点成为candidate，并且开始新的**eletion term**。

    例如三个节点a, b, c, 节点c最先结束timeout，那么节点c状态变为candidate, 其term为1，其他节点的term都为0

3. 节点c首先为自己投一票
4. 节点c向其他节点发送投票请求
5. 如果收到请求的节点还没有投票，那么就给节点c投票，并且重置eletion timeout。（投票后term怎么变化）
6. 一旦candidate节点收到半数以上的票数，就会成为Leader
7. leader开始向其他follower节点发送**Append Entries**信息，这些信息以**heartbeat timeout**间隔发送
8. follower节点会响应每一条Append Entries
9. election term将持续直到follower节点停止收到heartbeat, 并且成为candidate
10. 特殊地, 如果有4个节点，并且节点c和节点d同时结束timeout, 并且各自收到了两票，那么由于两个节点都没有收到半数以上的票数，所有节点将再次进行新一轮的election term。 

## Log Replication

1. 当集群选出leader后，leader会将集群所有的改变复制到follower节点上。使用心跳的Append Entries消息来完成。
2.  首先客户端发送一个更改请求到leader
3. leader收到后将该更改请求写入到本地的log中，并在下一次心跳中将该请求发送给follower节点
4. follower节点收到后，也写入本地的log，并且通知leader
5. 当半数以上的节点确认写入log后，leader节点会提交该次更改，并回复客户端，并且同步通知follower节点以提交该次更改，follower节点收到后也会提交该次更改。
6. 即使面临网络分区（会出现脑裂），raft算法也可以保持一致性
7. 假设有5个节点，添加一个网络分区，使得节点a b与节点c d e分隔，那么现在会有两个leader。
8. 假设有两个客户端，一个客户点端向节点b为leader的集群1发送3，因为节点b得不到半数以上的应答，因此该log entry不会被提交；另一个客户端向节点d为leader的集群2发送8，由于可以得到半数以上的应答，该log entry可以被提交。
9. 现在，恢复网络，由于集群1中的term为1, 而 集群2 中的term 为2，因此节点b将看到更高的election term, 并且停止工作。节点a b都将回退他们未提交的entries，并且匹配新leader的log。集群得以保持一致性。