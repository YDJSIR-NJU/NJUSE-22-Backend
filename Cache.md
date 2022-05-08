# Cache

## Redis

### 特点

快速读取、数据安全

key-value的Hash表结构，value是某数据结构 

内存数据库（缓存）

集群

主从（master/slave）复制

数据持久化

注意key、value区分大小写

### 数据类型

https://www.runoob.com/redis/redis-data-types.html

String

Linked Lists

+ 队列lpush/rpop

+ 阻塞等待：BRPOP和BLPOP

Hashes

Sets

### Java使用

会将数据序列化，所以redis内部是看不出来是什么东西的。Product类要实现序列化接口

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisConnectionFactory redisCF() {
        return new JedisConnectionFactory();
    }

    @Bean
    public RedisTemplate<String, Product> redisTemplate(RedisConnectionFactory cf) {
        RedisTemplate<String, Product> redis = new RedisTemplate<>();
        redis.setConnectionFactory(cf);
        return redis;
    }

}
```

```java
public class CartTest {

    /*
     * IMPORTANT: This test class requires that a Redis server be running on
     *            localhost and listening on port 6379 (the default port).
     */

    @Autowired
    private RedisConnectionFactory cf;

    @Autowired
    private RedisTemplate<String, Product> redis;

    @After
    public void cleanUp() {
        redis.delete("9781617291203");
        redis.delete("cart");
        redis.delete("cart1");
        redis.delete("cart2");
    }

    @Test
    public void workingWithSimpleValues() { // 设置一个key-value值
        Product product = new Product();
        product.setSku("9781617291203");
        product.setName("Spring in Action");
        product.setPrice(39.99f);

        redis.opsForValue().set(product.getSku(), product);

        Product found = redis.opsForValue().get(product.getSku());
        assertEquals(product.getSku(), found.getSku());
        assertEquals(product.getName(), found.getName());
        assertEquals(product.getPrice(), found.getPrice(), 0.005);
    }

    @Test
    public void workingWithLists() { // 添加list
        Product product = new Product();
        product.setSku("9781617291203");
        product.setName("Spring in Action");
        product.setPrice(39.99f);

        Product product2 = new Product();
        product2.setSku("9781935182436");
        product2.setName("Spring Integration in Action");
        product2.setPrice(49.99f);

        Product product3 = new Product();
        product3.setSku("9781935182955");
        product3.setName("Spring Batch in Action");
        product3.setPrice(49.99f);

        redis.opsForList().rightPush("cart", product);
        redis.opsForList().rightPush("cart", product2);
        redis.opsForList().rightPush("cart", product3);

        assertEquals(3, redis.opsForList().size("cart").longValue());

        Product first = redis.opsForList().leftPop("cart");
        Product last = redis.opsForList().rightPop("cart");

        assertEquals(product.getSku(), first.getSku());
        assertEquals(product.getName(), first.getName());
        assertEquals(product.getPrice(), first.getPrice(), 0.005);

        assertEquals(product3.getSku(), last.getSku());
        assertEquals(product3.getName(), last.getName());
        assertEquals(product3.getPrice(), last.getPrice(), 0.005);

        assertEquals(1, redis.opsForList().size("cart").longValue());
    }

    @Test
    public void performingOperationsOnSets_setOperations() { // 添加Set
        for (int i = 0; i < 30; i++) {
            Product product = new Product();
            product.setSku("SKU-" + i);
            product.setName("PRODUCT " + i);
            product.setPrice(i + 0.99f);
            redis.opsForSet().add("cart1", product);
            if (i % 3 == 0) {
                redis.opsForSet().add("cart2", product);
            }
        }

        Set<Product> diff = redis.opsForSet().difference("cart1", "cart2");
        Set<Product> union = redis.opsForSet().union("cart1", "cart2");
        Set<Product> isect = redis.opsForSet().intersect("cart1", "cart2");

        assertEquals(20, diff.size());
        assertEquals(30, union.size());
        assertEquals(10, isect.size());

        Product random = redis.opsForSet().randomMember("cart1");
        // not sure what to assert here...the result will be random
        assertNotNull(random);
    }



}
```

### Java序列化

#### 代码

自己定义序列化，让自己可以在redis里面读懂存的东西

默认处理：JdkSerializationRedisSerializer

key：StringRedisSerializer

Value：Jackson2JsonRedisSerializer

```java
@Test
public void settingKeyAndValueSerializers() { // 序列化
    // need a local version so we can tweak the serializer
    RedisTemplate<String, Product> redis = new RedisTemplate<>();
    redis.setConnectionFactory(cf);

    redis.setKeySerializer(new StringRedisSerializer());
    redis.setValueSerializer(new Jackson2JsonRedisSerializer<Product>(Product.class));
    redis.afterPropertiesSet(); // if this were declared as a bean, you wouldn't have to do this

    Product product = new Product();
    product.setSku("9781617291203");
    product.setName("Spring in Action");
    product.setPrice(39.99f);

    redis.opsForValue().set(product.getSku(), product);

    Product found = redis.opsForValue().get(product.getSku());
    assertEquals(product.getSku(), found.getSku());
    assertEquals(product.getName(), found.getName());
    assertEquals(product.getPrice(), found.getPrice(), 0.005);

    StringRedisTemplate stringRedis = new StringRedisTemplate(cf);
    String json = stringRedis.opsForValue().get(product.getSku());
    assertEquals("{\"sku\":\"9781617291203\",\"name\":\"Spring in Action\",\"price\":39.99}", json);
}
```

## EhCache

### 概念

CacheManager：Cache的容器对象，并管理着（添加或删除）Cache的生命周期

Cache: 一个Cache可以包含多个Element，并被CacheManager管理。它实现了对缓存的逻辑行为

Element：需要缓存的元素，它维护着一个键值对， 元素也可以设置有效期，0代表无限制

### Spring使用缓存

#### CacheManager配置

org.springframework.cache.concurrent.ConcurrentMapCacheManager（支持JDK的原生缓存）

org.springframework.cache.ehcache.EhCacheCacheManager（支持数据持久化）

org.springframework.data.redis.cache.RedisCacheManager（远程储存，分布式适用）

#### 启用缓存

@EnableCaching

+ aspect、pointcut

缓存管理器，与缓存实现集成：org.springframework.cache.CacheManager

#### 具体方法缓存注解

@Cacheable：调用方法之前，先在缓存中找方法的返回值

@CachePut：直接调用方法，在调用完之后将返回值存到缓存中。

@CacheEvict：清除缓存。