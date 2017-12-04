---
title: ConcurrentMap源码笔记
date: 2017-12-02 15:56:32
summary: 本篇对ConcurrentMap接口源码做了简单分析及整理
tags: Java Collections Framework
---

# ConcurrentMap源码笔记
## 主要特性
> * 1.提供线程安全及原子性保证
> * 2.线程将某个对象作为键/值放入ConcurrentMap的动作happens-before随后由其它线程执行的访问/删除该对象的动作。即保证了内存一致性效果。
> * 关于happens-before特性可以参考官方文档[happens-before](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/package-summary.html#MemoryVisibility)

## 主要方法
ConcurrentMap继承自Map接口，重写了Map接口中的部分default方法
```java
public interface ConcurrentMap<K, V> extends Map<K, V> 
```

#### getOrDefault
getOrDefault方法假定ConcurrentMap不能以null作为value，即get方法返回null就表示键不存在。
⚠️ 实现ConcurrentMap接口时如果需要支持null作为value，就必须重写本方法。
```java
@Override
default V getOrDefault(Object key, V defaultValue) {
    V v;
    return ((v = get(key)) != null) ? v : defaultValue;
}
```

#### forEach
ConcurrentMap接口重写的forEach方法在遍历entrySet过程中遇到mapping关系已经被移除的情况时不会向上抛出异常而是继续执行之后的遍历操作。
```java
@Override
default void forEach(BiConsumer<? super K, ? super V> action) {
    Objects.requireNonNull(action);
    for (Map.Entry<K, V> entry : entrySet()) {
        K k;
        V v;
        try {
            k = entry.getKey();
            v = entry.getValue();
        } catch(IllegalStateException ise) {
            // this usually means the entry is no longer in the map.
            continue;
        }
        action.accept(k, v);
    }
}
```

#### putIfAbsent/remove/replace
因为Map接口提供的默认实现并不具备同步特性，ConcurrentMap接口将这些方法重写为抽象方法，需要子类予以实现。
```java
// 不存在指定key时执行put操作
V putIfAbsent(K key, V value);

// 存在指定key/value对时执行remove操作
boolean remove(Object key, Object value);

// 存在指定key/oldValue对时执行replace操作
boolean replace(K key, V oldValue, V newValue);

// 存在指定key时执行replace操作并返回旧的value值
V replace(K key, V value);
```

#### replaceAll
当有多个线程使用同一个key调用function时，此默认实现可能执行多次针对该key的replace操作。
⚠️ 本方法假定不支持null作为value，实现ConcurrentMap接口时如果需要支持null作为value，就必须重写本方法。
```java
@Override
default void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) {
    Objects.requireNonNull(function);
    forEach((k,v) -> {
		// 重试replace操作直到操作成功或者指定key被其它线程移除
        while(!replace(k, v, function.apply(k, v))) {
            // v changed or k is gone
            if ( (v = get(k)) == null) {
                // k is no longer in the map.
                break;
            }
        }
    });
}
```

#### computeIfAbsent
当遇到多线程试图执行更新操作时，此默认实现可能多次指定针对指定key的compute操作。
📎此默认实现支持多线程的关键在于调用了putIfAbsent方法。
⚠️ 本方法假定不支持null作为value，实现ConcurrentMap接口时如果需要支持null作为value，就必须重写本方法。
```java
@Override
default V computeIfAbsent(K key,
        Function<? super K, ? extends V> mappingFunction) {
    Objects.requireNonNull(mappingFunction);
    V v, newValue;
    return ((v = get(key)) == null &&
            (newValue = mappingFunction.apply(key)) != null &&
            (v = putIfAbsent(key, newValue)) == null) ? newValue : v;
}

```

#### computeIfPresent
当遇到多线程试图执行更新操作时，本默认实现可能多次执行get->apply操作。
📎此默认实现支持多线程的关键在于当检测到由get操作获取到的value值已经由其它线程更新为新值时重复get->apply操作。
⚠️ 本方法假定不支持null作为value，实现ConcurrentMap接口时如果需要支持null作为value，就必须重写本方法。
```java
@Override
default V computeIfPresent(K key,
        BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
    Objects.requireNonNull(remappingFunction);
    V oldValue;
    while((oldValue = get(key)) != null) {
        V newValue = remappingFunction.apply(key, oldValue);
        if (newValue != null) {
        	// replace／remove方法返回false表示指定key对应的value已经发生变化
            if (replace(key, oldValue, newValue))
                return newValue;
        } else if (remove(key, oldValue))
           return null;
    }
    return oldValue;
}
```

#### compute
本方法特性和computeIfPresent类似，都会在检测到get获取的oldValue发生变化时重复get->apply操作来保证数据一致性，这里不再赘述。
```java
@Override
default V compute(K key,
        BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
    Objects.requireNonNull(remappingFunction);
    V oldValue = get(key);
    for(;;) {
        V newValue = remappingFunction.apply(key, oldValue);
        if (newValue == null) {
            // delete mapping
            if (oldValue != null || containsKey(key)) {
                // something to remove
                if (remove(key, oldValue)) {
                    // removed the old value as expected
                    return null;
                }

                // some other value replaced old value. try again.
                oldValue = get(key);
            } else {
                // nothing to do. Leave things as they were.
                return null;
            }
        } else {
            // add or replace old mapping
            if (oldValue != null) {
                // replace
                if (replace(key, oldValue, newValue)) {
                    // replaced as expected.
                    return newValue;
                }

                // some other value replaced old value. try again.
                oldValue = get(key);
            } else {
                // add (replace if oldValue was null)
                if ((oldValue = putIfAbsent(key, newValue)) == null) {
                    // replaced
                    return newValue;
                }

                // some other value replaced old value. try again.
            }
        }
    }
}
```

### merge
本方法特性和computeIfPresent/compute类似，都会在检测到get获取的oldValue发生变化时重复get->apply操作来保证数据一致性，这里不再赘述。
```java
@Override
default V merge(K key, V value,
        BiFunction<? super V, ? super V, ? extends V> remappingFunction) {
    Objects.requireNonNull(remappingFunction);
    Objects.requireNonNull(value);
    V oldValue = get(key);
    for (;;) {
        if (oldValue != null) {
            V newValue = remappingFunction.apply(oldValue, value);
            if (newValue != null) {
                if (replace(key, oldValue, newValue))
                    return newValue;
            } else if (remove(key, oldValue)) {
                return null;
            }
            oldValue = get(key);
        } else {
            if ((oldValue = putIfAbsent(key, value)) == null) {
                return value;
            }
        }
    }
}
```


