# Redis-Jedis简易操作指南

标签（空格分隔）： java

---


## Redis
如今Nosql的运用越来越广泛。Redis数据库除了本身具备内存数据库的各种优点以外，设计合理的数据结构以及友好的操作方式也是我选择使用Redis的原因之一。

## Jedis
Jedis是Reids官方所推荐的操作Redis所用的库。如果你厌烦了java的ORM思想，受够了令人眼花缭乱的xml配置，那么通过Redis你只需要花10分钟就能上手开始使用Redis了！

## Jedis操作手册
### 引入Jedis库
导入maven的依赖。如果你的项目没有使用maven的话，那么从现在开始请将你的项目重新用Maven构建一次吧！
```xml
<dependency>
			<groupId>redis.clients</groupId>
			<artifactId>jedis</artifactId>
			<version>2.9.0</version>
		</dependency>
```
### 构建数据库资源工具RedisUtils
```java
package com.totalextrip.util;


import com.totalextrip.exception.ApplicationRunTimeException;
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;

/**
 * redis工具类
 *
 * @author lincanhan
 */
public final class RedisUtil {


    /**
     * 私有化构造器
     */
    private RedisUtil() {
        //Nope
    }


    private final static class JedisPoolInnerHolder {

        /**
         * 私有化构造器
         */
        private JedisPoolInnerHolder() {

        }

        // Redis服务器IP
        private static String ADDR = "localhost";

        // Redis的端口号
        private static int PORT = 6379;

        // 访问密码
        private static String AUTH = "你的密码";

        // 可用连接实例的最大数目，默认值为8；
        // 如果赋值为-1，则表示不限制；如果pool已经分配了maxActive个jedis实例，则此时pool的状态为exhausted(耗尽)。
        private static int MAX_ACTIVE = 1024;

        // 控制一个pool最多有多少个状态为idle(空闲的)的jedis实例，默认值也是8。
        private static int MAX_IDLE = 200;

        // 等待可用连接的最大时间，单位毫秒，默认值为-1，表示永不超时。如果超过等待时间，则直接抛出JedisConnectionException；
        private static int MAX_WAIT = 10000;

        private static int TIMEOUT = 10000;

        // 在borrow一个jedis实例时，是否提前进行validate操作；如果为true，则得到的jedis实例均是可用的；
        private static boolean TEST_ON_BORROW = true;

        private static JedisPool jedisPool;

        static {
            try {
                JedisPoolConfig config = new JedisPoolConfig();
                //config.setMaxActive(MAX_ACTIVE);
                config.setMaxTotal(MAX_ACTIVE);
                config.setMaxIdle(MAX_IDLE);
                //config.setMaxWait(MAX_WAIT);
                config.setMaxWaitMillis(MAX_WAIT);
                config.setTestOnBorrow(TEST_ON_BORROW);
                jedisPool = new JedisPool(config, ADDR, PORT, TIMEOUT, AUTH);
                //jedisPool = new JedisPool(config, ADDR, PORT);
            } catch (Exception e) {
                throw new ApplicationRunTimeException("缓存服务器初始化异常", e);
            }
        }


    }

    /**
     * 获取Jedis实例
     *
     * @return
     */
    public static Jedis getJedisFromJedis() {
        return JedisPoolInnerHolder.jedisPool.getResource();
    }

}

```
Redis中没有账户的概念，用户的验证权限只和password有关。
无论你的Redis是什么版本，在默认配置下Redis在localhost环境中不需要密码就能够进行访问。
若你需要连接配置密码以及远程访问的话，请取消REDIS_HOME/redis.conf文件中的requirepass注释，且注释掉bind配置选项如下：
```python
requirepass 你的密码
#bind 127.0.0.1
```
启动redis请通过redis-server parameter1的方式进行。
若不传递配置文件参数，则redis会按照默认配置启动。
```shell
# 读取redis.conf配置文件启动redis服务器。
redis-server redis.conf
```

### 使用方法
```java
/**
     * redis存储字符串
     */
    public void testString() {
        //-----添加数据----------
        jedis.set("name", "lch");//向key-->name中放入了value-->xinxin
        System.out.println(jedis.get("name"));//执行结果：xinxin

        jedis.append("name", " is my lover"); //拼接
        System.out.println(jedis.get("name"));

        jedis.del("name");  //删除某个键
        System.out.println(jedis.get("name"));
        //设置多个键值对
        jedis.mset("name", "lsx", "age", "23", "qq", "476777XXX");
        jedis.incr("age"); //进行加1操作
        System.out.println(jedis.get("name") + "-" + jedis.get("age") + "-" + jedis.get("qq"));
    }
    
```

### 事务操作
想在redis中开启简单事务，需要用到multi，exec，close等指令。
为什么说是简单事务呢？因为复杂业务下的事务你还会用到watch等更高级的指令操作。
> ####redis中的事务 
redis的事务只具备隔离性与原子性。事务隔离性保证了凡是使用multi与exec进行的数据库操作，数据库会依次批量执行。待一次事务命令全部结束(exec被调用)后，才会开始下一次事务性操作，不会出现事务中的操作互相交叉影响的情况出现。原子性保证了一个事务中的所有操作要么全部成功，要么全部失败。

```java
 /**
     * 事务操作
     */
    public static void transcationTester() {
        Transaction t = jedis.multi();
        //添加
        jedis.sadd("user", "liuling");
        jedis.del("user");
        Response<String> address = t.hget("customerInfo", "Address");
        t.exec();
        System.out.println(address.get());
        t.close();
        jedis.close();
    }
```
在事务结束前，所有查询语句由于还未被提交到数据库，所以任何get类的方法，都只能返回Response对象。待exec()方法调用之后，再在response对象上调用get()方能拿到相应的值。

##总结
1. 使用Jedis库能够非常方便的操作Redis数据库。
2. redis中的事务与关系型数据库有所不同，只具备隔离性与原子性。
3. 在事务提交之前，任何get操作都拿不到值，需等待exec()方法调用后在Response对象上调用get()方法才能获得最后的值。
4. Redis设计了Watch命令，来防止并发环境下其他客户端通过非事务的方式修改某个键值，从而影响了事务的完整性。Watch命令是通过乐观锁来实现的。







