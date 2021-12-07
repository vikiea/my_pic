# Redis的三种模式：主从、哨兵、集群

学习博客：
https://blog.csdn.net/weixin_42236165/article/details/91492946
https://www.pianshen.com/article/1023276171/
http://www.manongjc.com/article/18970.html
https://blog.csdn.net/weixin_42236165/article/details/91492946
https://blog.csdn.net/King__Jack/article/details/104983668
https://jasonhzy.github.io/2017/12/12/redis/

# Redis的三种模式：主从、哨兵、集群

redis的多机数据库实现，主要分为以下三种：

- Redis哨兵（Sentinel）

- Redis复制（主从）

- Redis集群

  ## 一、Redis的主从复制

### 模型

![null](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/11/m_95321de6a052b87a9ad9a93ae9870dc4_r.png)

### –配置

准备两台服务器，分别安装redis ， server1 server2

1. 在server2的redis.conf文件中增加 slaveof server1-ip 6379 、 同时将bindip注释掉，允许所有ip访问。
2. 启动server1,server2
3. 访问server2的redis客户端，输入 INFO replication(查看节点状态信息)
4. 通过在master机器上输入命令，比如set foo bar 、 在slave服务器就能看到该值已经同步过来了。

### 原理

#### –全量复制

　　Redis全量复制一般发生在Slave初始化阶段，这时Slave需要将Master上的所有数据都复制一份。

![null](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/11/m_294dccde7eed672fb2834a0a3b22e6d0_r.png)

完成上面几个步骤后就完成了slave服务器数据初始化的所有操作，savle服务器此时才可以接收来自用户的读请求。

　　master/slave 复制策略是采用乐观复制，也就是说可以容忍在一定时间内master/slave数据的内容是不同的，但是两者的数据会最终同步。具体来说，redis的主从同步过程本身是异步的，意味着master执行完客户端请求的命令后会立即返回结果给客户端，然后以异步的方式把命令同步给slave。这一特征保证启用master/slave 后 master的性能不会受到影响。

　　另一方面，如果在这个数据不一致的窗口期间，master/slave因为网络问题断开连接，而这个时候，master是无法得知某个命令最终同步给了多少个slave数据库。不过redis提供了一个配置项来限制只有数据至少同步给多少个slave的时候，master才是可写的：

min-slaves-to-write 3 表示只有当3个或以上的slave连接到master，master才是可写的

min-slaves-max-lag 10 表示允许slave最长失去连接的时间，如果10秒还没收到slave的响应，则master认为该slave以断开。

#### –增量复制

从redis 2.8开始，就支持主从复制的断点续传，如果主从复制过程中，网络连接断掉了，那么可以接着上次复制的地方，继续复制下去，而不是从头开始复制一份。master node会在内存中创建一个backlog，master和slave都会保存一个replica offffset还有一个master id，offffset就是保存在backlog中的。如果master和slave网络连接断掉了，slave会让master从上次的replica offffset开始继续复制但是如果没有找到对应的offffset，那么就会执行一次全量同步。

#### –无硬盘复制

Redis复制的工作原理基于RDB方式的持久化实现的，也就是master在后台保存RDB快照，slave接收到rdb文件并载入，但是这种方式会存在一些问题：

　　1. 当master禁用RDB时，如果执行了复制初始化操作，Redis依然会生成RDB快照，当master下次启动时执行该RDB文件的恢复，但是因为复制发生的时间点不确定，所以恢复的数据可能是任何时间点的，就会造成数据出现问题。

　　2. 当硬盘性能比较慢的情况下（网络硬盘），那初始化复制过程会对性能产生影响。因此2.8.18以后的版本，Redis引入了无硬盘复制选项，可以不需要通过RDB文件去同步，直接发送数据，通过以下配置来开启该功能：

　　repl-diskless-sync yes（master**在内存中直接创建rdb，然后发送给slave，不会在自己本地落地磁盘了）

通过执行slaveof命令或设置slaveof选项，让一个服务器去复制另一个服务器的数据。被复制的服务器称为：Master主服务；对主服务器进行复制的服务器称为：Slave从服务器。主数据库可以进行读写操作，当写操作导致数据变化时会自动将数据同步给从数据库。而从数据库一般是只读的，并接受主数据库同步过来的数据。一个主数据库可以拥有多个从数据库，而一个从数据库只能拥有一个主数据库。

主从复制问题：当master down，需要手动将一台slave使用slaveof no one提升为master要实现自动，就需要redis哨兵。

### 实现原理步骤：

1. 从服务器向主服务器发送SYNC命令
2. 主服务器收到SYNC命令后，执行BGSAVE命令，在后台生成RDB文件，使用缓冲区记录从现在开始执行的所有的写命令。
3. 当主服务器的BGSAVE命令执行完毕后，主服务器后将BGSAVE命令生成的RDB文件发送给从服务器，从服务器接收并载入这个RDB文件，将自己的数据库状态更新至主服务器执行BGSAVE命令时的数据库状态。
4. 主服务器将记录在缓冲区里面的所有写命令发送给从服务器，从服务器执行这些写命令，将自己的数据库状态更新至主服务器数据库当前所处的状态。

