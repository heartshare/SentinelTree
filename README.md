# SentinelTree
阿里Sentinel流量控制技术研究


<pre>
Redis Sentinel

      Redis-Sentinel是Redis官方推荐的高可用HA方案，当用Redis做Master-Slave高可用方案时，
      加入Master宕机了，Redis本身都没有实现自动进行主备切换，而Redis-Sentinel本身也是一个
      独立运行运行的进程，它能监控多个Master-Slave集群，发现Master宕机后进行自动切换。
</pre>

<pre>
Sentinel概述

      分布式系统的流量哨兵，以流量为切入点，对比Redis的sentinel哨兵模式可以得出sentinel
      在微服务中的作用是对流量进行监控和管理，例如流量的控制，熔断降级，系统负载保护等。

      由Sentinel的源码可以看出，Sentinel在对流量进行管控的时候是通过责任链的模式来处理的。
      在Restful中，将系统的一切定义为资源，sentinel在此处也借鉴了此种思想，将需要流量控制
      的一切当做资源，然后定义一系列的规则来对资源进行处理。

	      1，不时地监控redis是否按照预期良好地运行;
	      2，如果发现某个redis节点运行出现状况，能够通知另外一个进程(例如它的客户端);
	      3，能够进行自动切换。当一个master节点不可用时，能够选举出master的多个slave(如果有
             超过一个slave的话)中的一个来作为新的master,其它的slave节点会将它所追随的
             master的地址改为被提升为master的slave的新地址。

      Master-Slave原理：
          ①从数据库向主数据库发送sync命令。
          ②主数据库接收sync命令后，执行BGSAVE命令（保存快照），创建一个RDB文件，在创建RDB
           文件期间的命令将保存在缓冲区中。
          ③当主数据库执行完BGSAVE时，会向从数据库发送RDB文件，而从数据库会接收并载入该文件。
          ④主数据库将缓冲区的所有写命令发给从服务器执行。
          ⑤以上处理完之后，之后主数据库每执行一个写命令，都会将被执行的写命令发送给从数据库。

      Redis目前的复制是异步的，只保证最终一致性，而不是强一致性（主从数据库的更新还是分先后，
      先主后从）。要是一致性要求高的应用，目前还是读写都在主库上去。

      sentinel是一个"监视器"，根据被监视实例的身份和状态来判断该执行何种操作。通过给定的配置文
      件来发现主服务器的，再通过向主服务器发送的info信息来发现该主服务器的从服务器。Sentinel 
      实际上就是一个运行在 Sentienl 模式下的 Redis 服务器,所以我们同样可以使用以下命令来启动
      一个 Sentinel实例

      Sentinel集群：

            1：即使有一些sentinel进程宕掉了，依然可以进行redis集群的主备切换；
            2：如果只有一个sentinel进程，如果这个进程运行出错，或者是网络堵塞，那么将无法实
               现redis集群的主备切换（单点问题）;
            3：如果有多个sentinel，redis的客户端可以随意地连接任意一个sentinel来获得关于
               redis集群中的信息。

      Sentinel集群工作原理：
            ①sentinel集群通过给定的配置文件发现master，启动时会监控master。通过向master发
             送info信息获得该服务器下面的所有从服务器。
            ②sentinel集群通过命令连接向被监视的主从服务器发送hello信息(每秒一次)，该信息包
             括sentinel本身的ip、端口、id等内容，以此来向其他sentinel宣告自己的存在。
            ③sentinel集群通过订阅连接接收其他sentinel发送的hello信息，以此来发现监视同一个
             主服务器的其他sentinel；集群之间会互相创建命令连接用于通信，因为已经有主从服务
             器作为发送和接收hello信息的中介，sentinel之间不会创建订阅连接。
            ④sentinel集群使用ping命令来检测实例的状态，如果在指定的时间内（down-after-
             milliseconds）没有回复或则返回错误的回复，那么该实例被判为下线。 
            ⑤当failover主备切换被触发后，failover并不会马上进行，还需要sentinel中的大多数
             sentinel授权后才可以进行failover，即进行failover的sentinel会去获得指定
             quorum个的sentinel的授权，成功后进入ODOWN状态。如在5个sentinel中配置了2个
             quorum，等到2个sentinel认为master死了就执行failover。
            ⑥sentinel向选为master的slave发送SLAVEOF NO ONE命令，选择slave的条件是
             sentinel首先会根据slaves的优先级来进行排序，优先级越小排名越靠前。如果优先级相
             同，则查看复制的下标，哪个从master接收的复制数据多，哪个就靠前。如果优先级和下标
             都相同，就选择进程ID较小的。
            ⑦sentinel被授权后，它将会获得宕掉的master的一份最新配置版本号(config-epoch)，
             当failover执行结束以后，这个版本号将会被用于最新的配置，通过广播形式通知其它
             sentinel，其它的sentinel则更新对应master的配置。


            ①到③是自动发现机制:
               1） 以10秒一次的频率，向被监视的master发送info命令，根据回复获取master当前信息。
               2）以1秒一次的频率，向所有redis服务器、包含sentinel在内发送PING命令，通过回
                 复判断服务器是否在线。
               3）以2秒一次的频率，通过向所有被监视的master，slave服务器发送当前sentinel，
                  master信息的消息。
            ④是检测机制，⑤和⑥是failover机制，⑦是更新配置机制。
</pre>

###alibaba sentinel
![](https://i.imgur.com/IugsAtp.png)

削峰填谷
![](https://i.imgur.com/ajQaqWn.png)

<pre>
限流：

      当QPS超出某个设定的阈值，系统可以通过拒绝，冷启动，匀速器三种方式来应对，从而起到流量
      控制的作用。
</pre>

<pre>
熔断降级：

      Sentinel通过并发线程数进行限制和响应时间对资源进行降级两种手段来对服务进行熔断或降级。
</pre>

<pre>
塑型：

      通常遇到的流量具有随机性，不规则，不受控的特点，但系统的处理能力往往是有限的，我们需
      要根据系统的处理能力对流量进行塑性，即规则化，从而根据我们的需要来处理流量，
</pre>

<pre>
系统负载均衡
</pre>

<pre>
轻巧
</pre>

###Alibaba Sentinel使用方法
![](https://i.imgur.com/hvOnaJO.png)

![](https://i.imgur.com/tLAA2DE.png)
在Sentinel中，对那些被阻塞的请求，都是用catch到BlockException异常的方式进行处理的。

定义流量规则
![](https://i.imgur.com/4Z5Lpjy.png)

流量规则加载
![](https://i.imgur.com/4uPeNz7.png)

###例子：
![](https://i.imgur.com/pj8ZDsD.png)

![](https://i.imgur.com/hXpkEpU.png)