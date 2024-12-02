---
layout: post
title: MyBatis
categories: [MyBatis源码]
description: 学习MyBatis源码
keywords: 源码, MyBatis
---

# 一、MyBatis源码之缓存模块



### 1. Cache

> **是一个容器，有点类似 HashMap ，可以往其中添加各种缓存**
>
> PerpetualCache 默认cache的实现类, 不用装饰, 其它缓存类均采用装饰者模式, 因为只是对Cache方法前后的扩展, 而不去更改Cache的方法
>
>  所在包: org.apache.ibatis.cache.Cache

```java
  void putObject(Object key, Object value);

  /**
   * 获得指定键的值
   * @param key
   *          The key
   *
   * @return The object stored in the cache.
   */
  Object getObject(Object key);

  /**
   * As of 3.3.0 this method is only called during a rollback for any previous value that was missing in the cache. This
   * lets any blocking cache to release the lock that may have previously put on the key. A blocking cache puts a lock
   * when a value is null and releases it when the value is back again. This way other threads will wait for the value
   * to be available instead of hitting the database.
   *
   * 移除指定键的值
   * @param key
   *          The key
   *
   * @return Not used
   */
  Object removeObject(Object key);

  /**
   * 清空缓存
   * Clears this cache instance.
   */
  void clear();

  /**
   * 获得容器中缓存的数量
   * Optional. This method is not called by the core.
   *
   * @return The number of elements stored in the cache (not its capacity).
   */
  int getSize();

  /**
   * 获得读取写锁, 该方法可以忽略了已经
   * Optional. As of 3.2.6 this method is no longer called by the core.
   * <p>
   * Any locking needed by the cache must be provided internally by the cache provider.
   *
   * @return A ReadWriteLock
   */
  default ReadWriteLock getReadWriteLock() {
    return null;
  }

}
```



#### 1.1 PerpetualCache

> 实现 Cache 接口，**永不过期**的 Cache 实现类，基于 HashMap 实现类, 所在包: org.apache.ibatis.cache.impl.PerpetualCache

```java
public class PerpetualCache implements Cache {

  /**
   * 标识
   */
  private final String id;
  /**
   * 缓存容器
   */
  private final Map<Object, Object> cache = new HashMap<>();

  public PerpetualCache(String id) {
    this.id = id;
  }

  @Override
  public String getId() {
    return id;
  }

  @Override
  public int getSize() {
    return cache.size();
  }

  @Override
  public void putObject(Object key, Object value) {
    cache.put(key, value);
  }

  @Override
  public Object getObject(Object key) {
    return cache.get(key);
  }

  @Override
  public Object removeObject(Object key) {
    return cache.remove(key);
  }

  @Override
  public void clear() {
    cache.clear();
  }
  // ....  省略 equals 和 hashCode 方法

}
```

#### 1.2 LoggingCache

> 实现 Cache 接口，**支持打印日志**的 Cache 实现类, 所在包: org.apache.ibatis.cache.decorators.LoggingCache

```java
public class LoggingCache implements Cache {

  /**
   * MyBatis Log 对象
   */
  private final Log log;
  /**
   * 装饰的 Cache 对象
   */
  private final Cache delegate;
  /**
   * 统计请求缓存的次数
   */
  protected int requests;
  /**
   * 统计命中缓存的次数
   */
  protected int hits;

  public LoggingCache(Cache delegate) {
    this.delegate = delegate;
    this.log = LogFactory.getLog(getId());
  }

  @Override
  public String getId() {
    return delegate.getId();
  }

  @Override
  public int getSize() {
    return delegate.getSize();
  }

  @Override
  public void putObject(Object key, Object object) {
    delegate.putObject(key, object);
  }

  @Override
  public Object getObject(Object key) {
    // 请求次数 ++
    requests++;
    // 如果命中缓存，则命中次数 ++
    final Object value = delegate.getObject(key);
    if (value != null) {
      hits++;
    }
    if (log.isDebugEnabled()) {
      log.debug("Cache Hit Ratio [" + getId() + "]: " + getHitRatio());
    }
    return value;
  }

  @Override
  public Object removeObject(Object key) {
    return delegate.removeObject(key);
  }

  @Override
  public void clear() {
    delegate.clear();
  }

  @Override
  public int hashCode() {
    return delegate.hashCode();
  }

  @Override
  public boolean equals(Object obj) {
    return delegate.equals(obj);
  }

  /**
   * 命中率
   */
  private double getHitRatio() {
    return (double) hits / (double) requests;
  }

}
```

