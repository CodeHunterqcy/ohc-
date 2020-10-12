

# 背景

<font color=#999AAA >
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;互联网项目中，一般是以堆内缓存作为首要选择，但是不论是是Guava，Caffeine缓存，还是HashMap，ConcurrentHashMap等，都是在依靠JVM在堆内内存中做存取淘汰操作，所以随着需求越来越大，堆内的压力也就随之增加，由于GC的存在，大量的存取引起了频繁的GC操作，堆内缓存操作的ops会受到不小的影响，造成原本小流量下10ms能够完成的内存计算，大流量下100ms甚至更长时间还未完成，产生大量的慢请求。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;而在推荐系统中，由推荐引擎提供线上推荐服务。每个阶段比如召回、排序等都需要大量的数据支撑，所以如何去快速读取这些数据对推荐引擎的性能起着关键性作用。
</font>

<br>
PS：本文涉及的关键点和敏感数据均已省略简写，请各位见谅。
<hr style=" border:solid; width:100px; height:1px;" color=#000000 size=1">


# 一、方案选取和试验
<font color=#999AAA > 推荐引擎需要大量数据支撑，并且要求在短时间内完成处理。OHC具有低延迟、容量大、不影响GC的特性，并且支持使用方根据自身业务需求进行灵活配置，因此我们选用OHC作为缓存框架，将堆内的数据搬到堆外。

## 1. 纯使用堆外做缓存

首先我们应该知道：
 - 堆外缓存是不受JVM管控的，所以也不受GC的影响导致的应用暂停问题。但是由于堆外缓存的使用，是以byte数组来进行的，需要自己取序列化反序列化操作，但是即便选取了高性能的序列化手段，速度也是比不上堆内存取的速度，这是其一；
 - 堆外内存放任依靠ohc管理，是不放心的，所以我们需要一个有较高准度的监控系统来监控堆外容量、命中率、回收数等硬性指标；
   当然还有其他问题，我目前还没想到；（开发完监控，我们开始试验）

纯使用堆外缓存，并发数为40进行压测（数值参考线上平均值）

平均RT很高，如果对于实时性要求不高的系统，可以接受，但是我们无法使用。

所以类比传统的DB+redis方案，做堆外+堆内组合使用，并且开发全透明化的哨兵监控，才能进行下一步的调优；

   
## 2. 堆内+堆外多级缓存

 - 首先开发全程透明的准确度高的监控，制定相关指标；
 - 然后选取合适的淘汰策略（至关重要的，踩了大坑）
 - 然后我们知道，如果堆内容量小，大量请求势必会穿透堆内，GC压力倒是缓解了，但是造成平均RT远远高于线上；如果堆内容量较大，那么堆外可能会形同虚设；如何选取，需要结合自己的系统，依照put的总量去调优，这时候监控就大显身手，调整到线上满意的效果就可以上线做灰度试验。



思路图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201012221808777.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQzMzkwMjM1,size_16,color_FFFFFF,t_70#pic_center)
这是简化的请求逻辑图：
逻辑很简单，一句话就可以说清楚：
先去查堆内，未命中再去查堆外，查到了回写到堆内并且返回结果；

利用缓存策略加上不断的替换数据

使得堆内缓存维持比较热的数据，这样既缓解了GC压力，又降低平均RT。
```java
    public V get(K key) {
        checkNotNull(key);
        V res;
        //先去堆内查询，未命中再去堆外查询
        if (null != onCache && (res = onCache.getIfPresent(key)) != null) {
            return res;
        }
        if (!keySet.contains(key)) {
            return null;
        }
        V result = ohCache.get(key);
        //如果开启堆内并且查询到结果，写入堆内
        if (result != null && onCache != null) {
            onCache.put(key, result);
        } else if (result == null) {
            keySet.remove(key);
        }
        return result;
    }
```

 




<br><br>

# 二、开发过程