![null](http://8.133.162.206:8181/uploads/redis/images/m_267ba4b3e8c59263d20ea235d7711c4a_r.png)

## 二、Redis的哨兵（Sentinel）

### 模型

哨兵是一个独立的进程,它的作用就是监控Redis系统的运行状况，使用哨兵后的架构图：
![null](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/11/m_b098c2ec9001fd9b85c20a449d137815_r.png)

它的功能包括两个：

1. 监控master和slave是否正常运行。
2. master出现故障时自动将slave数据库升级为master。

为了解决Redis的主从复制的不支持高可用性能，Redis实现了Sentinel哨兵机制解决方案。由一个或多个Sentinel去监听任意多个主服务以及主服务器下的所有从服务器，并在被监视的主服务器进入下线状态时，自动将下线的主服务器属下的某个从服务器升级为新的主服务器，然后由新的主服务器代替已经下线的从服务器，并且Sentinel可以互相监视。

![null](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/11/m_826da0b5c638ea897b0dcb52020dbd8a_r.png)

![null](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/11/m_cc21e4a15bd7762ebe27082fe4aaa8ca_r.png)

当有多个Sentinel，在进行监视和转移主从服务器时，Sentinel之间会自己首先进行选举，选出Sentinel的leader来进行执行任务。

#### –哨兵单点问题

　　为了解决master选举问题，又引出了一个单点问题，也就是哨兵的可用性如何解决，在一个一主多从的Redis系统中，可以使用多个哨兵进行监控任务以保证系统足够稳定。此时哨兵不仅会监控master和slave，同时还会互相监控；这种方式称为哨兵集群，哨兵集群需要解决故障发现、和master决策的协商机制问题

![null](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/11/m_96c8bc11981a6bc4b3361e1b18cbee11_r.png)

#### –sentinel之间的相互感知

　　sentinel节点之间会因为共同监视同一个master从而产生了关联，一个新加入的sentinel节点需要和其他监视相同master节点的sentinel相互感知：

　　1. 需要相互感知的sentinel都向他们共同监视的master节点订阅channel:sentinel:hello。

　　2. 新加入的sentinel节点向这个channel发布一条消息，包含自己本身的信息，这样订阅了这个channel的sentinel就可以发现这个新的sentinel。

　　3. 新加入得sentinel和其他sentinel节点建立长连接。
![null](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/11/m_ce0feed2ec2e79faf11abfe597a3ad69_r.png)

#### –master的故障发现

　　哨兵监控一个系统时，只需要配置监控master即可，哨兵会自动发现所有slave；这时候，我们把master关闭，等待指定时间后（默认是30秒），会自动进行切换。　　 sentinel节点会定期向master节点发送心跳包来判断存活状态，一旦master节点没有正确响应，sentinel会把master设置为“主观不可用状态”，然后它会把“主观不可用”发送给其他所有的sentinel节点去确认，当确认的sentinel节点数大于>quorum时，则会认为master是“客观不可用”，接着就开始进入选举新的master流程；但是这里又会遇到一个问题，就是sentinel中，本身是一个集群，如果多个节点同时发现master节点达到客观不可用状态，那谁来决策选择哪个节点作为master呢？这个时候就需要从sentinel集群中选择一个leader来做决策。而这里用到了一致性算法Raft算法、它和Paxos算法类似，都是分布式一致性算法。但是它比Paxos算法要更容易理解；Raft和Paxos算法一样，也是基于投票算法，只要保证过半数节点通过提议即可。

#### 配置实现：

　　在其中任意一台服务器上创建一个sentinel.conf文件，文件内容如下：

　　port 6040

　　sentinel monitor mymaster 192.168.11.131 6379 1（sentinel monitor name ip port quorum 其中name表示要监控的master的名字，这个名字是自己定义。ip和port表示master的ip和端口号。 最后一个1表示最低通过票数，也就是说至少需要几个哨兵节点统一才可以。）

　　sentinel down-after-milliseconds mymaster 5000（表示如果5s内mymaster没响应，就认为SDOWN）

　　sentinel failover-timeout mymaster 15000（表示如果15秒后,mysater仍没活过来，则启动failover，从剩下的slave中选一个升级为master）

#### 　–两种方式启动哨兵　　–两种方式启动哨兵

　　redis-sentinel sentinel.conf

　　redis-server /path/to/sentinel.conf –sentinel

#### –实操

![null](http://8.133.162.206:8181/uploads/redis/images/m_5b115cb853b1a330811cc03b20987a9a_r.png)
+sdown：表示哨兵主管认为master已经停止服务了。

+odown：表示哨兵客观认为master停止服务了。关于主观和客观，后面会给大家讲解。接着哨兵开始进行故障恢复，挑选一个slave升级为master。

+try-failover：表示哨兵开始进行故障恢复。

+failover-end：表示哨兵完成故障恢复。

+slave：表示列出新的master和slave服务器，我们仍然可以看到已经停掉的master，哨兵并没有清楚已停止的服务的实例，这是因为已经停止的服务器有可能会在某个时间进行恢复，恢复以后会以slave角色加入到整个集群中。

## 三、Redis集群

集群是Redis提供的分布式数据库方案，集群通过分片来进行数据共享，并提供复制和故障转移功能。一个Redis集群通常由多个节点组成；最初，每个节点都是独立的，需要将独立的节点连接起来才能形成可工作的集群。

Cluster Nodes命令和Cluster Meet命令，添加和连接节点形成集群。

![null](http://8.133.162.206:8181/uploads/redis/images/m_d915b2b52aefd6d0b8f2f6b695da53f1_r.png)
Redis中的集群分为主节点和从节点。其中主节点用于处理槽；而从节点用于复制某个主节点，并在被复制的主节点下线时，代替下线的主节点继续处理命令请求。

![null](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/11/m_920297eaca220d1b5270af25819123ab_r.png)