#### 1.3 BlockingCache

> 实现 Cache 接口，**阻塞的** Cache 实现类, 当A线程未获取到缓存数据时, 其它线程会阻塞, 因为A线程未获取到数据会获取对应的值并设置对应的缓存值, 防止其它线程重复设置缓存值
>
> 所在包: org.apache.ibatis.cache.decoratorsBlockingCache

```java
public class BlockingCache implements Cache {

  /**
   * 阻塞等待超时时间
   */
  private long timeout;
  /**
   * 装饰的 Cache 对象
   */
  private final Cache delegate;
  /**
   * 缓存键与 CountDownLatch 对象的映射
   */
  private final ConcurrentHashMap<Object, CountDownLatch> locks;

  public BlockingCache(Cache delegate) {
    this.delegate = delegate;
    this.locks = new ConcurrentHashMap<>();
  }

  @Override
  public String getId() {
    return delegate.getId();
  }

  @Override
  public int getSize() {
    return delegate.getSize();
  }

  @Override
  public void putObject(Object key, Object value) {
    try {
      // 添加缓存
      delegate.putObject(key, value);
    } finally {
      // 释放锁
      releaseLock(key);
    }
  }

  @Override
  public Object getObject(Object key) {
    // 获得锁
    acquireLock(key);
    // 获得缓存值
    Object value = delegate.getObject(key);
    // 释放锁
    if (value != null) {
      releaseLock(key);
    }
    return value;
  }

  @Override
  public Object removeObject(Object key) {
    // despite its name, this method is called only to release locks
    // 释放锁
    releaseLock(key);
    return null;
  }

  @Override
  public void clear() {
    delegate.clear();
  }

  private void acquireLock(Object key) {
    // 使用当前线程尝试获取锁时创建 计数器, 初始值为1
    CountDownLatch newLatch = new CountDownLatch(1);
    while (true) {
      // 每一个key相当于1把逻辑锁
      CountDownLatch latch = locks.putIfAbsent(key, newLatch);
      if (latch == null) {
        // 成功获取锁 退出循环
        break;
      }
      try {
        if (timeout > 0) {
          // 对该key加锁, 如果设置了超时时间, 在超时时间内为获取到锁, 抛出异常
          boolean acquired = latch.await(timeout, TimeUnit.MILLISECONDS);
          if (!acquired) {
            throw new CacheException(
                "Couldn't get a lock in " + timeout + " for the key " + key + " at the cache " + delegate.getId());
          }
        } else {
          // 如果未设置超时时间, 无限等待
          latch.await();
        }
      } catch (InterruptedException e) {
        throw new CacheException("Got interrupted while trying to acquire lock for key " + key, e);
      }
    }
  }

  private void releaseLock(Object key) {
    // 根据key释放锁
    CountDownLatch latch = locks.remove(key);
    if (latch == null) {
      throw new IllegalStateException("Detected an attempt at releasing unacquired lock. This should never happen.");
    }
    latch.countDown();
  }

  public long getTimeout() {
    return timeout;
  }

  public void setTimeout(long timeout) {
    this.timeout = timeout;
  }
}
```

#### 1.4 SynchronizedCache

> 实现 Cache 接口，**同步**的 Cache 实现类, 所在包: org.apache.ibatis.cache.decorators.SynchronizedCache

