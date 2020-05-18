[toc]

## 1、简要描述搭建Apache Hadoop集群步骤
1. 准备虚拟机
2. 修改静态IP以及主机名
3. 配置SSH免密登录
4. 关闭防火墙
5. 安装JDK
6. 安装Hadoop并配置相关核心文件
7. 格式化 （hdfs namenode -format）

## 2、节点动态上线
1. 从其它节点克隆一台虚拟机。
2. 修改IP和主机名称。
3. 删除原来 HDFS 文件留存的文件，根据配置文件删除 data 和 log 文件。
4. 直接启动 DataNode 和 NodeManager 服务。
5. 执行 ./start-balancer.sh 命令实现集群数据平衡。

上述 1、2、3 步骤省略了在 NameNode 和 DataNode 上配置对方的主机名、SSH免密登录、安装JDK、修改配置文件等繁琐操作。

## 3、节点动态下线

节点动态下线有两种方式：白名单和黑名单。

### 3.1 白名单

白名单即 Hadoop 根目录下面的 etc/hadoop/dfs.hosts 文件（如果没有，请创建）。添加到白名单的主机节点都允许访问 NameNode，不在白名单的主机节点，都不允许访问 NameNode。

白名单中存储允许访问 NameNode 的主机名称，每行一个。
具体下线步骤：
1. 从白名单剔除需要下线的节点名称。
2. 在 NameNode 的 hdfs-site.xml 文件中配置白名单属性：
```xml
<property>
	<name>dfs.hosts</name>
	<value>/opt/module/hadoop-2.7.2/etc/hadoop/dfs.hosts</value>
</property>
```
3. 将 hdfs-site.xml 同步到其它节点。
4. 执行 hdfs dfsadmin -refreshNodes 刷新 NameNode。
5. 执行 yarn rmadmin -refreshNodes 更新 ResourceManager。
6. 执行 ./start-balancer.sh 命令实现集群数据平衡

### 3.2 黑名单

黑名单即 Hadoop 根目录下面的 etc/hadoop/dfs.hosts.exclude 文件（如果没有，请创建）。黑名单中的主机会被强制退出。
具体下线步骤：

1. 在 dfs.hosts.exclude 文件中添加需要下线的主机名称。
2. 在 hdfs-site.xml 文件中增加黑名单属性：
```xml
<property>
	<name>dfs.hosts.exclude</name>
	<value>/opt/module/hadoop-2.7.2/etc/hadoop/dfs.hosts.exclude</value>
</property>
```
3. 执行 hdfs dfsadmin -refreshNodes 刷新 NameNode。
4. 执行 yarn rmadmin -refreshNodes 更新 ResourceManager。
5. 在 Web 浏览器中查看退役节点状态，如果状态为 为decommission in progress，说明此时正在执行退役操作（将其存储的文件复制到其它节点）。
6. 当退役节点状态为 decommissioned 时，说明文件复制完成。
7. 执行 ./start-balancer.sh 命令实现集群数据平衡

## 4、集群安全模式

在 NameNode 启动时，首先会进行合并 Fsimage 镜像文件和 Edits 日志文件，合并成功后会创建一个新的 Fsimage 文件和 Edits 文件。在这个过程期间。整个 NameNode 处于安全模式，在此期间，DataNode 会向 NameNode 发送最新的文件块列表信息。在安全模式期间，整个集群中的文件是只读的。

何时退出安全模式？
> 如果整个文件系统证据哦给你99.9%的文件块满足最小副本级别，NameNode 会在30秒之后推出安全模式。

在集群正常运行期间也可以进入安全模式，相关命令如下：

- 查看安全模式状态 `bin/hdfs dfsadmin -safemode get`
- 进入安全模式状态 `bin/hdfs dfsadmin -safemode enter`
- 离开安全模式状态 `bin/hdfs dfsadmin -safemode leave`
- 等待安全模式状态 `bin/hdfs dfsadmin -safemode wait`



