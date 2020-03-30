以下内容搜集整理于网络，并会不定期更新

## **HDFS 组成架构**
![](http://www.ysir308.com/wp-content/uploads/2020/03/59fbe6f7e89d979b54336649461c03bc.png)

1. Client（客户端）：访问 HDFS 的程序或者 HDFS 的 Shell 操作都可以认为是 HDFS 的客户端。
（1）负责文件切分。文件上传 HDFS 的时候，Client 将文件切分成一个个 Block，然后进行上传。
（2）与 NameNode 进行交互，获取文件的位置信息。
（3）与 DataNode 交互，读取或者写入数据。
（4）Client 提供一些命令来管理 HDFS，比如 NameNode 的格式化。
（5）Client 可以通过一些命令来访问 HDFS，对文件进行增删查改操作。

2. NameNode
（1）管理 HDFS 命名空间。
（2）配置副本策略。
（3）管理数据块的映射信息。
（4）处理 Client 的读写请求。

3. SecondaryNameNode
（1）辅助 NameNode 进行工作，分担其工作量，定期合并 Fsimage 和 Edits，并推送给 NameNode。
（2）在紧急情况下，可以辅助恢复 NameNode。
**注意：SecondaryNameNode 并不是 NameNode 的热备，只是辅助工作，当 NameNode 挂掉的时候不能马上替换 NameNode 提供服务，只是辅助其工作。**

4. DataNode
（1）存储实际的数据块。
（2）执行数据块的读写操作。

## **为什么文件块不能设置太小也不能太大**
文件块太小，元数据信息太多，会增加寻址时间；文件块太大，比较考验磁盘的传输速率，导致在程序处理过程中会非常慢。

文件块大小主要还是取决于磁盘传输速率。

## **HDFS 小文件优化方法**
1、Hadoop 自带的解决方案
（1）Hadoop Archive：Hadoop Archive 是一个高效地将小文件放入HDFS块中的文件存档工具，它能够将多个小文件打包成一个HAR文件，这样在减少 NameNode 内存使用的同时也不影响对外提供服务。
（2）Sequence file：Sequence file 由一系列的二进制 key/value 组成，如果为 key 小文件名，value 为文件内容，则可以将大批小文件合并成一个大文件。
（3）CombineFileInputFormat：CombineFileInputFormat 是一种新的 Inputformat，用于将多个文件合并成一个单独的split，另外，它会考虑数据的存储位置。

2、从业务逻辑方面解决


## **HDFS 数据写流程**
 ![](http://www.ysir308.com/wp-content/uploads/2020/03/a46ab5816ebc68600f8b888b685132d4.png)
 
 1. 客户端通过Distributed FileSystem模块向NameNode请求上传文件，NameNode检查目标文件是否已存在，父目录是否存在。
 2. NameNode返回是否可以上传。
 3. 客户端请求第一个 Block上传到哪几个DataNode服务器上。
 4. NameNode返回3个DataNode节点，分别为dn1、dn2、dn3。
 5. 客户端通过FSDataOutputStream模块请求dn1上传数据，dn1收到请求会继续调用dn2，然后dn2调用dn3，将这个通信管道建立完成。
 6. dn1、dn2、dn3逐级应答客户端。
 7. 客户端开始往dn1上传第一个Block（先从磁盘读取数据放到一个本地内存缓存），以Packet为单位，dn1收到一个Packet就会传给dn2，dn2传给dn3；dn1每传一个packet会放入一个应答队列等待应答。
 8. 当一个Block传输完成之后，客户端再次请求NameNode上传第二个Block的服务器。（重复执行3-7步）。

## **HDFS 数据读流程**
![](http://www.ysir308.com/wp-content/uploads/2020/03/d6d2ec85486809fc5e1ab785b67f9720.png)

1. 客户端通过Distributed FileSystem向NameNode请求下载文件，NameNode通过查询元数据，找到文件块所在的DataNode地址。
2. 挑选一台DataNode（就近原则，然后随机）服务器，请求读取数据。
3. DataNode开始传输数据给客户端（从磁盘里面读取数据输入流，以Packet为单位来做校验）。
4. 客户端以Packet为单位接收，先在本地缓存，然后写入目标文件。

## **NameNode 和 SecondaryNameNode 工作机制**
SecondaryNamenode，专门用于 Fsimage 和 Edits 的合并，辅助 NameNode 进行工作。
其中 Fsimage 是 NameNode 内存中的元数据序列化后形成的文件。Edits 是记录客户端更新元数据信息的每一步操作，只能追加。

![](http://www.ysir308.com/wp-content/uploads/2020/03/948be95e3c5a02fa3fb4b6196609d277.png)

1. 第一阶段：NameNode启动
（1）第一次启动NameNode格式化后，创建Fsimage和Edits文件。如果不是第一次启动，直接加载编辑日志和镜像文件到内存。
（2）客户端对元数据进行增删改的请求。
（3）NameNode记录操作日志，更新滚动日志。
（4）NameNode在内存中对数据进行增删改。

2. 第二阶段：Secondary NameNode工作
（1）Secondary NameNode询问NameNode是否需要CheckPoint。直接带回NameNode是否检查结果。
（2）Secondary NameNode请求执行CheckPoint。
（3）NameNode滚动正在写的Edits日志。
（4）将滚动前的编辑日志和镜像文件拷贝到Secondary NameNode。
（5）Secondary NameNode加载编辑日志和镜像文件到内存，并合并。
（6）生成新的镜像文件fsimage.chkpoint。
（7）拷贝fsimage.chkpoint到NameNode。
（8）NameNode将fsimage.chkpoint重新命名成fsimage。

关于 Checkpoint 的条件：
（1）通常情况下，SecondaryNameNode 每隔一小时执行一次。在 hdfs-default.xml 文件中配置。
```xml
<property>
	<name>dfs.namenode.checkpoint.period</name>
	<value>3600</value>
</property>
```
（2）每隔一分钟检查一次客户端操作次数，当操作次数达到100万时，执行一次。
```xml
<property>
	<name>dfs.namenode.checkpoint.txns</name>
	<value>1000000</value>
	<description>操作动作次数</description>
</property>

<property>
	<name>dfs.namenode.checkpoint.check.period</name>
	<value>60</value>
	<description> 1分钟检查一次操作次数</description>
</property >
```


## **机架感知**

HDFS采用一种称为机架感知的策略来改进数据的可靠性、可用性和网络带宽的利用率。

在通常情况下，当副本数量为3时，HDFS的放置策略是将一个副本放置在本地机架中的一个节点上，将另一个副本放置在本地机架中的另一个节点上，最后一个副本放置在不同机架中的另一个节点上。


## **DataNode 工作机制**

![](http://www.ysir308.com/wp-content/uploads/2020/03/e5018c7bc1a36d38091b0760e4fd2de2.png)

1. 一个数据块在DataNode上以文件形式存储在磁盘上，包括两个文件，一个是数据本身，一个是元数据包括数据块的长度，块数据的校验和，以及时间戳。
2. DataNode启动后向NameNode注册，通过后，周期性（1小时）的向NameNode上报所有的块信息。
3. 心跳是每3秒一次，心跳返回结果带有NameNode给该DataNode的命令如复制块数据到另一台机器，或删除某个数据块。如果超过10分钟没有收到某个DataNode的心跳，则认为该节点不可用。
4. 集群运行中可以安全加入和退出一些机器。

## **你认为 HDFS 有哪些需要改进的地方**
1. 不适合低延迟数据访问。
2. 不适合对大量小文件进行存储。因为小文件太多会占用 NameNode 大量内存来存储小文件元数据，增加寻址时间，违反了 HDFS 设计目标。
3. 不支持并发写入金额文件的随机修改操作（只支持追加操作）。


## **如何增加新节点（节点动态上线）**
（1）从其它节点克隆一台虚拟机。
（2）修改IP和主机名称。
（3）删除原来 HDFS 文件留存的文件，根据配置文件删除 data 和 log 文件。
（4）直接启动 DataNode 和 NodeManager 服务。
（5）执行 ./start-balancer.sh 命令实现集群数据平衡。
上述 1、2、3 步骤省略了在 NameNode 和 DataNode 上配置对方的主机名、SSH免密登录、安装JDK、修改配置文件等繁琐操作。

## **如何删除旧节点（节点动态下线）**
节点动态下线有两种方式：白名单和黑名单操作。
1、白名单：白名单即 Hadoop 根目录下面的 etc/hadoop/dfs.hosts 文件（如果没有，请创建）。添加到白名单的主机节点都允许访问 NameNode，不在白名单的主机节点，都不允许访问 NameNode。
白名单中存储允许访问 NameNode 的主机名称，每行一个。
具体下线步骤：
（1）从白名单剔除需要下线的节点名称。
（2）在 NameNode 的 hdfs-site.xml 文件中配置白名单属性：
```xml
<property>
	<name>dfs.hosts</name>
	<value>/opt/module/hadoop-2.7.2/etc/hadoop/dfs.hosts</value>
</property>
```
（3）将 hdfs-site.xml 同步到其它节点。
（4）执行 hdfs dfsadmin -refreshNodes 刷新 NameNode。
（5）执行 yarn rmadmin -refreshNodes 更新 ResourceManager。
（6）执行 ./start-balancer.sh 命令实现集群数据平衡

2、黑名单：黑名单即 Hadoop 根目录下面的 etc/hadoop/dfs.hosts.exclude 文件（如果没有，请创建）。黑名单中的主机会被强制退出。
具体下线步骤：
（1）在 dfs.hosts.exclude 文件中添加需要下线的主机名称。
（2）在 hdfs-site.xml 文件中增加黑名单属性：
```xml
<property>
	<name>dfs.hosts.exclude</name>
	<value>/opt/module/hadoop-2.7.2/etc/hadoop/dfs.hosts.exclude</value>
</property>
```
（3）执行 hdfs dfsadmin -refreshNodes 刷新 NameNode。
（4）执行 yarn rmadmin -refreshNodes 更新 ResourceManager。
（5）在 Web 浏览器中查看退役节点状态，如果状态为 为decommission in progress，说明此时正在执行退役操作（将其存储的文件复制到其它节点）。
（6）当退役节点状态为 decommissioned 时，说明文件复制完成。
（7）执行 ./start-balancer.sh 命令实现集群数据平衡

## **关于集群安全模式**
在 NameNode 启动时，首先会进行合并 Fsimage 镜像文件和 Edits 日志文件，合并成功后会创建一个新的 Fsimage 文件和 Edits 文件。在这个过程期间。整个 NameNode 处于安全模式，在此期间，DataNode 会向 NameNode 发送最新的文件块列表信息。在安全模式期间，整个集群中的文件是只读的。

何时退出安全模式？
> 如果整个文件系统证据哦给你99.9%的文件块满足最小副本级别，NameNode 会在30秒之后推出安全模式。

在集群正常运行期间也可以进入安全模式，相关命令如下：

- 查看安全模式状态 `bin/hdfs dfsadmin -safemode get`
- 进入安全模式状态 `bin/hdfs dfsadmin -safemode enter`
- 离开安全模式状态 `bin/hdfs dfsadmin -safemode leave`
- 等待安全模式状态 `bin/hdfs dfsadmin -safemode wait`

## **Hadoop 高可用**

请参见 [Hadoop HA](http://www.ysir308.com/archives/3035 "Hadoop HA")
