
# 应用场景

  ##### redis中键的生存时间(expire)

    使用expire命令设置一个键的生存时间，到时间后redis会自动删除它。
    expire key seconds(秒)  设置生存时间
    ttl key                 查看剩余过期时间
    persist key             取消生存时间
    应用场景:
      1.限时的优惠活动信息
      2.积分排行榜，定时去更新
      3.手机验证码，设置过期时间（设置10分钟或者15分钟内有效）
 ##### redis的事务

    redis中的事务是一组命令的集合，要么全部成功，要么全部失败
    原理： 先将输入一个事务的命令，发送给redis进行缓存，最后在让redis依次执行这些命令。
    应用场景:
     1. 一组命令必须同时都执行，或者都不执行。
     2. 我们想要保证一组命令在执行的过程之中不被其它命令插入。
     
     注意点： redis不支持回滚。但是它保证一组命令执行的时候，只要有一个命令有错误，其它的命令都不会执行。
 ##### 数据的排序(sort)
 
    sort命令可以对列表进行排序，集合和有序集合都可以
    sort key [desc] [limit offset count]
    通过上诉命令，可以对某个key进行排序，并且还可以分页
 ##### 任务队列
   
    任务队列:使用lpush和rpop(brpop)实现普通的任务队列
    里面使用lpush 存入 一定数量的队列，然后在通过brpop去取值
 ##### 管道(pipeline)
  管道功能在命令行是没有的，但是在jedis客户端是存在的。
  
    1：不使用管道方式，插入1000条数据耗时328毫秒
    
    public static void testInsert() {
    long currentTimeMillis = System.currentTimeMillis();
    Jedis jedis = new Jedis("127.0.0.1", 6379);
    for (int i = 0; i < 1000; i++) {
        jedis.set("test" + i, "test" + i);
    }
    long endTimeMillis = System.currentTimeMillis();
    System.out.println(endTimeMillis - currentTimeMillis);
   
     2：使用管道方式，插入1000条数据耗时37毫秒

    public static void testPip() {
    long currentTimeMillis = System.currentTimeMillis();
    Jedis jedis = new Jedis(127.0.0.1", 6379);
    Pipeline pipelined = jedis.pipelined(); 核心代码
    for (int i = 0; i < 1000; i++) {
        pipelined.set("bb" + i, i + "bb");
    }
    pipelined.sync();
    long endTimeMillis = System.currentTimeMillis();
    System.out.println(endTimeMillis - currentTimeMillis);
    在插入更多数据的时候，管道的优势更加明显：测试10万条数据的时候，不使用管道要40秒，实用管道378毫秒
    
  ##### 持久化和发布订阅机制，前面有说。简单用法

    // 订阅频道数据
     public static void testSubscribe() {
    //连接Redis数据库
    Jedis jedis = new Jedis("127.0.0.1", 6379);
    JedisPubSub jedisPubSub = new JedisPubSub() {
 
        // 当向监听的频道发送数据时，这个方法会被触发
        @Override
        public void onMessage(String channel, String message) {
            System.out.println("收到消息" + message);
            //当收到 "unsubscribe" 消息时，调用取消订阅方法
            if ("unsubscribe".equals(message)) {
                this.unsubscribe();
            }
        }
 
        // 当取消订阅指定频道的时候，这个方法会被触发
        @Override
        public void onUnsubscribe(String channel, int subscribedChannels) {
            System.out.println("取消订阅频道" + channel);
        }
 
    };
    // 订阅之后，当前进程一致处于监听状态，当被取消订阅之后，当前进程会结束
    jedis.subscribe(jedisPubSub, "ch1");
     }
     // 发布频道数据
    public static void testPubSub() throws Exception {
    //链接Redis数据库
    Jedis jedis = new Jedis("127.0.0.1", 6379);
    //发布频道 "ch1" 和消息 "hello redis"
    jedis.publish("ch1", "hello redis");
    //关闭连接
    jedis.close();
    }

  
  #####  redis回收策略，防止数据丢失
    
     1.使用hash结构，这样可以在在一个key中，存入多个数据，减少key的数量
     2.设置key的过期时间，可以有效的减少key的数量
     3.回收key，使用一定的回收策略，如下:
      3.1)在Redis配置文件中(一般叫Redis.conf)，通过设置“maxmemory”属性的值可以限制Redis最大使用的内存，修改后重启实例生效
      3.2)可在Redis.conf配置文件中修改“maxmemory-policy”属性值。 若是Redis数据集中的key都设置了过期时间，那么“volatile-ttl”策略是比较好的选择。但如果key在达到最大内存限制时没能够迅速过期，或者根本没有设置过期时间。那么设置为“allkeys-lru”值比较合适，它允许Redis从整个数据集中挑选最近最少使用的key进行删除(LRU淘汰算法，推荐使用)
      volatile-lru： 使用LRU算法从已设置过期时间的数据集合中淘汰数据。
      volatile-ttl：从已设置过期时间的数据集合中挑选即将过期的数据淘汰。
      volatile-random：从已设置过期时间的数据集合中随机挑选数据淘汰。
      allkeys-lru：使用LRU算法从所有数据集合中淘汰数据。
      allkeys-random：从数据集合中任意选择数据淘汰
      no-enviction：禁止淘汰数据。

  ##### 限制网站访问频率

    //指定Redis数据库连接的IP和端口
    String host = "192.168.33.130";
     int port = 6379;
    Jedis jedis = new Jedis(host, port);
   
    //限制网站访客访问频率 一分钟之内最多访问10次
   
      @Test
    public void test3() throws Exception {
    // 模拟用户的频繁请求
     for (int i = 0; i < 20; i++) {
        boolean result = testLogin("192.168.1.100");
        if (result) {
            System.out.println("正常访问");
        } else {
            System.err.println("访问受限");
        }
    }
 
 }
 
    public boolean testLogin(String ip) {
    String value = jedis.get(ip);
    if (value == null) {
        //初始化时设置IP访问次数为1
           jedis.set(ip, "1");
        //设置IP的生存时间为60秒，60秒内IP的访问次数由程序控制
           jedis.expire(ip, 60);
      } else {
           int parseInt = Integer.parseInt(value);
           //如果60秒内IP的访问次数超过10，返回false,实现了超过10次禁止分的功能
           if (parseInt > 10) {
               return false;
          } else {
              //如果没有10次，可以自增
              jedis.incr(ip);
           }
       }
     return true;
      }
 ##### 监控变量在事务执行时是否被修改

    // 指定Redis数据库连接的IP和端口
    String host = "192.168.33.130";
    int port = 6379;
    Jedis jedis = new Jedis(host, port);
  

    //监控变量a在一段时间内是否被修改，若没有，则执行事务，若被修改，则事务不执行

    @Test
    public void test4() throws Exception {
    //监控变量a，在事务执行后watch功能也结束
    jedis.watch("a");
    //需要数据库中先有a，并且a的值为字符串数字
    String value = jedis.get("a");
    int parseInt = Integer.parseInt(value);
    parseInt++;
    System.out.println("线程开始休息。。。");
    Thread.sleep(5000);
 
    //开启事务
    Transaction transaction = jedis.multi();
    transaction.set("a", parseInt + "");
    //执行事务
    List<Object> exec = transaction.exec();
    if (exec == null) {
        System.out.println("事务没有执行.....");
    } else {
        System.out.println("正常执行......");
    }
    }
  
  ##### 各种计数
   
       1.喜欢数
       2.评论数
       3.浏览数  等
  
     由于redis本身的特性(单线程)，所以用redis来进行计数是非常不错的方法，可以有效的防止并发情况下计数混乱问题。
    
   ##### 缓存功能
     
       存储一些查询时间较长,数量较大的数据，用作缓存，减少DB的压力(用作缓存的时候，需要考虑相关策略，以防内存被撑爆导致数据丢失)

  #####  锁功能（并发场景中需要写入DB的时候，保证数据的可以正确的写入DB中，不会因为，并发而导致数据错乱）
  
     1.INCR 锁
        加锁思路：key不存在，那么key的值就被初始化为0，然后在执行incr操作进行加一，然后其它用户在执行incr操作进行加一，如果返回结果大于1说明这个锁正在被使用中
    1)、 客户端A请求服务器获取key的值为1表示获取了锁
    2)、 客户端B也去请求服务器获取key的值为2表示获取锁失败
    3)、 客户端A执行代码完成，删除锁
    4)、 客户端B在等待一段时间后在去请求的时候获取key的值为1表示获取锁成功
    5)、 客户端B执行代码完成，删除锁

    $redis->incr($key);
    $redis->expire($key, $ttl); //设置生成时间为1秒
  
    2. setnx  锁
        加锁思路: 如果key不存在，将key设置为value，如果key已经存在，则setnx不做任何操作
    1)、 客户端A请求服务器设置key的值，如果设置成功就表示加锁成功
    2)、 客户端B也去请求服务器设置key的值，如果返回失败，那么就代表加锁失败
    3)、 客户端A执行代码完成，删除锁
    4)、 客户端B在等待一段时间后在去请求设置key的值，设置成功
    5)、 客户端B执行代码完成，删除锁

    $redis->setNX($key, $value);
    $redis->expire($key, $ttl);
