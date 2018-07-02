---
layout:     post
title:      Java集合类学习之WeakHashMap
date:       2017-11-25
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Java基础 
    - JDK源码阅读
---

>本篇随笔主要记录了我阅读WeakHashMap(JDK1.8)源码期间的对于WeakHashMap的一些实现上的个人理解，用于个人备忘，有不对的地方，请指出。  
  
_ _ _
### **前言**
顾名思义，WeakHashMap与HashMap的不同之处在于其key值是被弱引用包裹的，可以在一些情况下用于防止内存泄漏。关于WeakHashMap的应用场景，可以参考：[用弱引用堵住内存泄漏](https://www.ibm.com/developerworks/cn/java/j-jtp11225/index.html)。本文主要介绍WeakHashMap的内部实现机制。  

不同于HashMap的数组+链表+红黑树的数据结构，WeakHashMap对于HashMap的数据结构进行了简化，使用的是数组+链表，并没有使用到红黑树，其基本节点实现如下：  
```java

Entry<K,V>[] table;

/**
 * The entries in this hash table extend WeakReference, using its main ref
 * field as the key.
 */
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
    V value;
    final int hash;
    Entry<K,V> next;

    /**
     * Creates new entry.
     * 每创建一个新的Entry，就相当于创建了一个WeakReference，并将其关联到一个引用队列中
     */
    Entry(Object key, V value,
          ReferenceQueue<Object> queue,
          int hash, Entry<K,V> next) {
        super(key, queue);
        this.value = value;
        this.hash  = hash;
        this.next  = next;
    }

    @SuppressWarnings("unchecked")
    public K getKey() {
        return (K) WeakHashMap.unmaskNull(get());
    }

    public V getValue() {
        return value;
    }

    public V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public boolean equals(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry<?,?>)o;
        K k1 = getKey();
        Object k2 = e.getKey();
        if (k1 == k2 || (k1 != null && k1.equals(k2))) {
            V v1 = getValue();
            Object v2 = e.getValue();
            if (v1 == v2 || (v1 != null && v1.equals(v2)))
                return true;
        }
        return false;
    }

    public int hashCode() {
        K k = getKey();
        V v = getValue();
        return Objects.hashCode(k) ^ Objects.hashCode(v);
    }

    public String toString() {
        return getKey() + "=" + getValue();
    }
}
```

#### **Put操作实现**
```java
private final ReferenceQueue<Object> queue = new ReferenceQueue<>();

public V put(K key, V value) {
    Object k = maskNull(key);
    int h = hash(k); // 计算其hash值
    Entry<K,V>[] tab = getTable();
    int i = indexFor(h, tab.length); // 找到在表中要插入的位置

    // 待插入的值如果之前存在则替换
    for (Entry<K,V> e = tab[i]; e != null; e = e.next) {
        if (h == e.hash && eq(k, e.get())) {
            V oldValue = e.value;
            if (value != oldValue)
                e.value = value;
            return oldValue;
        }
    }

    modCount++;
    // 不存在则添加，注意新添加的所有节点都关联到了同一个Reference Queue中
    Entry<K,V> e = tab[i];
    tab[i] = new Entry<>(k, value, queue, h, e);
    if (++size >= threshold)
        resize(tab.length * 2);
    return null;
}
```
至于WeakHashMap的get操作，remove操作等都是基于数组+链表进行的操作，逻辑都比较简单，这里就不赘述了。  

#### **弱引用的删除**
在上面put方法中可以看到，WeakHashMap中的每个key都会关联到一个相同的ReferenceQueue中，当key被检测到是弱引用可达时，就会由Reference类中开启的后台线程将此key加入到此ReferenceQueue中，那么这个key的弱引用及其对应的value是如何从WeakHashMap中删除的呢？   

原来WeakHashMap 有一个名为 expungeStaleEntries() 的私有方法，大多数 WeakHashMap 操作中会调用它(比如size()方法)，并没有采用后台线程轮询此ReferenceQueue检测是否为空来实现。expungeStaleEntries()方法中去掉引用队列中所有失效的引用，并删除关联的映射。下面代码展示了 expungeStaleEntries() 的实现：    
```java
/**
 * Expunges stale entries from the table.
 */
private void expungeStaleEntries() {
    for (Object x; (x = queue.poll()) != null; ) {
        synchronized (queue) {
            @SuppressWarnings("unchecked")
                Entry<K,V> e = (Entry<K,V>) x;
            int i = indexFor(e.hash, table.length); // 找到要删除的引用在map中的位置

            Entry<K,V> prev = table[i];
            Entry<K,V> p = prev;
            while (p != null) {
                Entry<K,V> next = p.next;
                if (p == e) {
                    if (prev == e) // 如果table[i]就是要删除的引用节点
                        table[i] = next;
                    else
                        prev.next = next; // 删除此引用
                    // Must not null out e.next;
                    // stale entries may be in use by a HashIterator
                    e.value = null; // Help GC // 删除此引用对应的值
                    size--;
                    break;
                }
                prev = p;
                p = next;
            }
        }
    }
}
```

关于WeakHashMap的源码就分析到这里。  

(完)  