```java
public class SynchronizedCache implements Cache {

  /**
   * 可重入锁
   */
  private final ReentrantLock lock = new ReentrantLock();
  /**
   * 装饰的 Cache 对象
   */
  private final Cache delegate;

  public SynchronizedCache(Cache delegate) {
    this.delegate = delegate;
  }

  @Override
  public String getId() {
    return delegate.getId();
  }

  @Override
  public int getSize() {
    // 同步
    lock.lock();
    try {
      return delegate.getSize();
    } finally {
      lock.unlock();
    }
  }

  @Override
  public void putObject(Object key, Object object) {
    // 同步
    lock.lock();
    try {
      delegate.putObject(key, object);
    } finally {
      lock.unlock();
    }
  }

  @Override
  public Object getObject(Object key) {
    // 同步
    lock.lock();
    try {
      return delegate.getObject(key);
    } finally {
      lock.unlock();
    }
  }

  @Override
  public Object removeObject(Object key) {
    // 同步
    lock.lock();
    try {
      return delegate.removeObject(key);
    } finally {
      lock.unlock();
    }
  }

  @Override
  public void clear() {
    lock.lock();
    try {
      delegate.clear();
    } finally {
      lock.unlock();
    }
  }

  // ... 省略 equals 和 hashCode 方法

}
```

#### 1.5 SerializedCache

> 实现 Cache 接口，**支持序列化值**的 Cache 实现类, 所在包: org.apache.ibatis.cache.decorators.SerializedCache

```java
public class SerializedCache implements Cache {

  /**
   * 装饰对象
   */
  private final Cache delegate;

  public SerializedCache(Cache delegate) {
    this.delegate = delegate;
  }

  @Override
  public String getId() {
    return delegate.getId();
  }

  @Override
  public int getSize() {
    return delegate.getSize();
  }

  @Override
  public void putObject(Object key, Object object) {
    if ((object != null) && !(object instanceof Serializable)) {
      throw new CacheException("SharedCache failed to make a copy of a non-serializable object: " + object);
    }
    // 序列化
    delegate.putObject(key, serialize((Serializable) object));
  }

  @Override
  public Object getObject(Object key) {
    Object object = delegate.getObject(key);
    // 反序列化
    return object == null ? null : deserialize((byte[]) object);
  }

  @Override
  public Object removeObject(Object key) {
    return delegate.removeObject(key);
  }

  @Override
  public void clear() {
    delegate.clear();
  }

  @Override
  public int hashCode() {
    return delegate.hashCode();
  }

  @Override
  public boolean equals(Object obj) {
    return delegate.equals(obj);
  }

  private byte[] serialize(Serializable value) {
    try (ByteArrayOutputStream bos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(bos)) {
      oos.writeObject(value);
      oos.flush();
      return bos.toByteArray();
    } catch (Exception e) {
      throw new CacheException("Error serializing object.  Cause: " + e, e);
    }
  }

  private Serializable deserialize(byte[] value) {
    SerialFilterChecker.check();
    Serializable result;
    try (ByteArrayInputStream bis = new ByteArrayInputStream(value);
        ObjectInputStream ois = new CustomObjectInputStream(bis)) {
      result = (Serializable) ois.readObject();
    } catch (Exception e) {
      throw new CacheException("Error deserializing object.  Cause: " + e, e);
    }
    return result;
  }

  public static class CustomObjectInputStream extends ObjectInputStream {

    public CustomObjectInputStream(InputStream in) throws IOException {
      super(in);
    }

    @Override
    protected Class<?> resolveClass(ObjectStreamClass desc) throws ClassNotFoundException {
      return Resources.classForName(desc.getName());
    }

  }

}
```

#### 1.6 ScheduledCache

> 实现 Cache 接口，**定时清空整个容器**的 Cache 实现类, 所在包: org.apache.ibatis.cache.decorators.ScheduledCache

