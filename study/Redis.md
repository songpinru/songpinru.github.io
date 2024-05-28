# 部署

下载源码

解压

安装C的编译器：

```shell
sudo yum install gcc-c++

#没网手动安装
rpm -ivh mpfr-2.4.1-6.el6.x86_64.rpm
rpm -ivh cpp-4.4.7-17.el6.x86_64.rpm
rpm -ivh ppl-0.10.2-11.el6.x86_64.rpm
rpm -ivh cloog-ppl-0.15.7-1.2.el6.x86_64.rpm
rpm -ivh gcc-4.4.7-17.el6.x86_64.rpm
```

进入解压的Redis源码目录

```shell
# 如果Jemalloc/jemalloc.h：没有文件
# make disclean

make

sudo make install
```

默认安装目录：/usr/local/bin

* Redis-check-aof
* Redis-check-dump
* Redis-sentinel
* redis-server：服务器启动命令
  * 可以后面接配置文件（.conf）的路径
  * PS:后台启动**daemonize** **no** **改成** **yes**
* Redis-cli：客户端
  * 关闭：shutdown

# 操作

* select 选择数据库
* flushdb清空数据库
* flushall 清空所有

## KEY

| command             | description                                                |
| ------------------- | ---------------------------------------------------------- |
| del key             | 在 key 存在时删除 key                                      |
| exists key          | key 是否存在                                               |
| keys *              |                                                            |
| expire key          | 给 key 设置过期时间，以秒计                                |
| move key db         | 当前数据库的 key 移动到给定的数据库 db                     |
| persist key         | 移除 key 的过期时间，key 将持久保持                        |
| ttl key             | 以秒为单位，返回给定 key 的剩余生存时间(TTL, time to live) |
| randomkey           | 从当前数据库中随机返回一个 key                             |
| rename key newkey   | 修改 key 的名称                                            |
| renamenx key newkey | 仅当 newkey 不存在时，将 key 改名为 newkey                 |

## String

| command                 | description                                   |
| ----------------------- | --------------------------------------------- |
| set key value           | 设置指定 key 的值                             |
| get key                 | 获取指定 key 的值                             |
| getrange key start end  | 返回 key 中字符串值的子字符串                 |
| getset key value        | 将给定 key 的值设为 value ，并返回 key 的旧值 |
| mget key1 key2...       | 获取所有(一个或多个)给定 key 的值             |
| setex key seconds value | 设置key的过期时间和值                         |
| setnx key value         | 只有在 key 不存在时设置 key 的值              |
| strlen key              | 返回key的值的长度                             |
| incr key                | key的值增1                                    |
| incrby key increment    | key的值增increment                            |
| decr key                | key的值减1                                    |
| decrby key increment    | key的值减increment                            |
| append key value        | key的值追加value                              |

## List

