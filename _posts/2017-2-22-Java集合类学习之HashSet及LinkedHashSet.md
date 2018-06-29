---
layout:     post
title:      Java集合类学习之HashSet及LinkedHashSet
date:       2017-2-22
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Java基础 
    - JDK源码阅读
---

>本篇随笔主要记录了我阅读HashSet及LinkedHashSet(JDK1.8)源码期间的个人理解，用于个人备忘，有不对的地方，请指出。  
  
_ _ _
### **HashSet实现**
HashSet底层是依赖于HashMap来实现的，在之前的文章中我分析过了HashMap的源码，这里理解HashSet底层实现就非常容易了。  

其内部成员变量和构造方法如下所述：  
```java
private transient HashMap<E,Object> map;

// 底层map中key值各不相同，value对应的都是同一个对象
private static final Object PRESENT = new Object();

public HashSet() {
    map = new HashMap<>();
}

public HashSet(Collection<? extends E> c) {
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}

public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}

public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}

// 这个构造方法是给LinkedHashSet类来使用的
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```

其对外暴露的增删查方法如下,都非常简单：  
```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}
public void clear() {
    map.clear();
}
```

HashSet遍历不保证元素顺序，调用的是HashMap的遍历方法：  
```java
public Iterator<E> iterator() {
    return map.keySet().iterator();
}
abstract class HashIterator {
    final Node<K,V> nextNode() {
        Node<K,V>[] t;
        Node<K,V> e = next;
        // fail-fast机制
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        // 按照数组加链表方式进行查找
        if ((next = (current = e).next) == null && (t = table) != null) {          
            do {} while (index < t.length && (next = t[index++]) == null);
        }
        return e;
    }
}
```

_ _ _
### **LinkedHashSet实现**
LinkedHashSet底层是依赖于LinkedHashMap来实现的，与HashSet相比，LinkedHashSet的唯一不同之处在于对LinkedHashSet进行遍历时是保证元素顺序的。  

下面是LinkedHashSet实现的构造函数，其余函数均继承于HashSet：  
```java
public LinkedHashSet(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor, true);
}
public LinkedHashSet(int initialCapacity) {
    super(initialCapacity, .75f, true);
}
public LinkedHashSet() {
    super(16, .75f, true);
}
public LinkedHashSet(Collection<? extends E> c) {
    super(Math.max(2*c.size(), 11), .75f, true);
    addAll(c);
}

// HashSet中给LinkedHashSet类来使用的构造方法
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```
LinkedHashSet遍历实现：  
```java
final class LinkedKeySet extends AbstractSet<K> {  
    public final Iterator<K> iterator() {
        return new LinkedKeyIterator();
    }
}
final class LinkedKeyIterator extends LinkedHashIterator
    implements Iterator<K> {
    public final K next() { return nextNode().getKey(); }
}
abstract class LinkedHashIterator {
    LinkedHashIterator() {
        // 从头结点开始向后遍历
        next = head;
        expectedModCount = modCount;
        current = null;
    }
    final LinkedHashMap.Entry<K,V> nextNode() {
        LinkedHashMap.Entry<K,V> e = next;
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        current = e;
        next = e.after;
        return e;
    }
}
```
可以看到其遍历是从头结点开始依次向后遍历，故此可以保证按照元素插入顺序实现。   

(完)  