```java
public class ScheduledCache implements Cache {

  /**
   * 被装饰的 Cache 对象
   */
  private final Cache delegate;
  /**
   * 清空间隔，单位：毫秒
   */
  protected long clearInterval;
  /**
   * 最后清空时间，单位：毫秒
   */
  protected long lastClear;

  public ScheduledCache(Cache delegate) {
    this.delegate = delegate;
    this.clearInterval = TimeUnit.HOURS.toMillis(1);
    this.lastClear = System.currentTimeMillis();
  }

  public void setClearInterval(long clearInterval) {
    this.clearInterval = clearInterval;
  }

  @Override
  public String getId() {
    return delegate.getId();
  }

  @Override
  public int getSize() {
    // 判断是否要全部清空
    clearWhenStale();
    return delegate.getSize();
  }

  @Override
  public void putObject(Object key, Object object) {
    clearWhenStale();
    delegate.putObject(key, object);
  }

  @Override
  public Object getObject(Object key) {
    return clearWhenStale() ? null : delegate.getObject(key);
  }

  @Override
  public Object removeObject(Object key) {
    clearWhenStale();
    return delegate.removeObject(key);
  }

  @Override
  public void clear() {
    // 记录清空时间
    lastClear = System.currentTimeMillis();
    // 全部清空
    delegate.clear();
  }

  @Override
  public int hashCode() {
    return delegate.hashCode();
  }

  @Override
  public boolean equals(Object obj) {
    return delegate.equals(obj);
  }

  private boolean clearWhenStale() {
    // 需要全部清空则清空
    if (System.currentTimeMillis() - lastClear > clearInterval) {
      clear();
      return true;
    }
    return false;
  }

}
```

#### 1.7 FifoCache

> 实现 Cache 接口，**基于先进先出的淘汰机制**的 Cache 实现类, 所在包: org.apache.ibatis.cache.decorators.FifoCache

```java
public class FifoCache implements Cache {

  /**
   * 装饰的 Cache 对象
   */
  private final Cache delegate;
  /**
   * 双端队列，记录缓存键的添加
   */
  private final Deque<Object> keyList;
  /**
   * 队列上限
   */
  private int size;

  public FifoCache(Cache delegate) {
    this.delegate = delegate;
    // 链表的实现方式
    this.keyList = new LinkedList<>();
    this.size = 1024;
  }

  @Override
  public String getId() {
    return delegate.getId();
  }

  @Override
  public int getSize() {
    return delegate.getSize();
  }

  public void setSize(int size) {
    this.size = size;
  }

  @Override
  public void putObject(Object key, Object value) {
    // 循环双端队列
    cycleKeyList(key);
    delegate.putObject(key, value);
  }

  @Override
  public Object getObject(Object key) {
    return delegate.getObject(key);
  }

  @Override
  public Object removeObject(Object key) {
    keyList.remove(key);
    return delegate.removeObject(key);
  }

  @Override
  public void clear() {
    delegate.clear();
    keyList.clear();
  }

  private void cycleKeyList(Object key) {
    // 添加到keyList对应位置
    keyList.addLast(key);
    // 超过上限, 将队首位移除
    if (keyList.size() > size) {
      Object oldestKey = keyList.removeFirst();
      delegate.removeObject(oldestKey);
    }
  }

}
```

#### 1.8 LruCache

> 实现 Cache 接口，**基于最少使用的淘汰机制**的 Cache 实现类, 所在包: org.apache.ibatis.cache.decorators.LruCache

```java
public class LruCache implements Cache {
  /**
   * 装饰的 Cache 对象
   */
  private final Cache delegate;
  /**
   * 基于 LinkedHashMap 实现淘汰机制
   */
  private Map<Object, Object> keyMap;
  /**
   * 最老的键，即要被淘汰的
   */
  private Object eldestKey;

  public LruCache(Cache delegate) {
    this.delegate = delegate;
    // 初始化keyMap对象
    setSize(1024);
  }

  @Override
  public String getId() {
    return delegate.getId();
  }

  @Override
  public int getSize() {
    return delegate.getSize();
  }

  public void setSize(final int size) {
    // LinkedHashMap的一个构造函数，当参数accessOrder为true时，即会按照访问顺序排序，最近访问的放在最前，最早访问的放在后面
    keyMap = new LinkedHashMap<Object, Object>(size, .75F, true) {
      private static final long serialVersionUID = 4267176411845948333L;

      // LinkedHashMap自带的判断是否删除最老的元素方法，默认返回false，即不删除老数据
      // 我们要做的就是重写这个方法，当满足一定条件时删除老数据
      @Override
      protected boolean removeEldestEntry(Map.Entry<Object, Object> eldest) {
        boolean tooBig = size() > size;
        if (tooBig) {
          eldestKey = eldest.getKey();
        }
        return tooBig;
      }
    };
  }

  @Override
  public void putObject(Object key, Object value) {
    // 添加到缓存
    delegate.putObject(key, value);
    // 循环keyMao
    cycleKeyList(key);
  }

  @Override
  public Object getObject(Object key) {
    // 刷新keyMap的访问顺序
    keyMap.get(key); // touch
    // 获得缓存值
    return delegate.getObject(key);
  }

  @Override
  public Object removeObject(Object key) {
    keyMap.remove(key);
    return delegate.removeObject(key);
  }

  @Override
  public void clear() {
    delegate.clear();
    keyMap.clear();
  }

  private void cycleKeyList(Object key) {
    // 添加到keyMap中
    keyMap.put(key, key);
    // 如果超出上限, 则移除
    if (eldestKey != null) {
      delegate.removeObject(eldestKey);
      eldestKey = null;
    }
  }

}
```

