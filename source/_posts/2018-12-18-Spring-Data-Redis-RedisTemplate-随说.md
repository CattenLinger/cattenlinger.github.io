---
title: Spring Data Redis - RedisTemplate 随说
date: 2018-12-18 07:12:50
tags:
- 'Spring'
- 'Java'
---
一直用 Spring Data Redis 偷懒了不少次，但遇到问题了才明白了这个玩意到底怎么回事。

## 它干了啥

首先，RedisTemplate 封装了一系列 Redis 操作，常用的 Key-Value 操作、HashMap 操作以及数组操作等等，直接操作 Jedis 跟操作它其实是没啥两样的，不过既然能叫 Template 了，也就是说它考虑了很多常用的操作。（稍微说一下 Spring 对不同的数据库都有 Template 例如 JdbcTemplate 封装 SQL 常用草组，MongoTemplate 封装 MongoDB 常用操作）。

然后，它里面还管理了序列化：Key、Value 在 Redis 里面都是 byte ，所以需要把 Java 的对象序列化成 byte 数组。

Redis 支持事务[*](https://redis.io/topics/transactions) ，所以 RedisTemplate 也提供了事务管理。

查看它到底自己存储了什么东西就知道了

```java
public class RedisTemplate<K, V> extends RedisAccessor implements RedisOperations<K, V>, BeanClassLoaderAware {

	private boolean enableTransactionSupport = false;
	private boolean exposeConnection = false;
	private boolean initialized = false;
	private boolean enableDefaultSerializer = true;
	private @Nullable RedisSerializer<?> defaultSerializer;
	private @Nullable ClassLoader classLoader;

	@SuppressWarnings("rawtypes") private @Nullable RedisSerializer keySerializer = null;
	@SuppressWarnings("rawtypes") private @Nullable RedisSerializer valueSerializer = null;
	@SuppressWarnings("rawtypes") private @Nullable RedisSerializer hashKeySerializer = null;
	@SuppressWarnings("rawtypes") private @Nullable RedisSerializer hashValueSerializer = null;
	private RedisSerializer<String> stringSerializer = new StringRedisSerializer();

	private @Nullable ScriptExecutor<K> scriptExecutor;

	// cache singleton objects (where possible)
	private @Nullable ValueOperations<K, V> valueOps;
	private @Nullable ListOperations<K, V> listOps;
	private @Nullable SetOperations<K, V> setOps;
	private @Nullable ZSetOperations<K, V> zSetOps;
	private @Nullable GeoOperations<K, V> geoOps;
	private @Nullable HyperLogLogOperations<K, V> hllOps;
}
```

看，其实很简单，相信很多人在用 Redis 的时候都做过类似的东西去处理相关操作。

## 操作（Operations）

在 RedisTemplate 有不少自己的 operation domain（操作概念），调用对应的方法就能获取一个 domain operator（操作对象），方便使用者去做不同的操作：

- opsForValue - 操作 Key - Value ，简单的键值对
- opsForList - 操作列表
- opsForSet - 操作集合（set）
- opsForZSet - 操作有序集合（sorted set，在 Redis 内叫 zSet）
- opsForHash - 操作 Map 集合
- opsForGeo - 操作地理信息，可以当作 HashSet 看，不过里面的 Key 是二维地理坐标
- opsForHyperLogLog - 操作 HyperLogLog 值，是 counter 。
- opsForCluster - 操作集群，管理集群用。

稍微说一下 HyperLogLog ，给它塞数据进去就给你 count 已经有多少个，因为是基于“基数估计”所以不会很准确。用于大客流计算。

除了 HyperLogLog 和 Cluster 两个 domain，其他都有对应的 bounds（绑定操作），用于绑定到某个键，例如这两个操作可以互换：

```java
opsForValue().set("key","value");
boundValueOps("key").set("value"); // 不要这样操作，会产生很多 bounded operator（绑定操作对象）
```

允许这样做是为了保留上下文信息方便使用。其实 domain operator 和 bounded domain operator 都是用来保留上下文信息的对象，前者保留 redisTemplate 的上下文信息，后者则是前者加上 key 上下文信息，你可以把它保留并传递下去，这样就不需要直接暴露 redisTemplate 增加无谓的设计上的麻烦了（最小责任原则）。

## 序列化

因为 Java 是个强类型语言，所以序列化/反序列化都需要有类型信息，而类型信息怎么存储就是个问题了。

每个  RedisTemplate 都有自己的序列化器。你是可以持有多个 RedisTemplate 的。使用多个 RedisTemplate 实例会方便存储各种类型信息（当然也是可以存储在持久化到 Redis 的数据里面的，只是我不喜欢这么干），看看泛型传参就知道了。

每个 Template 都有四个字段存储序列化器

- keySerializer - key 序列化器
- valueSerializer - 值序列化器
- hashKeySerializer - 用于 Hash 操作的 key 序列化器
- hashValueSerializer - 用于 Hash 操作的 value 序列化器

默认情况下如果不设置， 各个序列化器会直接使用 defaultSerializer ，如果不全部都指定的话，建议设置一个 defaultSerializer 。这个行为是在 RedisTempalte 的 `afterPropertiesSet()` 方法里的。**新建完一个 RedisTemplate 必须调用一次这个方法，不然会报错叫你再调用一次**，我就是踩过这个坑（

序列化器都需要实现接口 RedisSerializer ，这个类实在很简单，就两个方法，序列化和反序列化。直接贴源码

```java
/**
 * Basic interface serialization and deserialization of Objects to byte arrays (binary data). It is recommended that
 * implementations are designed to handle null objects/empty arrays on serialization and deserialization side. Note that
 * Redis does not accept null keys or values but can return null replies (for non existing keys).
 *
 * @author Mark Pollack
 * @author Costin Leau
 * @author Christoph Strobl
 */
public interface RedisSerializer<T> {

	/**
	 * Serialize the given object to binary data.
	 *
	 * @param t object to serialize. Can be {@literal null}.
	 * @return the equivalent binary data. Can be {@literal null}.
	 */
	@Nullable
	byte[] serialize(@Nullable T t) throws SerializationException;

	/**
	 * Deserialize an object from the given binary data.
	 *
	 * @param bytes object binary representation. Can be {@literal null}.
	 * @return the equivalent object instance. Can be {@literal null}.
	 */
	@Nullable
	T deserialize(@Nullable byte[] bytes) throws SerializationException;
}
```

十分宽松噢，能抛异常还能抛 null （wwwwwww

如果你不想这么麻烦，那直接用官方提供的两个默认 Serializer 就好了：

- StringRedisSerializer - 直接序列化成 String ，只支持 String 。
- GenericJackson2JsonRedisSerializer - 序列化成 Json String ，啥都能序列化。

一般来说这两个就够用了。

GenericJackson2JsonRedisSerializer 有一点要说明一下，它做了些工作。

1. 它会对 null 值作特殊处理，内带一个自己的 NullValueSerializer ，在创建的时候会注册到自己的 ObjectMapper 上（就是 Jackson 的 ObjectMapper），把 Null 值序列化成 Spring 的 Cache 专用的 `NullValue.class` 。所以如果这个 Feature 对你有害，考虑对照它自己写一个新的。
2. 它默认会把类型信息同时写入 Redis ，方便兼容各种 Class 的 instance 的序列化，但我自己在用的时候发现输出到 Http Body 的 JSON 会带上 `@class` 字段，容易泄漏代码信息，想了几个办法都去不掉。所以如果这个对你有影响的话，自己实现一个比较好。

我自己方便使用的原因，所以每个拥有自己的 scope 的用到 Redis 的实例都带有类型信息而且单独拥有一个 RedisTemplate 并单独拥有 ValueSerializer。这里贴个我自己轮子的代码作示范，主要在里面的 valueSerializer 。

```java
public abstract class InMemoryStoreAdapter<T> implements InMemoryStore {
	private final static byte[] EMPTY_ARRAY = new byte[0];
    
    // ......其他无关紧要的东西
    
    protected final RedisTemplate<String, T> redisTemplate;
    
    // 所有的子类都使用同一个 KeySerializer 实例
    private final static RedisSerializer<String> keySerializer = new StringRedisSerializer();

    // 所有的子类都使用同一个 ObjectMapper 实例
    private final static ObjectMapper objectMapper = new ObjectMapper();
    
    // 构造器
    protected InMemoryStoreAdapter(RedisConnectionFactory redisConnectionFactory, String domain) {
        // 这里新建一个 RedisTemplate 给自己用
        this.redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory);

        // 获取自己的泛型参数类型信息，自己存储。
        Type mainType = getClass().getGenericSuperclass();
        ParameterizedType parameterizedType = (ParameterizedType) mainType;
        Class<T> mainTypeClass = (Class<T>) parameterizedType.getActualTypeArguments()[0];

        // 每个子类的 redisTemplate 都有自己的 ValueSerializer ，我直接把它写成匿名类了。
        RedisSerializer<T> valueSerializer = new RedisSerializer<T>() {
            // ObjectReader 其实也是个上下文存储对象，里面包含了类型信息，
            // 用的 ObjectMapper 还是原来的那个
            private final ObjectReader objectReader = objectMapper.readerFor(mainTypeClass);
            private final ObjectWriter objectWriter = objectMapper.writer();

            @Override
            public byte[] serialize(T t) throws SerializationException {
                try {
                    return t == null ? EMPTY_ARRAY : objectWriter.writeValueAsBytes(t);
                } catch (JsonProcessingException e) {
                    throw new SerializationException("Could not write object to JSON: ", e);
                }
            }

            @Override
            public T deserialize(byte[] bytes) throws SerializationException {
                try {
                    return (bytes == null || bytes.length <= 0) ? null : objectReader.readValue(bytes);
                } catch (IOException e) {
                    throw new SerializationException("Could not read object from JSON: ", e);
                }
            }
        };

        // 设定各个序列化器
        redisTemplate.setDefaultSerializer(valueSerializer);

        redisTemplate.setKeySerializer(keySerializer);
        redisTemplate.setHashKeySerializer(keySerializer);

        redisTemplate.setValueSerializer(valueSerializer);

        // 记得调用 afterPropertiesSet()
        redisTemplate.afterPropertiesSet();

        // ......其他代码
    }
}
```

## 唠叨话

要自己实现一个 redisTemplate 也不难，但重复造轮子实际上是一种浪费时间的做法，如果已经有了一个不难用的轮子，直接拿来用就好了，they are life saver。不过，在不想和 Spring 有依赖的场合里，做一个小轮子还是有必要的。