## 1. 改造ohc
[ohc的GitHub](https://github.com/snazy/ohc)


改造结果：
 - 以jar包形式在项目中使用，可以应用在各种大量使用堆内缓存的场景，均可使用；
 - 组合guava cache（也可以使用Caffeine缓存作为堆内缓存），采用链式builder设计，可根据项目需要自己选取是否组合堆内缓存；
 - 内置哨兵对请求全程监控（请求数量、堆内缓存命中率、堆外缓存命中率等等）
 
## 2. 非敏感代码（仅供参考）
改造后的ohc
```java
public class OffHeapCache<K, V> implements Cache<K, V> {

    private static final Logger logger = LoggerFactory.getLogger(OffHeapCache.class);

    public final OHCache<K, V> ohCache;
    public final Set<K> keySet;
    public final ThreadPoolExecutor threadPool;
    public final boolean batchOperation;
    public final boolean log;
    public final int maxSize;
    public String cacheKey;
    //堆内缓存
    public final Cache<K, V> onCache;

    private OffHeapCache(OHCache<K, V> ohCache, Set<K> keySet, int maxSize, ThreadPoolExecutor threadPool,
                         boolean batchOperation, boolean log, String cacheKey, com.google.common.cache.Cache<K, V> onCache) {
        checkNotNull(ohCache);
        checkNotNull(keySet);
        this.ohCache = ohCache;
        this.keySet = keySet;
        this.threadPool = batchOperation ? (threadPool == null ?
                new ThreadPoolExecutor(20, 20, 1000L, TimeUnit.SECONDS,
                        new LinkedBlockingQueue<>(100), new ThreadPoolExecutor.DiscardPolicy()) : threadPool) : null;
        this.maxSize = maxSize;
        this.batchOperation = batchOperation;
        this.log = log;
        this.cacheKey = cacheKey;
        this.onCache = onCache;
    }

    public OffHeapCache(OHCache<K, V> cache, int maxSize, String cacheKey) {
        this(cache, maxSize, cacheKey, null);
    }


    public OffHeapCache(OHCache<K, V> cache, int maxSize, String cacheKey, com.google.common.cache.Cache<K, V> onCache) {
        this(cache, maxSize, false, cacheKey, onCache);
    }

    public OffHeapCache(OHCache<K, V> cache, int maxSize, boolean batchOperation, String cacheKey, com.google.common.cache.Cache<K, V> onCache) {
        this(cache, maxSize, batchOperation, false, cacheKey, onCache);
    }

    public OffHeapCache(OHCache<K, V> cache, int maxSize, boolean batchOperation, boolean log, String cacheKey, com.google.common.cache.Cache<K, V> onCache) {
        this(cache, maxSize, null, batchOperation, log, cacheKey, onCache);
    }

    public OffHeapCache(OHCache<K, V> cache, int maxSize,
                        int corePoolSize, int maxPoolSize,
                        long keepAliveTime, TimeUnit unit,
                        BlockingQueue<Runnable> blockingQueue, RejectedExecutionHandler policy, String cacheKey, com.google.common.cache.Cache<K, V> onCache) {
        this(cache, maxSize, new ThreadPoolExecutor(corePoolSize, maxPoolSize, keepAliveTime, unit, blockingQueue, policy),
                true, false, cacheKey, onCache);
    }

    public OffHeapCache(OHCache<K, V> cache, int maxSize,
                        int corePoolSize, int maxPoolSize,
                        long keepAliveTime, TimeUnit unit,
                        BlockingQueue<Runnable> blockingQueue, RejectedExecutionHandler policy, boolean log, String cacheKey, com.google.common.cache.Cache<K, V> onCache) {
        this(cache, maxSize, new ThreadPoolExecutor(corePoolSize, maxPoolSize, keepAliveTime, unit, blockingQueue, policy),
                true, log, cacheKey, onCache);
    }

    public OffHeapCache(OHCache<K, V> cache, int maxSize, ThreadPoolExecutor threadPool, String cacheKey, com.google.common.cache.Cache<K, V> onCache) {
        this(cache, maxSize, threadPool, false, cacheKey, onCache);
    }

    public OffHeapCache(OHCache<K, V> cache, int maxSize, ThreadPoolExecutor threadPool, boolean log, String cacheKey, com.google.common.cache.Cache<K, V> onCache) {
        this(cache, maxSize, threadPool, true, log, cacheKey, onCache);
    }

    private OffHeapCache(OHCache<K, V> cache, int maxSize, ThreadPoolExecutor threadPool, boolean batchOperation, boolean log, String cacheKey, com.google.common.cache.Cache<K, V> onCache) {
        this(cache, new ConcurrentHashSet<>(maxSize), maxSize, threadPool, batchOperation, log, cacheKey, onCache);
    }

    @Override
    public V get(K key) {
        checkNotNull(key);
        V res;
        //先去堆内查询，未命中再去堆外查询
        if (null != onCache && (res = onCache.getIfPresent(key)) != null) {
            return res;
        }
        if (!keySet.contains(key)) {
            return null;
        }
        V result = ohCache.get(key);
        //如果开启堆内并且查询到结果，写入堆内
        if (result != null && onCache != null) {
            onCache.put(key, result);
        } else if (result == null) {
            keySet.remove(key);
        }
        return result;
    }

    @Override
    public Map<K, V> getAll(List<K> keys) {
        checkNotNull(keys);
        Map<K, V> result = new HashMap<>(keys.size());
        for (K key : keys) {
            result.put(key, get(key));
        }
        return result;
    }

    @Override
    public boolean put(K key, V value) {
        checkNotNull(key);
        checkNotNull(value);
        //更新文章先全部放到堆外
        if (ohCache.put(key, value)) {
            keySet.add(key);
            if (log) logger.info("ohc put: key = " + key.toString() + ";value = " + value.toString());
            return true;
        } else {
            if (log) logger.info("ohc put failed");
            return false;
        }
    }

    @Override
    public void putAll(List<KeyValuePair<K, V>> pairs) {
        checkNotNull(pairs);
        for (KeyValuePair<K, V> pair : pairs) {
            put(pair.getKey(), pair.getValue());
        }
    }

    @Override
    public boolean remove(K key) {
        checkNotNull(key);
        if (!keySet.contains(key)) {
            return true;
        }
        keySet.remove(key);
        if (onCache != null) {

            onCache.invalidate(key);
        }
        if (ohCache.remove(key)) {
            return true;
        } else {
            keySet.add(key);
            return false;
        }
    }

    @Override
    public void removeAll(List<K> keys) {
        checkNotNull(keys);
        for (K key : keys) {
            remove(key);
        }
    }

    @Override
    public void clear() {
        keySet.clear();
        ohCache.clear();
        if (onCache != null) {
            onCache.invalidateAll();
        }
    }


    @Override
    public boolean containsKey(K key) {
        return keySet.contains(key);
    }

    @Override
    public long freeSpace() {
        return ohCache.freeCapacity();
    }

    @Override
    public long size() {
        return ohCache.size();
    }

    /**
     * Ensures that an object reference passed as a parameter to the calling method is not null.
     *
     * @param reference an object reference
     * @throws NullPointerException if {@code reference} is null
     */
    public static <T> void checkNotNull(T reference) {
        if (reference == null) {
            throw new NullPointerException();
        }
    }

}

```



我们可以自由去组合堆内缓存（是否开启，大小，过期时间等）

```java
   /**
     * 开启堆内缓存
     * @param size
     * @param duration
     * @param unit
     * @return
     */
    public OffHeapCacheBuilder<K, V> withOnCache(long size, long duration, TimeUnit unit) {
        onCache = CacheBuilder.newBuilder().recordStats().maximumSize(size)
                .expireAfterAccess(duration, unit).build();
        return this;
    }
```
只需要在builder构造时：

```java
//加上这个就会开启
withOnCache(10000, 6, TimeUnit.HOUR).build();
```
并且会自动开启堆内缓存监控
```java
        if (cache != null) {
            //添加到堆外监控map
            OHCStateCollector.ohcCacheMap.put(cacheKey, (OffHeapCache) cache);
            //开启OHC监控
            OHCStateCollector.startMonitor(cacheKey);
            if (onCache != null) {
                //添加到堆内监控map
                OnCacheStateCollector.onCacheMap.put(cacheKey, (OffHeapCache) cache);
                //开启堆内监控
                OnCacheStateCollector.startMonitor(cacheKey);
            }
        }

```
builder：
```java
public class OffHeapCacheBuilder<K, V> {

    private OHCacheBuilder<K, V> ohCacheBuilder;
    // 序列化器
    private CacheSerializer<K> keySerializer = new DefaultKryoSerializer<>();
    private CacheSerializer<V> valueSerializer = new DefaultKryoSerializer<>();
    // 缓存设置
    private boolean timeouts = false;
    private long defaultTTLmillis = Long.MAX_VALUE;
    private int hashTableSize = 8192;
    private int segmentCount;
    private long capacity;
    private long maxEntrySize = 2048;
    private Eviction eviction = Eviction.LRU;
    private boolean log = false;
    private boolean batchOperation = false;
    private int maxSize;
    private HashAlgorithm hashAlgorithm = HashAlgorithm.CRC32;
    // 线程池
    private ThreadPoolExecutor threadPool = null;
	//文章缓存池唯一标识名称（监控对应）
    private String cacheKey;

    private Cache<K, V> onCache = null;


    private OffHeapCacheBuilder() {
        int cpus = Runtime.getRuntime().availableProcessors();
        segmentCount = roundUpToPowerOf2(cpus * 2, 1 << 30);
        capacity = Math.min(cpus * 16, 64) * 1024 * 1024;
        segmentCount = cpus * 2;
        ohCacheBuilder = OHCacheBuilder.newBuilder();
        maxSize = segmentCount * hashTableSize;
    }

    public static <K, V> OffHeapCacheBuilder<K, V> newBuilder() {
        return new OffHeapCacheBuilder<>();
    }

    public OffHeapCacheBuilder<K, V> keySerializer(CacheSerializer<K> keySerializer) {
        this.keySerializer = keySerializer;
        return this;
    }

    public OffHeapCacheBuilder<K, V> valueSerializer(CacheSerializer<V> valueSerializer) {
        this.valueSerializer = valueSerializer;
        return this;
    }

    public OffHeapCacheBuilder<K, V> threadPool(ThreadPoolExecutor threadPool) {
        this.threadPool = threadPool;
        this.batchOperation = true;
        return this;
    }

    public OffHeapCacheBuilder<K, V> segmentCount(int segmentCount) {
        this.segmentCount = segmentCount;
        return this;
    }

    public OffHeapCacheBuilder<K, V> defaultTTLmillis(long defaultTTLmillis) {
        this.defaultTTLmillis = defaultTTLmillis;
        return this;
    }

    public OffHeapCacheBuilder<K, V> hashTableSize(int hashTableSize) {
        this.hashTableSize = hashTableSize;
        return this;
    }

    public OffHeapCacheBuilder<K, V> maxEntrySize(long maxEntrySize) {
        this.maxEntrySize = maxEntrySize;
        return this;
    }

    public OffHeapCacheBuilder<K, V> capacity(int capacity) {
        this.capacity = capacity;
        return this;
    }


    public OffHeapCacheBuilder<K, V> cacheKey(String cacheKey) {
        this.cacheKey = cacheKey;
        return this;
    }


    public OffHeapCacheBuilder<K, V> capacity(int capacity, CapacityUnit unit) {
        this.capacity = capacity * unit.getSize();
        return this;
    }

    public OffHeapCacheBuilder<K, V> eviction(Eviction eviction) {
        this.eviction = eviction;
        return this;
    }

    public OffHeapCacheBuilder<K, V> hashAlgorithm(HashAlgorithm hashAlgorithm) {
        this.hashAlgorithm = hashAlgorithm;
        return this;
    }

    public OffHeapCacheBuilder<K, V> debug() {
        this.log = true;
        return this;
    }

    public OffHeapCacheBuilder<K, V> enableBatch() {
        this.batchOperation = true;
        return this;
    }


    public OHCacheBuilder<K, V> getOhCacheBuilder() {
        return ohCacheBuilder;
    }

    public com.google.common.cache.Cache<K, V> getOnCache() {
        return onCache;
    }

    public OffHeapCacheBuilder<K, V> timeouts(boolean timeouts) {
        this.timeouts = timeouts;
        return this;
    }

    /**
     * 开启堆内缓存
     * @param size
     * @param duration
     * @param unit
     * @return
     */
    public OffHeapCacheBuilder<K, V> withOnCache(long size, long duration, TimeUnit unit) {
        onCache = CacheBuilder.newBuilder().recordStats().maximumSize(size)
                .expireAfterAccess(duration, unit).build();
        return this;
    }

    public Cache<K, V> build() {

        OHCache<K, V> ohCache = ohCacheBuilder
                .keySerializer(keySerializer)
                .valueSerializer(valueSerializer)
                .segmentCount(segmentCount)
                .hashTableSize(hashTableSize)
                .maxEntrySize(maxEntrySize)
                .capacity(capacity)
                .defaultTTLmillis(defaultTTLmillis)
                .timeouts(timeouts)
                .eviction(eviction)
                .hashMode(hashAlgorithm)
                .build();
        Cache<K, V> cache;
        if (!batchOperation) {
            cache = new OffHeapCache<>(ohCache, maxSize, false, log, cacheKey, onCache);
        } else {
            if (threadPool == null) {
                cache = new OffHeapCache<>(ohCache, maxSize, true, log, cacheKey, onCache);
            } else cache = new OffHeapCache<>(ohCache, maxSize, threadPool, log, cacheKey, onCache);
        }

        if (cache != null) {
            //添加到堆外监控map
            OHCStateCollector.ohcCacheMap.put(cacheKey, (OffHeapCache) cache);
            //开启OHC监控
            OHCStateCollector.startMonitor(cacheKey);
            if (onCache != null) {
                //添加到堆内监控map
                OnCacheStateCollector.onCacheMap.put(cacheKey, (OffHeapCache) cache);
                //开启堆内监控
                OnCacheStateCollector.startMonitor(cacheKey);
            }
        }

        return cache;
    }

    private static int roundUpToPowerOf2(int number, int max) {
        return number >= max
                ? max
                : (number > 1) ? Integer.highestOneBit((number - 1) << 1) : 1;
    }
}
```

<hr style=" border:solid; width:100px; height:1px;" color=#000000 size=1">