#### 1.9 WeakCache

> 实现 Cache 接口，基于 `java.lang.ref.WeakReference` 的 Cache 实现类, 所在包: org.apache.ibatis.cache.decorators.WeakCache

```java
public class WeakCache implements Cache {
  /**
   * 强引用的键的队列
   */
  private final Deque<Object> hardLinksToAvoidGarbageCollection;
  /**
   * 被 GC 回收的 WeakEntry 集合，避免被 GC。
   */
  private final ReferenceQueue<Object> queueOfGarbageCollectedEntries;
  /**
   * 装饰的cache对象
   */
  private final Cache delegate;
  /**
   * {@link #hardLinksToAvoidGarbageCollection} 的大小
   */
  private int numberOfHardLinks;
  private final ReentrantLock lock = new ReentrantLock();

  public WeakCache(Cache delegate) {
    this.delegate = delegate;
    this.numberOfHardLinks = 256;
    this.hardLinksToAvoidGarbageCollection = new LinkedList<>();
    this.queueOfGarbageCollectedEntries = new ReferenceQueue<>();
  }

  @Override
  public String getId() {
    return delegate.getId();
  }

  @Override
  public int getSize() {
    // 移除已被GC回收的 WeakEntry
    removeGarbageCollectedItems();
    return delegate.getSize();
  }

  public void setSize(int size) {
    this.numberOfHardLinks = size;
  }

  @Override
  public void putObject(Object key, Object value) {
    // 移除已被GC回收的 WeakEntry
    removeGarbageCollectedItems();
    // 添加到delegate中
    delegate.putObject(key, new WeakEntry(key, value, queueOfGarbageCollectedEntries));
  }

  @Override
  public Object getObject(Object key) {
    Object result = null;
    @SuppressWarnings("unchecked") // assumed delegate cache is totally managed by this cache
    // 获得值的 WeakReference 对象
    WeakReference<Object> weakReference = (WeakReference<Object>) delegate.getObject(key);
    if (weakReference != null) {
      // 获得值
      result = weakReference.get();
      // 为空，从 delegate 中移除 。为空的原因是，意味着已经被 GC 回收
      if (result == null) {
        delegate.removeObject(key);
        // 非空，添加到 hardLinksToAvoidGarbageCollection 中，避免被 GC
      } else {
        lock.lock();
        try {
          // 添加到 hardLinksToAvoidGarbageCollection 的队头
          hardLinksToAvoidGarbageCollection.addFirst(result);
          // 超过上限，移除 hardLinksToAvoidGarbageCollection 的队尾
          if (hardLinksToAvoidGarbageCollection.size() > numberOfHardLinks) {
            hardLinksToAvoidGarbageCollection.removeLast();
          }
        } finally {
          lock.unlock();
        }
      }
    }
    return result;
  }

  @Override
  public Object removeObject(Object key) {
    // 移除已被GC回收的WeakEntry
    removeGarbageCollectedItems();
    @SuppressWarnings("unchecked")
    // 移除 delegate
    WeakReference<Object> weakReference = (WeakReference<Object>) delegate.removeObject(key);
    return weakReference == null ? null : weakReference.get();
  }

  @Override
  public void clear() {
    lock.lock();
    try {
      // 清空 hardLinksToAvoidGarbageCollection
      hardLinksToAvoidGarbageCollection.clear();
    } finally {
      lock.unlock();
    }
    // 移除已经被 GC 回收的 WeakEntry
    removeGarbageCollectedItems();
    // 清空delegate
    delegate.clear();
  }

  /**
   * 移除已被GC回收的键
   */
  private void removeGarbageCollectedItems() {
    WeakEntry sv;
    while ((sv = (WeakEntry) queueOfGarbageCollectedEntries.poll()) != null) {
      delegate.removeObject(sv.key);
    }
  }

  private static class WeakEntry extends WeakReference<Object> {
    private final Object key;

    private WeakEntry(Object key, Object value, ReferenceQueue<Object> garbageCollectionQueue) {
      super(value, garbageCollectionQueue);
      this.key = key;
    }
  }

}
```

