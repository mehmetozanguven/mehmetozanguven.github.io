---
layout: post
title: "Create simple hash map with expiration(time-to-live) feature"
date: 2022-02-16 +0530
categories: "java"
author: "mehmetozanguven"
---

You want to add & use simple cache in your application via hash map, but also you want your cache to remove automatically expired values(expiration timeout determined by you when you created the hashmap).

For me, I also needed simple cache with like yours requirements.

Then I just wrote java class(es) to create a new simple cache(using hashmap) with expiration(time-to-live) feature.

Key point is that wrap your cache items with another object called `CacheValue` includes `LocalDateTime`field

Let's start with the interface. Our simple cache must implement the following methods:

```java
public interface SimpleCache<K, V> {
    void clearAll();

    boolean hasKey(K key);

    Optional<V> get(K Key);

    void put(K key, V value);

    void remove(K key);

    long getMaxSize();

    String getCacheName();
}
```

Here is the practical implementation:

```java
public class SimpleExpiredCacheImpl<K, V> implements SimpleCache<K, V>
{
    private static final Logger logger = LoggerFactory.getLogger(SimpleExpiredCacheImpl.class);
    private static final String DEFAULT_CACHE_NAME = SimpleExpiredCacheImpl.class.getSimpleName();
    private static final long DEFAULT_CACHE_TIMEOUT = 60000L; // one min in mis
    private static final long DEFAULT_MAX_SIZE = 100000L;

    private Map<K, CacheValue<V>> simpleCache;
    private final String cacheName;
    private final long cacheTimeout;
    private final long maxSize;

    public SimpleExpiredCacheImpl()
    {
        this(DEFAULT_CACHE_NAME, DEFAULT_CACHE_TIMEOUT, DEFAULT_MAX_SIZE);
    }

    public SimpleExpiredCacheImpl(String cacheName, long cacheTimeout, long maxSize)
    {
        this.cacheName = cacheName;
        this.cacheTimeout = cacheTimeout;
        this.maxSize = maxSize;
        this.simpleCache = new HashMap<>();
        logger.info("Cache: '{}' with maxSize: {} and timeout: {} was created.", cacheName, maxSize, cacheTimeout);
    }

    @Override
    public void clearAll()
    {
        this.simpleCache.clear();
    }

    @Override
    public boolean hasKey(K key)
    {
        return this.simpleCache.containsKey(key);
    }

    @Override
    public Optional<V> get(K key)
    {
        CacheValue<V> cachedItem = this.simpleCache.get(key);
        if (cachedItem == null)
        {
            return Optional.empty();
        }
        else if (isCacheValueExpired(cachedItem))
        {
            remove(key);
            return Optional.empty();
        }
        else
        {
            return Optional.of(cachedItem.value);
        }
    }

    @Override
    public void put(K key, V value)
    {
        if (this.simpleCache.size() > getMaxSize())
        {
            logger.warn("Cache: '{}' size reached to its max size: {}. Clearing the cache", getCacheName(), getMaxSize());
            this.clearAll();
        }

        CacheValue<V> cacheValue = new CacheValue<>(value, cacheTimeout);
        this.simpleCache.put(key, cacheValue);
    }

    @Override
    public void remove(K key)
    {
        this.simpleCache.remove(key);
    }

    @Override
    public long getMaxSize()
    {
        return this.maxSize;
    }

    @Override
    public String getCacheName()
    {
        return cacheName;
    }

    private boolean isCacheValueExpired(CacheValue<V> value)
    {
        LocalDateTime expiryDate = value.expiryDate;
        return LocalDateTime.now().isAfter(expiryDate);
    }

    // Wrapper for cache items
    private static class CacheValue<V>
    {
        public final V value;
        public final LocalDateTime expiryDate;

        public CacheValue(V value, long cacheTimeoutInMs)
        {
            this.value = value;
            this.expiryDate = LocalDateTime.now().plus(cacheTimeoutInMs, ChronoUnit.MILLIS);
        }
    }
}
```

Before putting the item(s) to the cache, we are wrapping with the new object called `CacheValue` which includes expiration date = now + expirationInMs.

And also while getting element from the cache:

- If it is not found, then we are just returning Optional.empty();

- If it is found, then we are just checking item in the cache expired or not. If it is expired, we are deleting that record and returning Optional.empty, otherwise we are just returning the item

> To check, whether item in the cache is expired or not, we are just running one line of code:
>
> ```java
> LocalDateTime.now().isAfter(expiryDate);
> ```

You may also remove all the expired keys while you are putting new item in the cache. However iterating over all hashmap may lead to the performance issue. That's why I decided to check expiration date in the get method.

At the end, you can use this simple java classes to create simple cache in your application.
