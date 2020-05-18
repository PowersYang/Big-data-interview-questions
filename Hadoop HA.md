## 背景

在 Hadoop 之前的版本中存在单点故障的问题，就是说每个集群中只有一个 NameNode 提供服务，如果发生意外宕机或者对机器进行维护（如更换硬盘）的时候，整个集群就不能正常工作，基于以上原因，提出了 HA（Hadoop 高可用）的概念。

Hadoop 官方网站上提供了两种高可用方案：QJM 和 NFS，企业中常用的为 QJM 方式。

## 实现原理
在典型的 HA 群集中，将两个或更多单独的计算机配置为 NameNode。在任何时间点，一个 NameNode 都恰好处于 Active 状态，而其他 NameNode 处于 Standby 状态。Active NameNode负责集群中的所有客户端操作，而 Standby 则仅充当工作程序，并保持足够的状态以在必要时提供快速故障转移。

为了 Standby 节点保持其状态与 Active 节点同步，两个节点都与一组称为“ JournalNodes”（JN）的单独守护程序进行通信。当 Active 节点执行任何名称空间修改时，它会持久地将修改记录记录到这些 JN 的大多数中。Standby 节点能够从 JN 中读取编辑内容，并不断监视它们是否有编辑日志的更改。当“Standby节点”看到编辑内容时，会将其应用到自己的名称空间。发生故障转移时，Standby 服务器将确保在将自身提升为 Active 状态之前，已从 JournalNode 读取所有编辑内容。这样可以确保在发生故障转移之前，所有数据已经同步。

## 自动故障转移
自动故障转移为HDFS部署添加了两个新组件：ZooKeeper 和 ZKFailoverController 进程（缩写为ZKFC）。

在故障转移过程中，Zookeeper 负责：

- 故障检测：群集中的每个 NameNode 都在ZooKeeper中维护一个持久会话。如果计算机崩溃，则ZooKeeper会话将终止，通知其它 NameNode 应该触发故障转移。

- Active NameNode 选举：ZooKeeper 提供了一种简单的机制来专门选举一个节点为 Active 的节点。如果当前 Active NameNode 崩溃，则另一个节点可能会在 ZooKeeper 中得到特殊的排它锁，指示它应成为下一个 Active NameNode。

ZKFC 是 Zookeeper 的一个客户端，主要负责：

- 健康监测：ZKFC使用一个健康检查命令定期地ping与之在相同主机的NameNode，只要该NameNode及时地回复健康状态，ZKFC认为该节点是健康的。如果该节点崩溃，冻结或进入不健康状态，健康监测器标识该节点为非健康的。

- ZooKeeper会话管理：当本地NameNode是健康的，ZKFC保持一个在ZooKeeper中打开的会话。如果本地NameNode处于active状态，ZKFC也保持一个特殊的znode锁，该锁使用了ZooKeeper对短暂节点的支持，如果会话终止，锁节点将自动删除。

- 基于ZooKeeper的选择：如果本地NameNode是健康的，且ZKFC发现没有其它的节点当前持有znode锁，它将为自己获取该锁。如果成功，则它已经赢得了选择，并负责运行故障转移进程以使它的本地NameNode为Active。故障转移进程与前面描述的手动故障转移相似，首先如果必要保护之前的现役NameNode，然后本地NameNode转换为Active状态。

## 如何保证同时只有一个 NameNode 往 JournalNode 写数据

处于 Active 的 NameNode 每次和 JournalNode 通信都会携带一个整形的 Epoch Number，该数值是唯一的。而且是有严格顺序的，每切换一次 Active NameNode，其值都会自动加1。当 Active NameNode 向 JournalNode 发送消息的时候，JournalNode 会将 Active NameNode 携带的 Epoch Number 和自己保存的进行对比，如果该数值比自己保存的小，则拒绝写入请求。