| command                                                      | description                  |
| ------------------------------------------------------------ | ---------------------------- |
| lset key index value                                         | 通过索引设置列表元素的值     |
| [lrem key count value](https://www.runoob.com/redis/lists-lrem.html) | 移除列表元素                 |
| lindex key index                                             | 通过索引获取列表中的元素     |
| llen key                                                     | 获取列表长度                 |
| [linsert key BEFORE\|AFTER pivot value](https://www.runoob.com/redis/lists-linsert.html) | 在列表的元素前或者后插入元素 |
| lpush key value1 value2...                                   | 将一个或多个值插入到列表头部 |
| rpush key value1 value2...                                   | 将一个或多个值插入到列表尾部 |
| lpop key                                                     | 移出并获取列表的第一个元素   |
| rpop key                                                     | 移出并获取列表的最后一个元素 |
| lrange key start end                                         | 获取列表指定范围内的元素     |
| ltrim key start end                                          | 让列表只保留指定区间内的元素 |

## Set

| command                   | description                         |
| ------------------------- | ----------------------------------- |
| sadd key value1 value2... | 向集合添加一个或多个成员            |
| scard key                 | 查看集合中的总数                    |
| smembers key              | 返回集合中的所有成员                |
| sismember key value       | 判断value 元素是否是集合 key 的成员 |
| srem key value1 value2..  | 移除集合中一个或多个成员            |
| spop key                  | 移除并返回集合中的一个随机元素      |
| sunion key1 key2          | 并集                                |
| sinsert key1 key2         | 交集                                |
| sdiff key1 key2           | 差集                                |

## Zset(Sorted Set)

| command                                  | description                                            |
| ---------------------------------------- | ------------------------------------------------------ |
| zadd key score1 member1 score2 member2.. | 向有序集合添加一个或多个成员，或者更新已存在成员的分数 |
| zcard key                                | 获取有序集合的成员数                                   |
| zcount key min max                       | 计算在有序集合中指定区间分数的成员数                   |
| zincrby key increment member             | 有序集合中对指定成员的分数加上增量 increment           |
| zrange key start stop [withscores]       | 通过索引区间返回有序集合指定区间内的成员               |
| zrevrangekey start stop [withscores]     | zrange逆序                                             |
| zrank key member                         | 返回有序集合中指定成员的索引                           |
| zrem key member1 member2...              | 移除有序集合中的一个或多个成员                         |

## Hash

| command                                  | description                           |
| ---------------------------------------- | ------------------------------------- |
| hset key field value                     | 增加                                  |
| hsetnx key field value                   | 不存在才增加                          |
| hmset key field1 value1 field2 value2... | 增加多个                              |
| hget key field                           | 获取存储在哈希表中指定字段的值        |
| hkeys key                                | 获取所有哈希表中的字段                |
| hvals key                                | 获取哈希表中所有值                    |
| hdel key field1 field2...                | 删除一个或多个哈希表字段              |
| hexists key field                        | 查看哈希表 key 中，指定的字段是否存在 |
| hlen key                                 | 获取哈希表中字段的数量                |

## GEOSpatial 地理位置

官方文档：https://www.redis.net.cn/order/3685.html

* GEOADD key 值（经度，纬度，地名）
* GEOPOS key 地名
* GEODIST key 地名 地名 单位
* GEORADIUS key 经度 纬度 半径 单位
* GEORADIUSBYMEMBER key 地名 半径 单位
* GEOHASH

### GEOADD

```bash
# getadd 添加地理位置
# 规则：两级无法直接添加，我们一般会下载城市数据，直接通过java程序一次性导入！
# 有效的经度从-180度到180度。
# 有效的纬度从-85.05112878度到85.05112878度。
# 当坐标位置超出上述指定范围时，该命令将会返回一个错误。
# 127.0.0.1:6379> geoadd china:city 39.90 116.40 beijin
(error) ERR invalid longitude,latitude pair 39.900000,116.400000
# 参数 key 值（）
127.0.0.1:6379> geoadd china:city 116.40 39.90 beijing
(integer) 1
127.0.0.1:6379> geoadd china:city 121.47 31.23 shanghai
(integer) 1
127.0.0.1:6379> geoadd china:city 106.50 29.53 chongqi 114.05 22.52 shengzhen
(integer) 2
127.0.0.1:6379> geoadd china:city 120.16 30.24 hangzhou 108.96 34.26 xian
(integer) 2
```

### GEOPOS

```bash
127.0.0.1:6379> GEOPOS china:city beijing # 获取指定的城市的经度和纬度！
1) 1) "116.39999896287918091"
2) "39.90000009167092543"
127.0.0.1:6379> GEOPOS china:city beijing chongqi
1) 1) "116.39999896287918091"
2) "39.90000009167092543"
2) 1) "106.49999767541885376"
2) "29.52999957900659211"
```

### GEODIST

单位：

* m
* km
* mi 英里
* ft 英尺

```bash
127.0.0.1:6379> GEODIST china:city beijing shanghai km # 查看上海到北京的直线距离
"1067.3788"
127.0.0.1:6379> GEODIST china:city beijing chongqi km # 查看重庆到北京的直线距离
"1464.0708"
```

### GEORADIUS

```bash
127.0.0.1:6379> GEORADIUS china:city 110 30 1000 km # 以110，30 这个经纬度为中心，寻找方圆1000km内的城市
1) "chongqi"
2) "xian"
3) "shengzhen"
4) "hangzhou"
127.0.0.1:6379> GEORADIUS china:city 110 30 500 km
1) "chongqi"
2) "xian"
127.0.0.1:6379> GEORADIUS china:city 110 30 500 km withdist # 显示到中间距离的位置
1) 1) "chongqi"
2) "341.9374"
2) 1) "xian"
2) "483.8340"
127.0.0.1:6379> GEORADIUS china:city 110 30 500 km withcoord # 显示他人的定位信息
1) 1) "chongqi"
2) 1) "106.49999767541885376"
2) "29.52999957900659211"
2) 1) "xian"
2) 1) "108.96000176668167114"
2) "34.25999964418929977"
127.0.0.1:6379> GEORADIUS china:city 110 30 500 km withdist withcoord count 1 #筛选出指定的结果！
1) 1) "chongqi"
2) "341.9374"
3) 1) "106.49999767541885376"
2) "29.52999957900659211"
127.0.0.1:6379> GEORADIUS china:city 110 30 500 km withdist withcoord count 2
1) 1) "chongqi"
2) "341.9374"
3) 1) "106.49999767541885376"
2) "29.52999957900659211"
2) 1) "xian"
2) "483.8340"
3) 1) "108.96000176668167114"
2) "34.25999964418929977"
```

### GEORADIUSBYMEMBER

```bash
# 找出位于指定元素周围的其他元素！
127.0.0.1:6379> GEORADIUSBYMEMBER china:city beijing 1000 km
1) "beijing"
2) "xian"
127.0.0.1:6379> GEORADIUSBYMEMBER china:city shanghai 400 km
1) "hangzhou"
2) "shanghai"
```

### GEOHASH

```bash
#返回一个或多个位置元素的 Geohash 表示。
# 将二维的经纬度转换为一维的字符串，如果两个字符串越接近，那么则距离越近！
127.0.0.1:6379> geohash china:city beijing chongqi
1) "wx4fbxxfke0"
2) "wm5xzrybty0"

```

### ZSET所有命令

>  GEO 底层的实现原理其实就是 Zset！我们可以使用Zset命令来操作geo！

```bash
127.0.0.1:6379> ZRANGE china:city 0 -1 # 查看地图中全部的元素
1) "chongqi"
2) "xian"
3) "shengzhen"
4) "hangzhou"
5) "shanghai"
6) "beijing"
127.0.0.1:6379> zrem china:city beijing # 移除指定元素！
(integer) 1
127.0.0.1:6379> ZRANGE china:city 0 -1
1) "chongqi"
2) "xian"
3) "shengzhen"
4) "hangzhou"
5) "shanghai"
```

## Hyperloglog 基数

* PFADD key value value...
* PFCOUNT key
* PFMERGE key key1 key2 ...

```bash
127.0.0.1:6379> PFadd mykey a b c d e f g h i j # 创建第一组元素 mykey
(integer) 1
127.0.0.1:6379> PFCOUNT mykey # 统计 mykey 元素的基数数量
(integer) 10
127.0.0.1:6379> PFadd mykey2 i j z x c v b n m # 创建第二组元素 mykey2
(integer) 1
127.0.0.1:6379> PFCOUNT mykey2
(integer) 9
127.0.0.1:6379> PFMERGE mykey3 mykey mykey2 # 合并两组 mykey mykey2 => mykey3 并集
OK
127.0.0.1:6379> PFCOUNT mykey3 # 看并集的数量！
(integer) 15
```

优点：占用的内存是固定，2^64 不同的元素的技术，只需要废 12KB内存！如果要从内存角度来比较的话 Hyperloglog 首选！

**网页的 UV （一个人访问一个网站多次，但是还是算作一个人！）**
	传统的方式， set 保存用户的id，然后就可以统计 set 中的元素数量作为标准判断 !这个方式如果保存大量的用户id，就会比较麻烦！我们的目的是为了计数，而不是保存用户id；0.81% 错误率！ 统计UV任务，可以忽略不计的！

**如果允许容错，那么一定可以使用 Hyperloglog ！**
**如果不允许容错，就使用 set 或者自己的数据类型即可！**

## BITMAP 位存储(布隆过滤器)

* SETBIT key offset（类似数组下标） bit
* GETBIT key offset
* BITCOUNT key
* BITPOS key 0/1 查找指定值为0或1的第一个位
* BITOP 

统计用户信息，活跃，不活跃！ 登录 、 未登录！ 打卡，365打卡！ 两个状态的，都可以使用Bitmaps！

Bitmap 位图，数据结构！ 都是操作二进制位来进行记录，就只有0 和 1 两个状态！

365 天 = 365 bit 1字节 = 8bit 46 个字节左右！

```bash
bitop
# 对一个或多个保存二进制位的字符串key进行位元操作，并将结果保存到destkey上。
# operation可以是AND、OR、NOT、XOR这四种操作中的任意一种。
# BITOP AND destkey key [key ...]  ，对一个或多个key求逻辑并，并将结果保存到destkey
# BITOP OR destkey key [key ...] ，对一个或多个key求逻辑或，并将结果保存到destkey
# BITOP XOR destkey key [key ...] ，对一个或多个key求逻辑异或，并将结果保存到destkey
# BITOP NOT destkey key ，对给定key求逻辑非，并将结果保存到destkey
# 除了NOT操作外，其他操作都可以接受一个或多个key作为输入
# 当BITOP处理不同长度的字符串时，较短的那个字符串所缺少的部分会被看做0
# 空的key也被看作是包含0的字符串序列
127.0.0.1:6379> SET key1 "foobar"
OK
127.0.0.1:6379> SET key2 "abcdeff"
OK
127.0.0.1:6379> BITOP OR dest key1 key2
(integer) 7
127.0.0.1:6379> GET dest
"goofevf"
```

## 事务、乐观锁

* 开启 multi
* 入队
* 执行 exec （取消 discard）

Redis没有原子性：

语法没错的话事务中其他命令还是会执行，例如类型不匹配，除以0这种



**悲观锁：**

* 很悲观，认为什么时候都会出问题，无论做什么都会加锁！

**乐观锁：**

* 很乐观，认为什么时候都不会出问题，所以不会上锁！ 更新数据的时候去判断一下，在此期间是否有人修改过这个数据
* 获取version
* 更新的时候比较 versionRedis测监视测

```bash
127.0.0.1:6379> watch money # 监视 money
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> DECRBY money 10
QUEUED
127.0.0.1:6379> INCRBY out 10
QUEUED
127.0.0.1:6379> exec # 执行之前，另外一个线程，修改了我们的值，这个时候，就会导致事务执行失败！
(nil)

#如果事务失败，新的事务要先取消监视（unwatch），然后再次监视（watch）
```

PS：常使用setnx做分布式锁

# API

## Jedis

POM:

```xml
<dependency>
        <groupId>redis.clients</groupId>
        <artifactId>jedis</artifactId>
        <version>2.9.0</version>
</dependency>
```

```scala
object RedisUtil {

  private var jedisPool: JedisPool = _


  def getJedisClient: Jedis = {
    if (jedisPool == null) {
      println("开辟一个连接池")
      val config: Properties = PropertiesUtil.getpro("config.properties")
      val host: String = config.getProperty("redis.host")
      val port: String = config.getProperty("redis.port")

      val jedisPoolConfig = new JedisPoolConfig()
      jedisPoolConfig.setMaxTotal(100) //最大连接数
      jedisPoolConfig.setMaxIdle(20) //最大空闲
      jedisPoolConfig.setMinIdle(20) //最小空闲
      jedisPoolConfig.setBlockWhenExhausted(true) //忙碌时是否等待
      jedisPoolConfig.setMaxWaitMillis(500) //忙碌时等待时长 毫秒
      jedisPoolConfig.setTestOnBorrow(true) //每次获得连接的进行测试

      jedisPool = new JedisPool(jedisPoolConfig, host, port.toInt)
    }
//    println(s"jedisPool.getNumActive = ${jedisPool.getNumActive}")
//    println("获得一个连接")
    jedisPool.getResource
  }
}
```

## SpringBoot:

```java

import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;
@Configuration
public class RedisConfig {

// 自己定义了一个 RedisTemplate
@Bean
@SuppressWarnings("all")
public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory
factory) {
// 我们为了自己开发方便，一般直接使用 <String, Object>
RedisTemplate<String, Object> template = new RedisTemplate<String,
Object>();
template.setConnectionFactory(factory);
// Json序列化配置
Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new
Jackson2JsonRedisSerializer(Object.class);
ObjectMapper om = new ObjectMapper();
om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
jackson2JsonRedisSerializer.setObjectMapper(om);
// String 的序列化
StringRedisSerializer stringRedisSerializer = new
StringRedisSerializer();
// key采用String的序列化方式
template.setKeySerializer(stringRedisSerializer);
// hash的key也采用String的序列化方式
template.setHashKeySerializer(stringRedisSerializer);
// value序列化方式采用jackson
template.setValueSerializer(jackson2JsonRedisSerializer);
// hash的value序列化方式采用jackson
template.setHashValueSerializer(jackson2JsonRedisSerializer);
template.afterPropertiesSet();
return template;
}
} 
```

## RedisUtil

```java

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;
import org.springframework.util.CollectionUtils;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.TimeUnit;

//自己封装RedisUtil
@Component
public final class RedisUtil {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    // =============================common============================
    /**
     * 指定缓存失效时间
     * @param key  键
     * @param time 时间(秒)
     */
    public boolean expire(String key, long time) {
        try {
            if (time > 0) {
                redisTemplate.expire(key, time, TimeUnit.SECONDS);
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    /**
     * 根据key 获取过期时间
     * @param key 键 不能为null
     * @return 时间(秒) 返回0代表为永久有效
     */
    public long getExpire(String key) {
        return redisTemplate.getExpire(key, TimeUnit.SECONDS);
    }


    /**
     * 判断key是否存在
     * @param key 键
     * @return true 存在 false不存在
     */
    public boolean hasKey(String key) {
        try {
            return redisTemplate.hasKey(key);
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }


    /**
     * 删除缓存
     * @param key 可以传一个值 或多个
     */
    @SuppressWarnings("unchecked")
    public void del(String... key) {
        if (key != null && key.length > 0) {
            if (key.length == 1) {
                redisTemplate.delete(key[0]);
            } else {
                redisTemplate.delete(CollectionUtils.arrayToList(key));
            }
        }
    }


    // ============================String=============================

    /**
     * 普通缓存获取
     * @param key 键
     * @return 值
     */
    public Object get(String key) {
        return key == null ? null : redisTemplate.opsForValue().get(key);
    }
    
    /**
     * 普通缓存放入
     * @param key   键
     * @param value 值
     * @return true成功 false失败
     */

    public boolean set(String key, Object value) {
        try {
            redisTemplate.opsForValue().set(key, value);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }


    /**
     * 普通缓存放入并设置时间
     * @param key   键
     * @param value 值
     * @param time  时间(秒) time要大于0 如果time小于等于0 将设置无限期
     * @return true成功 false 失败
     */

    public boolean set(String key, Object value, long time) {
        try {
            if (time > 0) {
                redisTemplate.opsForValue().set(key, value, time, TimeUnit.SECONDS);
            } else {
                set(key, value);
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }


    /**
     * 递增
     * @param key   键
     * @param delta 要增加几(大于0)
     */
    public long incr(String key, long delta) {
        if (delta < 0) {
            throw new RuntimeException("递增因子必须大于0");
        }
        return redisTemplate.opsForValue().increment(key, delta);
    }


    /**
     * 递减
     * @param key   键
     * @param delta 要减少几(小于0)
     */
    public long decr(String key, long delta) {
        if (delta < 0) {
            throw new RuntimeException("递减因子必须大于0");
        }
        return redisTemplate.opsForValue().increment(key, -delta);
    }


    // ================================Map=================================

    /**
     * HashGet
     * @param key  键 不能为null
     * @param item 项 不能为null
     */
    public Object hget(String key, String item) {
        return redisTemplate.opsForHash().get(key, item);
    }
    
    /**
     * 获取hashKey对应的所有键值
     * @param key 键
     * @return 对应的多个键值
     */
    public Map<Object, Object> hmget(String key) {
        return redisTemplate.opsForHash().entries(key);
    }
    
    /**
     * HashSet
     * @param key 键
     * @param map 对应多个键值
     */
    public boolean hmset(String key, Map<String, Object> map) {
        try {
            redisTemplate.opsForHash().putAll(key, map);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }


    /**
     * HashSet 并设置时间
     * @param key  键
     * @param map  对应多个键值
     * @param time 时间(秒)
     * @return true成功 false失败
     */
    public boolean hmset(String key, Map<String, Object> map, long time) {
        try {
            redisTemplate.opsForHash().putAll(key, map);
            if (time > 0) {
                expire(key, time);
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }


    /**
     * 向一张hash表中放入数据,如果不存在将创建
     *
     * @param key   键
     * @param item  项
     * @param value 值
     * @return true 成功 false失败
     */
    public boolean hset(String key, String item, Object value) {
        try {
            redisTemplate.opsForHash().put(key, item, value);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    /**
     * 向一张hash表中放入数据,如果不存在将创建
     *
     * @param key   键
     * @param item  项
     * @param value 值
     * @param time  时间(秒) 注意:如果已存在的hash表有时间,这里将会替换原有的时间
     * @return true 成功 false失败
     */
    public boolean hset(String key, String item, Object value, long time) {
        try {
            redisTemplate.opsForHash().put(key, item, value);
            if (time > 0) {
                expire(key, time);
            }
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }


    /**
     * 删除hash表中的值
     *
     * @param key  键 不能为null
     * @param item 项 可以使多个 不能为null
     */
    public void hdel(String key, Object... item) {
        redisTemplate.opsForHash().delete(key, item);
    }


    /**
     * 判断hash表中是否有该项的值
     *
     * @param key  键 不能为null
     * @param item 项 不能为null
     * @return true 存在 false不存在
     */
    public boolean hHasKey(String key, String item) {
        return redisTemplate.opsForHash().hasKey(key, item);
    }


    /**
     * hash递增 如果不存在,就会创建一个 并把新增后的值返回
     *
     * @param key  键
     * @param item 项
     * @param by   要增加几(大于0)
     */
    public double hincr(String key, String item, double by) {
        return redisTemplate.opsForHash().increment(key, item, by);
    }


    /**
     * hash递减
     *
     * @param key  键
     * @param item 项
     * @param by   要减少记(小于0)
     */
    public double hdecr(String key, String item, double by) {
        return redisTemplate.opsForHash().increment(key, item, -by);
    }


    // ============================set=============================

    /**
     * 根据key获取Set中的所有值
     * @param key 键
     */
    public Set<Object> sGet(String key) {
        try {
            return redisTemplate.opsForSet().members(key);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }


    /**
     * 根据value从一个set中查询,是否存在
     *
     * @param key   键
     * @param value 值
     * @return true 存在 false不存在
     */
    public boolean sHasKey(String key, Object value) {
        try {
            return redisTemplate.opsForSet().isMember(key, value);
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }


    /**
     * 将数据放入set缓存
     *
     * @param key    键
     * @param values 值 可以是多个
     * @return 成功个数
     */
    public long sSet(String key, Object... values) {
        try {
            return redisTemplate.opsForSet().add(key, values);
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }


    /**
     * 将set数据放入缓存
     *
     * @param key    键
     * @param time   时间(秒)
     * @param values 值 可以是多个
     * @return 成功个数
     */
    public long sSetAndTime(String key, long time, Object... values) {
        try {
            Long count = redisTemplate.opsForSet().add(key, values);
            if (time > 0)
                expire(key, time);
            return count;
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }


    /**
     * 获取set缓存的长度
     *
     * @param key 键
     */
    public long sGetSetSize(String key) {
        try {
            return redisTemplate.opsForSet().size(key);
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }


    /**
     * 移除值为value的
     *
     * @param key    键
     * @param values 值 可以是多个
     * @return 移除的个数
     */

    public long setRemove(String key, Object... values) {
        try {
            Long count = redisTemplate.opsForSet().remove(key, values);
            return count;
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }

    // ===============================list=================================
    
    /**
     * 获取list缓存的内容
     *
     * @param key   键
     * @param start 开始
     * @param end   结束 0 到 -1代表所有值
     */
    public List<Object> lGet(String key, long start, long end) {
        try {
            return redisTemplate.opsForList().range(key, start, end);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }


    /**
     * 获取list缓存的长度
     *
     * @param key 键
     */
    public long lGetListSize(String key) {
        try {
            return redisTemplate.opsForList().size(key);
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }
    }


    /**
     * 通过索引 获取list中的值
     *
     * @param key   键
     * @param index 索引 index>=0时， 0 表头，1 第二个元素，依次类推；index<0时，-1，表尾，-2倒数第二个元素，依次类推
     */
    public Object lGetIndex(String key, long index) {
        try {
            return redisTemplate.opsForList().index(key, index);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }


    /**
     * 将list放入缓存
     *
     * @param key   键
     * @param value 值
     */
    public boolean lSet(String key, Object value) {
        try {
            redisTemplate.opsForList().rightPush(key, value);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }


    /**
     * 将list放入缓存
     * @param key   键
     * @param value 值
     * @param time  时间(秒)
     */
    public boolean lSet(String key, Object value, long time) {
        try {
            redisTemplate.opsForList().rightPush(key, value);
            if (time > 0)
                expire(key, time);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }

    }


    /**
     * 将list放入缓存
     *
     * @param key   键
     * @param value 值
     * @return
     */
    public boolean lSet(String key, List<Object> value) {
        try {
            redisTemplate.opsForList().rightPushAll(key, value);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }

    }


    /**
     * 将list放入缓存
     *
     * @param key   键
     * @param value 值
     * @param time  时间(秒)
     * @return
     */
    public boolean lSet(String key, List<Object> value, long time) {
        try {
            redisTemplate.opsForList().rightPushAll(key, value);
            if (time > 0)
                expire(key, time);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    /**
     * 根据索引修改list中的某条数据
     *
     * @param key   键
     * @param index 索引
     * @param value 值
     * @return
     */

    public boolean lUpdateIndex(String key, long index, Object value) {
        try {
            redisTemplate.opsForList().set(key, index, value);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    /**
     * 移除N个值为value
     *
     * @param key   键
     * @param count 移除多少个
     * @param value 值
     * @return 移除的个数
     */

    public long lRemove(String key, long count, Object value) {
        try {
            Long remove = redisTemplate.opsForList().remove(key, count, value);
            return remove;
        } catch (Exception e) {
            e.printStackTrace();
            return 0;
        }

    }

}
```

# 发布订阅

订阅端：

```bash
127.0.0.1:6379> SUBSCRIBE kuangshenshuo # 订阅一个频道 kuangshenshuo
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "kuangshenshuo"
3) (integer) 1

# 等待读取推送的信息
1) "message" # 消息
2) "kuangshenshuo" # 那个频道的消息
3) "hello,kuangshen" # 消息的具体内容

1) "message"
2) "kuangshenshuo"
3) "hello,redis"
```

发送端：

```bash
127.0.0.1:6379> PUBLISH kuangshenshuo "hello,kuangshen" # 发布者发布消息到频道！
(integer) 1
127.0.0.1:6379> PUBLISH kuangshenshuo "hello,redis" # 发布者发布消息到频道！
(integer) 1
127.0.0.1:6379>
```



# 持久化

## RDB

`dbfilename dump.rdb`

`save <seconds> <changes>`

## AOF

```bash
appendonly no # 默认是不开启aof模式的，默认是使用rdb方式持久化的，在大部分所有的情况下，rdb完全够用！
appendfilename "appendonly.aof" # 持久化的文件的名字

# appendfsync always # 每次修改都会 sync。消耗性能
appendfsync everysec # 每秒执行一次 sync，可能会丢失这1s的数据！
# appendfsync no # 不执行 sync，这个时候操作系统自己同步数据，速度最快！

# rewrite 重写，
```

优点：

1、每一次修改都同步，文件的完整会更加好！

2、每秒同步一次，可能会丢失一秒的数据

3、从不同步，效率最高的！

缺点：

1、相对于数据文件来说，aof远远大于 rdb，修复的速度也比 rdb慢！

2、Aof 运行效率也要比 rdb 慢，所以我们redis默认的配置就是rdb持久化！

## 扩展：
1、RDB 持久化方式能够在指定的时间间隔内对你的数据进行快照存储

2、AOF 持久化方式记录每次对服务器写的操作，当服务器重启的时候会重新执行这些命令来恢复原始的数据，AOF命令以Redis 协议追加保存每次写的操作到文件末尾，Redis还能对AOF文件进行后台重写，使得AOF文件的体积不至于过大。

3、只做缓存，如果你只希望你的数据在服务器运行的时候存在，你也可以不使用任何持久化

4、同时开启两种持久化方式

* 在这种情况下，当redis重启的时候会优先载入AOF文件来恢复原始的数据，因为在通常情况下AOF文件保存的数据集要比RDB文件保存的数据集要完整。
* RDB 的数据不实时，同时使用两者时服务器重启也只会找AOF文件，那要不要只使用AOF呢？作者建议不要，因为RDB更适合用于备份数据库（AOF在不断变化不好备份），快速重启，而且不会有AOF可能潜在的Bug，留着作为一个万一的手段。

5、性能建议

* 因为RDB文件只用作后备用途，建议只在Slave上持久化RDB文件，而且只要15分钟备份一次就够了，只保留 save 900 1 这条规则。
* 如果Enable AOF ，好处是在最恶劣情况下也只会丢失不超过两秒数据，启动脚本较简单只load自己的AOF文件就可以了，代价一是带来了持续的IO，二是AOF rewrite 的最后将 rewrite 过程中产生的新数据写到新文件造成的阻塞几乎是不可避免的。只要硬盘许可，应该尽量减少AOF rewrite的频率，AOF重写的基础大小默认值64M太小了，可以设到5G以上，默认超过原大小100%大小重写可以改到适当的数值。
* 如果不Enable AOF ，仅靠 Master-Slave Repllcation 实现高可用性也可以，能省掉一大笔IO，也减少了rewrite时带来的系统波动。代价是如果Master/Slave 同时倒掉，会丢失十几分钟的数据，启动脚本也要比较两个 Master/Slave 中的 RDB文件，载入较新的那个，微博就是这种架构。

# 主从复制

slave of

# 哨兵模式

sentinel配置

```bash
# Example sentinel.conf
# 哨兵sentinel实例运行的端口 默认26379
port 26379
# 哨兵sentinel的工作目录
dir /tmp
# 哨兵sentinel监控的redis主节点的 ip port
# master-name 可以自己命名的主节点名字 只能由字母A-z、数字0-9 、这三个字符".-_"组成。
# quorum 配置多少个sentinel哨兵统一认为master主节点失联 那么这时客观上认为主节点失联了
# sentinel monitor <master-name> <ip> <redis-port> <quorum>
sentinel monitor mymaster 127.0.0.1 6379 2
# 当在Redis实例中开启了requirepass foobared 授权密码 这样所有连接Redis实例的客户端都要提供密码
# 设置哨兵sentinel 连接主从的密码 注意必须为主从设置一样的验证密码
# sentinel auth-pass <master-name> <password>
sentinel auth-pass mymaster MySUPER--secret-0123passw0rd
# 指定多少毫秒之后 主节点没有应答哨兵sentinel 此时 哨兵主观上认为主节点下线 默认30秒
# sentinel down-after-milliseconds <master-name> <milliseconds>
sentinel down-after-milliseconds mymaster 30000
# 这个配置项指定了在发生failover主备切换时最多可以有多少个slave同时对新的master进行同步，这个数字越小，完成failover所需的时间就越长，但是如果这个数字越大，就意味着越 多的slave因为replication而不可用。可以通过将这个值设为 1 来保证每次只有一个slave 处于不能处理命令请求的状态。
# sentinel parallel-syncs <master-name> <numslaves>
sentinel parallel-syncs mymaster 1
# 故障转移的超时时间 failover-timeout 可以用在以下这些方面：
#1. 同一个sentinel对同一个master两次failover之间的间隔时间。
#2. 当一个slave从一个错误的master那里同步数据开始计算时间。直到slave被纠正为向正确的master那里同步数据时。
#3.当想要取消一个正在进行的failover所需要的时间。
#4.当进行failover时，配置所有slaves指向新的master所需的最大时间。不过，即使过了这个超时，slaves依然会被正确配置为指向master，但是就不按parallel-syncs所配置的规则来了
# 默认三分钟
# sentinel failover-timeout <master-name> <milliseconds>
sentinel failover-timeout mymaster 180000
# SCRIPTS EXECUTION
#配置当某一事件发生时所需要执行的脚本，可以通过脚本来通知管理员，例如当系统运行不正常时发邮件通知相关人员。
#对于脚本的运行结果有以下规则：
#若脚本执行后返回1，那么该脚本稍后将会被再次执行，重复次数目前默认为10
#若脚本执行后返回2，或者比2更高的一个返回值，脚本将不会重复执行。
#如果脚本在执行过程中由于收到系统中断信号被终止了，则同返回值为1时的行为相同。
#一个脚本的最大执行时间为60s，如果超过这个时间，脚本将会被一个SIGKILL信号终止，之后重新执行。
#通知型脚本:当sentinel有任何警告级别的事件发生时（比如说redis实例的主观失效和客观失效等等），将会去调用这个脚本，这时这个脚本应该通过邮件，SMS等方式去通知系统管理员关于系统不正常运行的信息。调用该脚本时，将传给脚本两个参数，一个是事件的类型，一个是事件的描述。如果sentinel.conf配置文件中配置了这个脚本路径，那么必须保证这个脚本存在于这个路径，并且是可执行的，否则sentinel无法正常启动成功。
#通知脚本
# shell编程
# sentinel notification-script <master-name> <script-path>
sentinel notification-script mymaster /var/redis/notify.sh
# 客户端重新配置主节点参数脚本
# 当一个master由于failover而发生改变时，这个脚本将会被调用，通知相关的客户端关于master地址已经发生改变的信息。
# 以下参数将会在调用脚本时传给脚本:
# <master-name> <role> <state> <from-ip> <from-port> <to-ip> <to-port>
# 目前<state>总是“failover”,
# <role>是“leader”或者“observer”中的一个。
# 参数 from-ip, from-port, to-ip, to-port是用来和旧的master和新的master(即旧的slave)通信的
# 这个脚本应该是通用的，能被多次调用，不是针对性的。
# sentinel client-reconfig-script <master-name> <script-path>
sentinel client-reconfig-script mymaster /var/redis/reconfig.sh # 一般都是由运维来配置！

```

# 缓存穿透和雪崩

## 缓存穿透

大并发下数据**查不到**，mysql压力过大失效（没有数据）

* 布隆过滤器
* redis缓存空

## 缓存击穿

单个key没有缓存在Redis中，大量请求转移给MySQL，造成服务宕机，例如突发事件，短时间大量请求

* 加互斥锁
* key不过期

## 缓存雪崩

redis失效或大量key没有缓存，直接访问MySQL，宕机

* redis高可用HA
* 预缓存key
* 加锁或限流

# 参数说明

```bash
# 通用 GENERAL
bind 127.0.0.1 # 绑定的ip
protected-mode yes # 保护模式
port 6379 # 端口设置
daemonize yes # 以守护进程的方式运行，默认是 no，我们需要自己开启为yes！
pidfile /var/run/redis_6379.pid # 如果以后台的方式运行，我们就需要指定一个 pid 文件！

# 日志
# Specify the server verbosity level.
# This can be one of:
# debug (a lot of information, useful for development/testing)
# verbose (many rarely useful info, but not a mess like the debug level)
# notice (moderately verbose, what you want in production probably) 生产环境
# warning (only very important / critical messages are logged)
loglevel notice
logfile "" # 日志的文件位置名
databases 16 # 数据库的数量，默认是 16 个数据库
always-show-logo yes # 是否总是显示LOGO

#快照
# 如果900s内，如果至少有一个1 key进行了修改，我们及进行持久化操作
save 900 1
# 如果300s内，如果至少10 key进行了修改，我们及进行持久化操作
save 300 10
# 如果60s内，如果至少10000 key进行了修改，我们及进行持久化操作
save 60 10000
# 我们之后学习持久化，会自己定义这个测试！

stop-writes-on-bgsave-error yes # 持久化如果出错，是否还需要继续工作！
rdbcompression yes # 是否压缩 rdb 文件，需要消耗一些cpu资源！
rdbchecksum yes # 保存rdb文件的时候，进行错误的检查校验！
dir ./ # rdb 文件保存的目录！

maxclients 10000 # 设置能连接上redis的最大客户端的数量
maxmemory <bytes> # redis 配置最大的内存容量
maxmemory-policy noeviction # 内存到达上限之后的处理策略
1、volatile-lru：只对设置了过期时间的key进行LRU（默认值）
2、allkeys-lru ： 删除lru算法的key
3、volatile-random：随机删除即将过期key
4、allkeys-random：随机删除
5、volatile-ttl ： 删除即将过期的
6、noeviction ： 永不过期，返回错误
```

密码：

```bash
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> config get requirepass # 获取redis的密码
1) "requirepass"
2) ""
127.0.0.1:6379> config set requirepass "123456" # 设置redis的密码
OK
127.0.0.1:6379> config get requirepass # 发现所有的命令都没有权限了
(error) NOAUTH Authentication required.
127.0.0.1:6379> ping
(error) NOAUTH Authentication required.
127.0.0.1:6379> auth 123456 # 使用密码进行登录！
OK
127.0.0.1:6379> config get requirepass
1) "requirepass"
2) "123456"
```



redis.conf 配置项说明如下：

| 序号 | 配置项                                                 | 说明                                                         |
| ---- | ------------------------------------------------------ | ------------------------------------------------------------ |
| 1    | `daemonize no`                                         | Redis 默认不是以守护进程的方式运行，可以通过该配置项修改，使用 yes 启用守护进程（Windows 不支持守护线程的配置为 no ） |
| 2    | `pidfile /var/run/redis.pid`                           | 当 Redis 以守护进程方式运行时，Redis 默认会把 pid 写入 /var/run/redis.pid 文件，可以通过 pidfile 指定 |
| 3    | `port 6379`                                            | 指定 Redis 监听端口，默认端口为 6379，作者在自己的一篇博文中解释了为什么选用 6379 作为默认端口，因为 6379 在手机按键上 MERZ 对应的号码，而 MERZ 取自意大利歌女 Alessia Merz 的名字 |
| 4    | `bind 127.0.0.1`                                       | 绑定的主机地址                                               |
| 5    | `timeout 300`                                          | 当客户端闲置多长时间后关闭连接，如果指定为 0，表示关闭该功能 |
| 6    | `loglevel notice`                                      | 指定日志记录级别，Redis 总共支持四个级别：debug、verbose、notice、warning，默认为 notice |
| 7    | `logfile stdout`                                       | 日志记录方式，默认为标准输出，如果配置 Redis 为守护进程方式运行，而这里又配置为日志记录方式为标准输出，则日志将会发送给 /dev/null |
| 8    | `databases 16`                                         | 设置数据库的数量，默认数据库为0，可以使用SELECT 命令在连接上指定数据库id |
| 9    | `save <seconds> <changes>`                             | 指定在多长时间内，有多少次更新操作，就将数据同步到数据文件，可以多个条件配合。Redis 默认配置文件中提供了三个条件：**save 900 1 save 300 10 save 60 10000**分别表示 900 秒（15 分钟）内有 1 个更改，300 秒（5 分钟）内有 10 个更改以及 60 秒内有 10000 个更改。 |
| 10   | `rdbcompression yes`                                   | 指定存储至本地数据库时是否压缩数据，默认为 yes，Redis 采用 LZF 压缩，如果为了节省 CPU 时间，可以关闭该选项，但会导致数据库文件变的巨大 |
| 11   | `dbfilename dump.rdb`                                  | 指定本地数据库文件名，默认值为 dump.rdb                      |
| 12   | `dir ./`                                               | 指定本地数据库存放目录                                       |
| 13   | `slaveof <masterip> <masterport>`                      | 设置当本机为 slave 服务时，设置 master 服务的 IP 地址及端口，在 Redis 启动时，它会自动从 master 进行数据同步 |
| 14   | `masterauth <master-password>`                         | 当 master 服务设置了密码保护时，slav 服务连接 master 的密码  |
| 15   | `requirepass foobared`                                 | 设置 Redis 连接密码，如果配置了连接密码，客户端在连接 Redis 时需要通过 AUTH <password> 命令提供密码，默认关闭 |
| 16   | ` maxclients 128`                                      | 设置同一时间最大客户端连接数，默认无限制，Redis 可以同时打开的客户端连接数为 Redis 进程可以打开的最大文件描述符数，如果设置 maxclients 0，表示不作限制。当客户端连接数到达限制时，Redis 会关闭新的连接并向客户端返回 max number of clients reached 错误信息 |
| 17   | `maxmemory <bytes>`                                    | 指定 Redis 最大内存限制，Redis 在启动时会把数据加载到内存中，达到最大内存后，Redis 会先尝试清除已到期或即将到期的 Key，当此方法处理 后，仍然到达最大内存设置，将无法再进行写入操作，但仍然可以进行读取操作。Redis 新的 vm 机制，会把 Key 存放内存，Value 会存放在 swap 区 |
| 18   | `appendonly no`                                        | 指定是否在每次更新操作后进行日志记录，Redis 在默认情况下是异步的把数据写入磁盘，如果不开启，可能会在断电时导致一段时间内的数据丢失。因为 redis 本身同步数据文件是按上面 save 条件来同步的，所以有的数据会在一段时间内只存在于内存中。默认为 no |
| 19   | `appendfilename appendonly.aof`                        | 指定更新日志文件名，默认为 appendonly.aof                    |
| 20   | `appendfsync everysec`                                 | 指定更新日志条件，共有 3 个可选值：**no**：表示等操作系统进行数据缓存同步到磁盘（快）**always**：表示每次更新操作后手动调用 fsync() 将数据写到磁盘（慢，安全）**everysec**：表示每秒同步一次（折中，默认值） |
| 21   | `vm-enabled no`                                        | 指定是否启用虚拟内存机制，默认值为 no，简单的介绍一下，VM 机制将数据分页存放，由 Redis 将访问量较少的页即冷数据 swap 到磁盘上，访问多的页面由磁盘自动换出到内存中（在后面的文章我会仔细分析 Redis 的 VM 机制） |
| 22   | `vm-swap-file /tmp/redis.swap`                         | 虚拟内存文件路径，默认值为 /tmp/redis.swap，不可多个 Redis 实例共享 |
| 23   | `vm-max-memory 0`                                      | 将所有大于 vm-max-memory 的数据存入虚拟内存，无论 vm-max-memory 设置多小，所有索引数据都是内存存储的(Redis 的索引数据 就是 keys)，也就是说，当 vm-max-memory 设置为 0 的时候，其实是所有 value 都存在于磁盘。默认值为 0 |
| 24   | `vm-page-size 32`                                      | Redis swap 文件分成了很多的 page，一个对象可以保存在多个 page 上面，但一个 page 上不能被多个对象共享，vm-page-size 是要根据存储的 数据大小来设定的，作者建议如果存储很多小对象，page 大小最好设置为 32 或者 64bytes；如果存储很大大对象，则可以使用更大的 page，如果不确定，就使用默认值 |
| 25   | `vm-pages 134217728`                                   | 设置 swap 文件中的 page 数量，由于页表（一种表示页面空闲或使用的 bitmap）是在放在内存中的，，在磁盘上每 8 个 pages 将消耗 1byte 的内存。 |
| 26   | `vm-max-threads 4`                                     | 设置访问swap文件的线程数,最好不要超过机器的核数,如果设置为0,那么所有对swap文件的操作都是串行的，可能会造成比较长时间的延迟。默认值为4 |
| 27   | `glueoutputbuf yes`                                    | 设置在向客户端应答时，是否把较小的包合并为一个包发送，默认为开启 |
| 28   | `hash-max-zipmap-entries 64 hash-max-zipmap-value 512` | 指定在超过一定的数量或者最大的元素超过某一临界值时，采用一种特殊的哈希算法 |
| 29   | `activerehashing yes`                                  | 指定是否激活重置哈希，默认为开启（后面在介绍 Redis 的哈希算法时具体介绍） |
| 30   | `include /path/to/local.conf`                          | 指定包含其它的配置文件，可以在同一主机上多个Redis实例之间使用同一份配置文件，而同时各个实例又拥有自己的特定 |