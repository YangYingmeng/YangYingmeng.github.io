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