--------------------------------------------------
    
      上面的两种加锁方式，在配上过期时间等，可以满足日常并发场景下的数据不一致问题(后续会就尝试实现 zookeeper的锁，redis的锁，java锁，探讨其中区别)

   #### 缓存雪崩,缓存穿透 
   
     缓存穿透是指查询一个一定不存在的数据，由于缓存是不命中时需要从数据库查询，查不到数据则不写入缓存，这将导致这个不存在的数据每次请求都要到数据库去查询，造成缓存穿透。对数据库压力过大没导致服务宕机。
     解决方法：
     也可以采用一个更为简单粗暴的方法，如果一个查询返回的数据为空（不管是数 据不存在，还是系统故障），我们仍然把这个空结果进行缓存，但它的过期时间会很短，最长不超过五分钟。
     
     缓存雪崩，某个时间点的缓存全部过期，发送所有请求都往数据库里面查询，造成数据库压力过大，而宕机。
     解决方法
    1. 在缓存失效后，通过加锁或者队列来控制读数据库写缓存的线程数量。比如对某个key只允许一个线程查询数据和写缓存，其他线程等待。
    2.可以通过缓存reload机制，预先去更新缓存，再即将发生大并发访问前手动触发加载缓存
    3. 不同的key，设置不同的过期时间，让缓存失效的时间点尽量均匀
     4.做二级缓存，或者双缓存策略。A1为原始缓存，A2为拷贝缓存，A1失效时，可以访问A2，A1缓存失效时间设置为短期，A2设置为长期。

       
      