#### 1.10 SoftCache

> 实现 Cache 接口，基于 `java.lang.ref.SoftReference` 的 Cache 实现类, 所在包: org.apache.ibatis.cache.decorators.SoftCache

```java
public class SoftCache implements Cache {
  /**
   * 强引用的键的队列
   */
  private final Deque<Object> hardLinksToAvoidGarbageCollection;
  /**
   * 被 GC 回收的 WeakEntry 集合, 当软引用被回收时, 会将回收对象加入队列, 方便判断哪些对象被回收或者维护缓存的一致性
   */
  private final ReferenceQueue<Object> queueOfGarbageCollectedEntries;
  /**
   * 装饰的cache
   */
  private final Cache delegate;
  /**
   * {@link #hardLinksToAvoidGarbageCollection} 的大小
   */
  private int numberOfHardLinks;
  private final ReentrantLock lock = new ReentrantLock();

  public SoftCache(Cache delegate) {
    this.delegate = delegate;
    this.numberOfHardLinks = 256;
    this.hardLinksToAvoidGarbageCollection = new LinkedList<>();
    this.queueOfGarbageCollectedEntries = new ReferenceQueue<>();
  }

  @Override
  public String getId() {
    return delegate.getId();
  }

  @Override
  public int getSize() {
    removeGarbageCollectedItems();
    return delegate.getSize();
  }

  public void setSize(int size) {
    this.numberOfHardLinks = size;
  }

  @Override
  public void putObject(Object key, Object value) {
    // 移除已经被 GC 回收的 SoftEntry
    removeGarbageCollectedItems();
    delegate.putObject(key, new SoftEntry(key, value, queueOfGarbageCollectedEntries));
  }

  @Override
  public Object getObject(Object key) {
    Object result = null;
    @SuppressWarnings("unchecked") // assumed delegate cache is totally managed by this cache
    // 获得值的 WeakReference 对象
    SoftReference<Object> softReference = (SoftReference<Object>) delegate.getObject(key);
    if (softReference != null) {
      // 获得值
      result = softReference.get();
      // 为 null, 意味着已经被GC清除, 从delegate中删除
      if (result == null) {
        delegate.removeObject(key);
      } else {
        // See #586 (and #335) modifications need more than a read lock
        lock.lock();
        try {
          hardLinksToAvoidGarbageCollection.addFirst(result);
          if (hardLinksToAvoidGarbageCollection.size() > numberOfHardLinks) {
            hardLinksToAvoidGarbageCollection.removeLast();
          }
        } finally {
          lock.unlock();
        }
      }
    }
    return result;
  }

  @Override
  public Object removeObject(Object key) {
    removeGarbageCollectedItems();
    @SuppressWarnings("unchecked")
    SoftReference<Object> softReference = (SoftReference<Object>) delegate.removeObject(key);
    return softReference == null ? null : softReference.get();
  }

  @Override
  public void clear() {
    lock.lock();
    try {
      hardLinksToAvoidGarbageCollection.clear();
    } finally {
      lock.unlock();
    }
    // 移除已经被gc回收的对象
    removeGarbageCollectedItems();
    delegate.clear();
  }

  private void removeGarbageCollectedItems() {
    // 通过垃圾回收队列 保证缓存和回收对象的一致性
    SoftEntry sv;
    while ((sv = (SoftEntry) queueOfGarbageCollectedEntries.poll()) != null) {
      delegate.removeObject(sv.key);
    }
  }

  private static class SoftEntry extends SoftReference<Object> {
    private final Object key;

    SoftEntry(Object key, Object value, ReferenceQueue<Object> garbageCollectionQueue) {
      super(value, garbageCollectionQueue);
      this.key = key;
    }
  }

}
```

