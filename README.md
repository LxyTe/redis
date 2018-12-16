
#### 主从复制

     1、redis的复制功能是支持多个数据库之间的数据同步。一类是主数据库（master）一类是从数据库（slave），主数据库可以进行读写操作，当发生写操作的时候自动将数据同步到从数据库，而从数据库一般是只读的，并接收主数据库同步过来的数据，一个主数据库可以有多个从数据库，而一个从数据库只能有一个主数据库。
     2、通过redis的复制功能可以很好的实现数据库的读写分离，提高服务器的负载能力。主数据库主要进行写操作，而从数据库负责读操作。         
    
   ![图图](https://github.com/LxyTe/redis/blob/master/%E4%B8%BB%E4%BB%8E%E5%90%8C%E6%AD%A5.png) 
     
    原理
     1：当一个从数据库启动时，会向主数据库发送sync命令，
     2：主数据库接收到sync命令后会开始在后台保存快照（执行rdb操作），并将保存期间接收到的命令缓存起来
     3：当快照完成后，redis会将快照文件和所有缓存的命令发送给从数据库。
     4：从数据库收到后，会载入快照文件并执行收到的缓存的命令。
     
    配置方法 修改redis.conf
    
    修改从库，redis中的redis.conf
    slaveof ip(主库ip) 6379 主库端口号
    
    masterauth  *****  主redis服务器配置的密码。没有可不写
    
 #### 哨兵机制
    
    Redis的哨兵(sentinel) 系统用于管理多个 Redis 服务器,该系统执行以下三个任务:
        监控(Monitoring): 哨兵(sentinel) 会不断地检查你的Master和Slave是否运作正常。
        提醒(Notification):当被监控的某个 Redis出现问题时, 哨兵(sentinel) 可以通过 API 向管理员或者其他应用程序发送通知。
        
        自动故障迁移(Automatic failover):当一个Master不能正常工作时，哨兵(sentinel) 会开始一次自动故障迁移操作,它会将失效Master的其中一个Slave升级为新的Master, 并让失效Master的其他Slave改为复制新的Master; 当客户端试图连接失效的Master时,集群也会向客户端返回新Master的地址,使得集群可以使用Master代替失效Master。
       哨兵(sentinel) 是一个分布式系统,你可以在一个架构中运行多个哨兵(sentinel) 进程,这些进程使用流言协议(gossipprotocols)来接收关于Master是否下线的信息,并使用投票协议(agreement protocols)来决定是否执行自动故障迁移,以及选择哪个Slave作为新的Master. 每个哨兵(sentinel) 会向其它哨兵(sentinel)、master、slave定时发送消息,以确认对方是否”活”着,如果发现对方在指定时间(可配置)内未回应,则暂时认为对方已挂(所谓的”主观认为宕机” Subjective Down,简称sdown).
      若“哨兵群”中的多数sentinel,都报告某一master没响应,系统才认为该master"彻底死亡"(即:客观上的真正down机,Objective Down,简称odown),通过一定的vote算法,从剩下的slave节点中,选一台提升为master,然后自动修改相关配置.虽然哨兵(sentinel) 释出为一个单独的可执行文件 redis-sentinel ,但实际上它只是一个运行在特殊模式下的 Redis 服务器，你可以在启动一个普通 Redis 服务器时通过给定 --sentinel 选项来启动哨兵(sentinel).
     哨兵(sentinel) 的一些设计思路和zookeeper非常类似
     单个哨兵(sentinel)
      
 ![图图](https://github.com/LxyTe/redis/blob/master/%E5%93%A8%E5%85%B5%E6%9C%BA%E5%88%B6.png)
 
      配置哨兵机制的参数(如果没有sentinel.conf文件，可以先进行创建)

    # 这个是Redis6379配置内容，其他文件同理新增然后改一下端口即可，26380，和 26381。
    #当前Sentinel服务运行的端口
    port 26379
    # 哨兵监听的主服务器 ,#主节点 名称 IP 端口号 选举次数
    sentinel monitor mymaster 127.0.0.1 6379 1
    # 3s内mymaster无响应，则认为mymaster宕机了
    sentinel down-after-milliseconds mymaster 3000
    #如果10秒后,mysater仍没启动过来，则启动failover
    sentinel failover-timeout mymaster 10000
    # 执行故障转移时， 最多有1个从服务器同时对新的主服务器进行同步
    sentinel parallel-syncs mymaster 2
   启动参数为(windows) redis-server.exe(sh) sentinel.conf --sentinel
    
     注意点 哨兵搭建最少3台，这样才可以正确的判断当前主机是否真正失去响应，如果在指定的时间内，原主机没有连接，那么就会判定为原主服务以及挂掉，这时候会采取选举投票机制，重新选取一个主服务B。选取成功后，B就是最新的主服务，其它为从服务。就算原有的主服务恢复了连接，那么它也是主服务B的从服务
     
  ####  事务  
     Redis 事务可以一次执行多个命令， 并且带有以下两个重要的保证：
     ：事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断。
       事务是一个原子操作：事务中的命令要么全部被执行，要么全部都不执行。
      一个事务从开始到执行会经历以下三个阶段：
      开始事务。
      命令入队。
      执行事务
    
    事务的操作方式如下
    redis 127.0.0.1:6379> MULTI   开启事务(事务只能开启一次，如果已经开启了一次，那么再次输入multi命令会报异常)
         ----***----一系列的set 操作
    redis 127.0.0.1:6379> EXEC  提交事务(提交的时候，如果没有执行写的操作，那么空提交也会报异常)
    DISCARD  取消事务，放弃执行事务块内的所有命令。
    EXEC     执行所有事务块内的命令 。
    MULTI    标记一个事务块的开始 。
    UNWATCH  取消 WATCH 命令对所有 key 的监视。
    WATCH key [key ...]  监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断。
    
  #### 持久化机制  
  
    redis的持久化分为 RDB和AOF
   
   1.RDB持久化(默认持久化机制)
      
       RDB是在某个时间点将数据写入一个临时文件，持久化结束后用这次的临时文件替代上次的临时文件。服务启动的时候读取这个临时文件已达到数据恢复的效果
       优点 :使用单独子进程来进行持久化，主进程不进行IO操作，可以保证redis的高性能
       缺点 : RDB是隔一段时间进行持久化，如果持久化之间redis发送故障，这个时候数据就会丢失.重要的数据就不适合这种机制
      这里说的这个执行数据写入到临时文件的时间点是可以通过配置来自己确定的，通过配置redis 在 n 秒内如果超过 m 个 key 被修改这执行一次 RDB 操作。这个操作就类似于在这个时间点来保存一次 Redis 的所有数据，一次快照数据。所有这个持久化方法也通常叫做 snapshots。

     #dbfilename：持久化数据存储在本地的文件
      dbfilename dump.rdb
      #save时间，以下分别表示更改了1个key时间隔900s进行持久化存储；更改了10个key300s进行存储；更改10000个key60s进行存储。
    save 900 1
    save 300 10
    save 60 10000
    
     ##当snapshot时出现错误无法继续时，是否阻塞客户端“变更操作”，“错误”可能因为磁盘已满/磁盘故障/OS级别异常等  
     stop-writes-on-bgsave-error yes  
     ##是否启用rdb文件压缩，默认为“yes”，压缩往往意味着“额外的cpu消耗”，同时也意味这较小的文件尺寸以及较短的网络传输时间  
     rdbcompression yes 
     
   2. AOF持久化机制(在使用aof机制恢复数据之前，一定要先检查aof日志文件是否可用，不能用就进行修复)
        
    AOF持久化的形式是将 操作+数据，以格式化指令(一些修改操作，set等)的放入放入日志中
    优点 aof文件的内容是字符串，易于人工解读。 在数据的完整性方面有更高的支持，如果设置file的时间是1s，那么redis发生故障，只会丢失1s的数据。
    并且如果日志出现问题，可以使用redis-check-aof来进行修复。
    缺点 AOF文件比RDB文件大，并且恢复速度慢，因为它是以命令行的形式来进行恢复的
    
    >>>我们可以简单的认为 AOF 就是日志文件，此文件只会记录“变更操作”(例如：set/del 等)，如果 server 中持续的大量变更操作，将会导致 AOF 文件非常的庞大，意味着 server 失效后，数据恢复的过程将会很长；事实上，一条数据经过多次变更，将会产生多条 AOF 记录，其实只要保存当前的状态，历史的操作记录是可以抛弃的；因为 AOF 持久化模式还伴生了“AOF rewrite”。
AOF 的特性决定了它相对比较安全，如果你期望数据更少的丢失，那么可以采用 AOF 模式。如果 AOF 文件正在被写入时突然 server 失效，有可能导致文件的最后一次记录是不完整，你可以通过手工或者程序的方式去检测并修正不完整的记录，以便通过 aof 文件恢复能够正常；同时需要提醒，如果你的 redis 持久化手段中有 aof，那么在 server 故障失效后再次启动前，需要检测 aof 文件的完整性。
  
      如果想启动AOF格式的数据持久化机制，那么就要关闭RDB机制，
     ##此选项为aof功能的开关，默认为“no”，可以通过“yes”来开启aof功能  
     ##只有在“yes”下，aof重写/文件同步等特性才会生效  
     appendonly yes  
     ##指定aof文件名称  
     appendfilename appendonly.aof 
     ##指定aof操作中文件同步策略，有三个合法值：always everysec no,默认为everysec  
     always：每一条 aof 记录都立即同步到文件，这是最安全的方式，也以为更多的磁盘操作和阻塞延迟，是 IO 开支较大。
     everysec：每秒同步一次，性能和安全都比较中庸的方式，也是 redis 推荐的方式。如果遇到物理服务器故障，有可能导致最近一秒内 aof 记录丢失(可能为部分丢失)。
     no：redis 并不直接调用文件同步，而是交给操作系统来处理，操作系统可以根据 buffer 填充情况 / 通道空闲时间等择机触发同步；这是一种普通的文件操作方式。性能较好，在物理服务器故障时，数据丢失量会因 OS 配置有关
     appendfsync everysec 
     ##在aof-rewrite()期间，appendfsync是否暂缓文件同步，"no"表示“不暂缓”，“yes”表示“暂缓”，默认为“no”  
     no-appendfsync-on-rewrite no  
     ##aof文件rewrite触发的最小文件尺寸(mb,gb),只有大于此aof文件大于此尺寸是才会触发rewrite，默认“64mb”，建议“512mb”  
     auto-aof-rewrite-min-size 64mb
     ##触发下一次rewrite，每一次aof记录的添加，都会检测当前aof文件的尺寸。  
     auto-aof-rewrite-percentage 100  
     
     总结
      1) AOF 更加安全，可以将数据更加及时的同步到文件中，但是 AOF 需要较多的磁盘 IO 开支，AOF 文件尺寸较大，文件内容恢复数度相对较慢。
     *2) snapshot（RDB），安全性较差，它是“正常时期”数据备份以及 master-slave 数据同步的最佳手段，文件尺寸较小，恢复数度较快。
     
     master 通常使用 AOF，slave 使用 snapshot，主要原因是 master 需要首先确保数据完整性，它作为数据备份的第一选择；slave 提供只读服务(目前 slave 只能提供读取服务)，它的主要目的就是快速响应客户端 read 请求。
     但是如果写入操作过于频繁，那么master可以采用RDB(snapshot)的形式，从机采用aof
     具体情况看业务场景
     
   #### 发布订阅  (如果只是想实现简单的发布订阅需求，可以使用redis，复杂的redis好像实现不了，复杂可以使用MQ)
   
        Redis 发布订阅(pub/sub)是一种消息通信模式：发送者(pub)发送消息，订阅者(sub)接收消息。
        Redis 客户端可以订阅任意数量的频道
        实现原理也和MQ里面的发布订阅一致。(先订阅一个channelA，或者多个channelA，B，C，然后发布的时候往A，或者A,B,C中发消息，这样订阅此信道的客户端，就可以收到消息)
   
       redis 127.0.0.1:6379> SUBSCRIBE redisChat
      Reading messages... (press Ctrl-C to quit)   订阅一个名字叫redisChar的信道
    1) "subscribe"
    2) "redisChat"
    3) (integer) 1
      
      redis 127.0.0.1:6379> PUBLISH redisChat "Redis is a great caching technique"
      上面的脚步是往 redisChar这个信道发送消息，发送之后，订阅此信道的客户端窦都可以收到消息
      
  redis 发布订阅常用命令  
      
      PSUBSCRIBE pattern [pattern ...]  订阅一个或多个符合给定模式的频道。
      PUBSUB subcommand [argument [argument ...]]  查看订阅与发布系统状态。
      PUBLISH channel message   将信息发送到指定的频道。
      PUNSUBSCRIBE [pattern [pattern ...]]  退订所有给定模式的频道。
      SUBSCRIBE channel [channel ...]  订阅给定的一个或多个频道的信息。
      UNSUBSCRIBE [channel [channel ...]]  指退订给定的频道。
  
  #### 集群
