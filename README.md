
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