### 2. CacheKey

> 因为 MyBatis 中的缓存键不是一个简单的 String ，**而是通过多个对象组成**, 所以 CacheKey 可以理解成将多个对象放在一起，计算其缓存键

- 构造方法

  ```java
    private static final long serialVersionUID = 1146682552656046210L;
  
    /**
     * 单例 - 空缓存键
     */
    public static final CacheKey NULL_CACHE_KEY = new CacheKey() {
  
      private static final long serialVersionUID = 1L;
  
      @Override
      public void update(Object object) {
        throw new CacheException("Not allowed to update a null cache key instance.");
      }
  
      @Override
      public void updateAll(Object[] objects) {
        throw new CacheException("Not allowed to update a null cache key instance.");
      }
    };
    /**
     * 默认 {@link #multiplier} 的值
     */
    private static final int DEFAULT_MULTIPLIER = 37;
    /**
     * 默认 {@link #hashcode} 的值
     */
    private static final int DEFAULT_HASHCODE = 17;
    /**
     * hashcode 求值的系数
     */
    private final int multiplier;
    /**
     * 缓存键的 hashcode
     */
    private int hashcode;
    /**
     * 校验和
     */
    private long checksum;
    /**
     * {@link #update(Object)} 的数量
     */
    private int count;
    // 8/21/2017 - Sonarlint flags this as needing to be marked transient. While true if content is not serializable, this
    // is not always true and thus should not be marked transient.
    /**
     * 计算 {@link #hashcode} 的对象的集合
     */
    private List<Object> updateList;
  
    public CacheKey() {
      this.hashcode = DEFAULT_HASHCODE;
      this.multiplier = DEFAULT_MULTIPLIER;
      this.count = 0;
      this.updateList = new ArrayList<>();
    }
  
    public CacheKey(Object[] objects) {
      this();
      updateAll(objects);
    }
  ```

- updateAll

  ```java
    public void updateAll(Object[] objects) {
      for (Object o : objects) {
        update(o);
      }
    }
  
    /**
     * 组合的缓存key是通过组合key的hashcode替代, 所以更新缓存需要更新对应组合key的hashcode
     * @param object
     */
    public void update(Object object) {
      int baseHashCode = object == null ? 1 : ArrayUtil.hashCode(object);
  
      count++;
      checksum += baseHashCode;
      baseHashCode *= count;
  
      hashcode = multiplier * hashcode + baseHashCode;
  
      updateList.add(object);
    }
  ```

- hashcode & equals

  ```java
    @Override
    public int hashCode() {
      return hashcode;
    }
  
    @Override
    public boolean equals(Object object) {
      if (this == object) {
        return true;
      }
      if (!(object instanceof CacheKey)) {
        return false;
      }
  
      final CacheKey cacheKey = (CacheKey) object;
  
      if ((hashcode != cacheKey.hashcode) || (checksum != cacheKey.checksum) || (count != cacheKey.count)) {
        return false;
      }
  
      for (int i = 0; i < updateList.size(); i++) {
        Object thisObject = updateList.get(i);
        Object thatObject = cacheKey.updateList.get(i);
        if (!ArrayUtil.equals(thisObject, thatObject)) {
          return false;
        }
      }
      return true;
    }
  ```

- clone

  ```java
    @Override
    public CacheKey clone() throws CloneNotSupportedException {
      // 克隆cacheKey对象
      CacheKey clonedCacheKey = (CacheKey) super.clone();
      // 创建 updateList 数组, 避免原数组修改
      clonedCacheKey.updateList = new ArrayList<>(updateList);
      return clonedCacheKey;
    }
  ```

- NullCacheKey

  ```java
    public static final CacheKey NULL_CACHE_KEY = new CacheKey() {
  
      private static final long serialVersionUID = 1L;
  
      @Override
      public void update(Object object) {
        throw new CacheException("Not allowed to update a null cache key instance.");
      }
  
      @Override
      public void updateAll(Object[] objects) {
        throw new CacheException("Not allowed to update a null cache key instance.");
      }
    };
  ```

  

