### redis持久化rdb和aof的区别

#### rdb:

Snapshot快照，恢复时是将快照文件直接读到内存里。
Redis会单独创建（fork）一个子进程来进行持久化，会先将数据写入到一个临时文件中，待持久化过程都结束了，再用这个临时文件替换上次持久化好的文件。整个过程中，主进程是不进行任何IO操作的，这就确保了极高的性能。
优点：
1）节省磁盘空间
2）恢复速度快
缺点：
1）虽然Redis在fork时使用了写时拷贝技术，数据庞大时还是比较消耗性能
2）在备份周期在一定间隔时间做一次备份，所以如果Redis意外down调的话，就会丢失最后一次快照后的所有修改

#### aof:

以日志的形式来记录每个写操作，将Redis执行过的所有写指令记录下来（读操作不记录），只许追加文件但不可以改写文件
优点:
1）丢失数据概率更低
2）可读的日志文本，通过操作AOF文件，可以处理误操作
缺点:
1）比起RDB占用更多的磁盘空间
2）恢复备份速度要慢
3）每次读写都同步的话，有一定的性能压力

> [https://gitee.com/zssea/notes/blob/master/面试/尚硅谷Java面试_高频重点面试题 （第一季）](https://gitee.com/zssea/notes/blob/master/%E9%9D%A2%E8%AF%95/%E5%B0%9A%E7%A1%85%E8%B0%B7Java%E9%9D%A2%E8%AF%95_%E9%AB%98%E9%A2%91%E9%87%8D%E7%82%B9%E9%9D%A2%E8%AF%95%E9%A2%98%20%EF%BC%88%E7%AC%AC%E4%B8%80%E5%AD%A3%EF%BC%89.md#rdb)

### 怎样设计一个完善而可靠的redis分布式锁
#### 初始版搭建程序
``` java
@RestController
public class GoodController{

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @Value("${server.port}")
    private String serverPort;

    @GetMapping("/buy_goods")
    public String buy_Goods(){

        String result = stringRedisTemplate.opsForValue().get("goods:001");// get key ====看看库存的数量够不够
        int goodsNumber = result == null ? 0 : Integer.parseInt(result);
        if(goodsNumber > 0){
            int realNumber = goodsNumber - 1;
            stringRedisTemplate.opsForValue().set("goods:001", String.valueOf(realNumber));
            System.out.println("成功买到商品，库存还剩下: "+ realNumber + " 件" + "\t服务提供端口" + serverPort);
            return "成功买到商品，库存还剩下:" + realNumber + " 件" + "\t服务提供端口" + serverPort;
        }else{
            System.out.println("商品已经售完/活动结束/调用超时,欢迎下次光临" + "\t服务提供端口" + serverPort);
        }

        return "商品已经售完/活动结束/调用超时,欢迎下次光临" + "\t服务提供端口" + serverPort;
    }
    
}
```

#### redis分布式锁01
上面的程序问题分析：
这样的程序就算在单机版下面，多线程环境下都会超卖，更别说分布式环境。
修改：
加synchronized或者ReentrantLock

``` java
@RestController
public class GoodController {

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @Value("${server.port}")
    private String serverPort;

    @GetMapping("/buy_goods")
    public String buy_Goods() {

        synchronized (this) {
            String result = stringRedisTemplate.opsForValue().get("goods:001");// get key ====看看库存的数量够不够
            int goodsNumber = result == null ? 0 : Integer.parseInt(result);
            if (goodsNumber > 0) {
                int realNumber = goodsNumber - 1;
                stringRedisTemplate.opsForValue().set("goods:001", String.valueOf(realNumber));
                System.out.println("成功买到商品，库存还剩下: " + realNumber + " 件" + "\t服务提供端口" + serverPort);
                return "成功买到商品，库存还剩下:" + realNumber + " 件" + "\t服务提供端口" + serverPort;
            } else {
                System.out.println("商品已经售完/活动结束/调用超时,欢迎下次光临" + "\t服务提供端口" + serverPort);
            }

            return "商品已经售完/活动结束/调用超时,欢迎下次光临" + "\t服务提供端口" + serverPort;
        }
    }
}
```
#### redis分布式锁02
上面的程序问题分析：
分布式部署后，单机锁还是出现超卖现象，需要分布式锁
![](https://upload-images.jianshu.io/upload_images/1626396-09623ef7f28fa656.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
启动两个微服务：1111，2222，多次访问http://localhost/buy_goods，服务提供端口在1111，2222两者之间横跳。
上面手点，下面高并发模拟

用到Apache JMeter，100个线程同时访问http://localhost/buy_goods
![](https://upload-images.jianshu.io/upload_images/1626396-a2394ed4080ffdfd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](https://upload-images.jianshu.io/upload_images/1626396-852cdce4feac2d79.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
启动测试，后台打印如下：
![](https://upload-images.jianshu.io/upload_images/1626396-e60861dd5daf3f8d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](https://upload-images.jianshu.io/upload_images/1626396-381716107efe0f00.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
修改：
上redis分布式锁setnx，代码修改如下：
``` java
@RestController
public class GoodController {

    public static final String REDIS_LOCK = "atguiguLock";

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @Value("${server.port}")
    private String serverPort;

    @GetMapping("/buy_goods")
    public String buy_Goods() {

        String value = UUID.randomUUID().toString() + Thread.currentThread().getName();
        Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(REDIS_LOCK, value);
        if (!flag) {
            // 加锁失败
            return "抢锁失败";
        }
        String result = stringRedisTemplate.opsForValue().get("goods:001");// get key ====看看库存的数量够不够
        int goodsNumber = result == null ? 0 : Integer.parseInt(result);
        if (goodsNumber > 0) {
            int realNumber = goodsNumber - 1;
            stringRedisTemplate.opsForValue().set("goods:001", String.valueOf(realNumber));
            System.out.println("成功买到商品，库存还剩下: " + realNumber + " 件" + "\t服务提供端口" + serverPort);
            stringRedisTemplate.delete(REDIS_LOCK);
            return "成功买到商品，库存还剩下:" + realNumber + " 件" + "\t服务提供端口" + serverPort;
        } else {
            System.out.println("商品已经售完/活动结束/调用超时,欢迎下次光临" + "\t服务提供端口" + serverPort);
        }

        return "商品已经售完/活动结束/调用超时,欢迎下次光临" + "\t服务提供端口" + serverPort;
    }

}
```

#### redis分布式锁03
上面的程序问题分析：
1. 加锁解锁，lok/unlock必须同时出现并保证调用。
出现异常的话，可能无法释放锁，必须要在代码层面finally释放锁。
解决方法：try…finally…
2. 部署了微服务jar包的机器挂了，代码层面根本没有走到finally这块，没办法保证解锁，这个key没有被删除，需要加入一个过期时间限定key

代码修改如下：
``` java
@RestController
public class GoodController {

    public static final String REDIS_LOCK = "atguiguLock";

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @Value("${server.port}")
    private String serverPort;

    @GetMapping("/buy_goods")
    public String buy_Goods() {

        String value = UUID.randomUUID().toString() + Thread.currentThread().getName();
        try {
            Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(REDIS_LOCK, value);
            stringRedisTemplate.expire(REDIS_LOCK, 10L, TimeUnit.SECONDS);
            if (!flag) {
                // 加锁失败
                return "抢锁失败";
            }
            String result = stringRedisTemplate.opsForValue().get("goods:001");// get key ====看看库存的数量够不够
            int goodsNumber = result == null ? 0 : Integer.parseInt(result);
            if (goodsNumber > 0) {
                int realNumber = goodsNumber - 1;
                stringRedisTemplate.opsForValue().set("goods:001", String.valueOf(realNumber));
                System.out.println("成功买到商品，库存还剩下: " + realNumber + " 件" + "\t服务提供端口" + serverPort);
                return "成功买到商品，库存还剩下:" + realNumber + " 件" + "\t服务提供端口" + serverPort;
            } else {
                System.out.println("商品已经售完/活动结束/调用超时,欢迎下次光临" + "\t服务提供端口" + serverPort);
            }

            return "商品已经售完/活动结束/调用超时,欢迎下次光临" + "\t服务提供端口" + serverPort;
        } finally {
            stringRedisTemplate.delete(REDIS_LOCK);
        }
    }

}
```
#### redis分布式锁04
上面的程序问题分析：
设置key+过期时间分开了，必须要合并成一行具备原子性。意思就是要么加锁成功设置过期时间也成功，要么加锁失败设置过期时间也失败，否则假如是加锁成功后立马宕机，那就设置过期时间失败，只有加锁没有解锁了。

代码修改如下：
``` java
@RestController
public class GoodController {

    public static final String REDIS_LOCK = "atguiguLock";

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @Value("${server.port}")
    private String serverPort;

    @GetMapping("/buy_goods")
    public String buy_Goods() {

        String value = UUID.randomUUID().toString() + Thread.currentThread().getName();
        try {
            Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(REDIS_LOCK, value, 10L, TimeUnit.SECONDS);
            if (!flag) {
                // 加锁失败
                return "抢锁失败";
            }
            String result = stringRedisTemplate.opsForValue().get("goods:001");// get key ====看看库存的数量够不够
            int goodsNumber = result == null ? 0 : Integer.parseInt(result);
            if (goodsNumber > 0) {
                int realNumber = goodsNumber - 1;
                stringRedisTemplate.opsForValue().set("goods:001", String.valueOf(realNumber));
                System.out.println("成功买到商品，库存还剩下: " + realNumber + " 件" + "\t服务提供端口" + serverPort);
                return "成功买到商品，库存还剩下:" + realNumber + " 件" + "\t服务提供端口" + serverPort;
            } else {
                System.out.println("商品已经售完/活动结束/调用超时,欢迎下次光临" + "\t服务提供端口" + serverPort);
            }

            return "商品已经售完/活动结束/调用超时,欢迎下次光临" + "\t服务提供端口" + serverPort;
        } finally {
            stringRedisTemplate.delete(REDIS_LOCK);
        }
    }

}
```
新的问题：
张冠李戴，删除了别人的锁。
![](https://upload-images.jianshu.io/upload_images/1626396-c58d0daeaaf61011.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
什么意思呢？假设这么一个场景，A线程先进来，设置了redis锁，但是业务调用时间过长，超过了redis的超时时间，redis锁自动删除了。然后B线程进来了，加了redis锁，开始做业务逻辑。这是A线程的业务终于做完了，做完之后立马就把key给删了。然后过了一会儿B线程也做完了，发现他的key已经被人删了。
解决方法：
在删除key之前，先获取key所对应的值，也就是存在redis里的value。如果这个value和当前线程的value变量一样（value是uuid+线程名字），才删除key
``` java
@RestController
public class GoodController {

    public static final String REDIS_LOCK = "atguiguLock";

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @Value("${server.port}")
    private String serverPort;

    @GetMapping("/buy_goods")
    public String buy_Goods() {

        String value = UUID.randomUUID().toString() + Thread.currentThread().getName();
        try {
            Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(REDIS_LOCK, value, 10L, TimeUnit.SECONDS);
            if (!flag) {
                // 加锁失败
                return "抢锁失败";
            }
            String result = stringRedisTemplate.opsForValue().get("goods:001");// get key ====看看库存的数量够不够
            int goodsNumber = result == null ? 0 : Integer.parseInt(result);
            if (goodsNumber > 0) {
                int realNumber = goodsNumber - 1;
                stringRedisTemplate.opsForValue().set("goods:001", String.valueOf(realNumber));
                System.out.println("成功买到商品，库存还剩下: " + realNumber + " 件" + "\t服务提供端口" + serverPort);
                return "成功买到商品，库存还剩下:" + realNumber + " 件" + "\t服务提供端口" + serverPort;
            } else {
                System.out.println("商品已经售完/活动结束/调用超时,欢迎下次光临" + "\t服务提供端口" + serverPort);
            }

            return "商品已经售完/活动结束/调用超时,欢迎下次光临" + "\t服务提供端口" + serverPort;
        } finally {
            if (stringRedisTemplate.opsForValue().get(REDIS_LOCK).equalsIgnoreCase(value)) {
                stringRedisTemplate.delete(REDIS_LOCK);
            }
        }
    }

}
```
#### redis分布式锁05
上面的程序问题分析：
finally块的判断 + del删除操作不是原子性的
解决方法：
1. 用lua脚本
> https://redis.io/commands/set

It is possible to make this system more robust modifying the unlock schema as follows:
- Instead of setting a fixed string, set a non-guessable large random string, called token.
- Instead of releasing the lock with [DEL](https://redis.io/commands/del), send a script that only removes the key if the value matches.

This avoids that a client will try to release the lock after the expire time deleting the key created by another client that acquired the lock later.
An example of unlock script would be similar to the following:
``` lua
if redis.call("get",KEYS[1]) == ARGV[1]
then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

2. 用redis自身的事务(这里略过，不是重点，以后有空再补充)

#### redis分布式锁06
继续上一章节，解决之道
``` java
public static final String REDIS_LOCK = "redis_lock";

@Autowired
private StringRedisTemplate stringRedisTemplate;

public void m(){
    String value = UUID.randomUUID().toString() + Thread.currentThread().getName();

    try{
		Boolean flag = stringRedisTemplate.opsForValue()//使用另一个带有设置超时操作的方法
            .setIfAbsent(REDIS_LOCK, value, 10L, TimeUnit.SECONDS);
		//设定时间
        //stringRedisTemplate.expire(REDIS_LOCK, 10L, TimeUnit.SECONDS);
        
   		if(!flag) {
        	return "抢锁失败";
	    }
        
    	...//业务逻辑
            
    }finally{
        while(true){
            stringRedisTemplate.watch(REDIS_LOCK);
            if(stringRedisTemplate.opsForValue().get(REDIS_LOCK).equalsIgnoreCase(value)){
                stringRedisTemplate.setEnableTransactionSupport(true);
                stringRedisTemplate.multi();
                stringRedisTemplate.delete(REDIS_LOCK);
                List<Object> list = stringRedisTemplate.exec();
                if (list == null) {
                    continue;
                }
            }
            stringRedisTemplate.unwatch();
            break;
        } 
    }
}

```
#### redis分布式锁07
Redis调用Lua脚本通过eval命令保证代码执行的原子性
RedisUtils：
``` java
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;

public class RedisUtils {

	private static JedisPool jedisPool;
	
	static {
		JedisPoolConfig jpc = new JedisPoolConfig();
		jpc.setMaxTotal(20);
		jpc.setMaxIdle(10);
		jedisPool = new JedisPool(jpc);
	}
	
	public static JedisPool getJedis() throws Exception{
		if(jedisPool == null)
			throw new NullPointerException("JedisPool is not OK.");
		return jedisPool;
	}
	
}
```
``` java
public static final String REDIS_LOCK = "redis_lock";

@Autowired
private StringRedisTemplate stringRedisTemplate;

public void m(){
    String value = UUID.randomUUID().toString() + Thread.currentThread().getName();

    try{
		Boolean flag = stringRedisTemplate.opsForValue()//使用另一个带有设置超时操作的方法
            .setIfAbsent(REDIS_LOCK, value, 10L, TimeUnit.SECONDS);
		//设定时间
        //stringRedisTemplate.expire(REDIS_LOCK, 10L, TimeUnit.SECONDS);
        
   		if(!flag) {
        	return "抢锁失败";
	    }
        
    	...//业务逻辑
            
    }finally{
    	Jedis jedis = RedisUtils.getJedis();
    	
    	String script = "if redis.call('get', KEYS[1]) == ARGV[1] "
    			+ "then "
    			+ "    return redis.call('del', KEYS[1]) "
    			+ "else "
    			+ "    return 0 "
    			+ "end";
    	
    	try {
    		
    		Object o = jedis.eval(script, Collections.singletonList(REDIS_LOCK),// 
    				Collections.singletonList(value));
    		
    		if("1".equals(o.toString())) {
    			System.out.println("---del redis lock ok.");
    		}else {
    			System.out.println("---del redis lock error.");
    		}
    		
    		
    	}finally {
    		if(jedis != null) 
    			jedis.close();
    	}
    }
}
```
#### redis分布式锁08
确保RedisLock过期时间大于业务执行时间的问题
Redis分布式锁如何续期？






