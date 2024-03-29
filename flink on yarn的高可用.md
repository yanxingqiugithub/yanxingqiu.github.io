# Yarn集群高可用
在运行高可用性YARN群集时，我们不会运行多个JobManager（ApplicationMaster）实例，而只会运行一个，由YARN在失败时重新启动。确切的行为取决于您使用的特定YARN版本。

## 配置
最大应用程序主要尝试次数（yarn-site.xml）
您必须在yarn-site.xml中为您的yarn设置配置Application Master的最大尝试次数：


```
<property>
  <name>yarn.resourcemanager.am.max-attempts</name>
  <value>4</value>
  <description>
    The maximum number of application master execution attempts.
  </description>
</property>
```

当前YARN版本的默认值为2（表示允许单个JobManager失败）。

申请重试(flink-conf.yaml)
除HA配置（参见上文）外，您还必须配置最大尝试次数conf/flink-conf.yaml：

yarn.application-attempts: 10
这意味着在Yarn应用程序失败之前，应用程序可以重新启动9次(9次重试+ 1次初始尝试)。如果Yarn需要，Yarn可以执行额外的重启操作:抢占、节点硬件故障或重新引导，或NodeManager重新同步。这些重启不计入yarn.application-attempts，请参阅Jian Fang的博客文章。需要注意的是，yarn.resourcemanager.am.max-attempts是应用程序重新启动的上限。因此，Flink中设置的应用程序尝试次数不能超过开始使用Yarn的Yarn集群设置。

容器关闭行为
YARN 2.3.0 <版本<2.4.0。如果应用程序主机失败，则重新启动所有容器。

YARN 2.4.0 <版本<2.6.0。TaskManager容器在应用程序主故障期间??保持活动状态。具有以下优点：启动时间更快并且用户不必再等待再次获得容器资源。

YARN 2.6.0 <= version：将尝试失败有效性间隔设置为Flinks的Akka超时值。尝试失败有效性间隔表示只有在系统在一个间隔期间看到最大应用程序尝试次数后才会终止应用程序。这避免了持久的工作会耗尽它的应用程序尝试。

注意：Hadoop YARN 2.4.0有一个主要的bug(修复在2.5.0中)，它阻止容器从重新启动的Application Master/JobManager容器重新启动。有关详细信息，请参阅FLINK-4142。我们建议至少在YARN上使用Hadoop 2.5.0进行高可用性设置。

示例: 高可用YARN Session
1.在conf/flink-conf.yaml中配置高可用模式和ZooKeeper quorum:


```
high-availability: zookeeper
high-availability.zookeeper.quorum: localhost:2181
high-availability.storageDir: hdfs:///flink/recovery
high-availability.zookeeper.path.root: /flink
yarn.application-attempts: 10
```

2.在conf/zoo.cfg中配置zookeeper servers(目前每台机器只能运行一台ZooKeeper服务器):


```
server.0=localhost:2888:3888
```

3.启动ZooKeeper quorum:


```
$ bin/start-zookeeper-quorum.sh
Starting zookeeper daemon on host localhost.
```

4.启动一个高可用集群


```
$ bin/yarn-session.sh -n 2
```

### Zookeeper安全配置
如果ZooKeeper使用Kerberos以安全模式运行，则可以flink-conf.yaml根据需要覆盖以下配置：

zookeeper.sasl.service-name: zookeeper     # default is "zookeeper". If the ZooKeeper quorum is configured
                                           # with a different service name then it can be supplied here.
zookeeper.sasl.login-context-name: Client  # default is "Client". The value needs to match one of the values
                                           # configured in "security.kerberos.login.contexts".
有关Kerberos安全性的Flink配置的更多信息，请参阅此处。您还可以在此处找到有关Flink内部如何设置基于Kerberos的安全性的更多详细信息。

引导Zookeeper
如果您没有独立的已运行的Zookeeper，则可以使用Flink附带的帮助程序脚本。

在conf/zoo.cfg中有一个ZooKeeper配置模板。您可以将主机配置为使用server.X条目运行ZooKeeper ，其中X是每个服务器的唯一ID：


```
server.X=addressX:peerPort:leaderPort
[...]
server.Y=addressY:peerPort:leaderPort
bin/start-zookeeper-quorum.sh
```
脚本将在每个配置的主机上启动ZooKeeper服务器。启动的进程通过Flink包装器启动ZooKeeper服务器，该包装器从中读取配置conf/zoo.cfg并确保为方便起见设置一些必需的配置值。在生产设置中，建议构建单独的zookeeper集群管理。

参考文献：https://flink-docs-cn.gitbook.io/project/06-bu-shu-cao-zuo/gao-ke-yong-ha
