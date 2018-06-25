---
layout:     post
title:      Java集合类学习之LinkedHashMap
date:       2017-2-20
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Java基础 
    - JDK源码阅读
---

>本篇随笔主要记录了我阅读LinkedHashMap(JDK1.8)源码期间的对于LinkedHashMap的一些实现上的个人理解，用于个人备忘，有不对的地方，请指出。  
  
_ _ _
### **前言**
LinkedHashMap相比于HashMap，主要就是增加了控制节点访问顺序的特性，我们可以指定按照插入顺序遍历LinkedHashMap，也可以指定按照LRU方式遍历LinkedHashMap。使用LinkedHashMap实现的LRU cache代码如下：  
```java
import java.util.*;  

//扩展一下LinkedHashMap这个类，让他实现LRU算法  
class LRULinkedHashMap<K,V> extends LinkedHashMap<K,V> {  
    //定义缓存的容量  
    private int capacity;  
    private static final long serialVersionUID = 1L;  
    //带参数的构造器     
    LRULinkedHashMap(int capacity) {  
        //调用LinkedHashMap的构造器，传入以下参数  
        super(8,0.75f,true);  
        //传入指定的缓存最大容量  
        this.capacity = capacity;  
    }  
    //实现LRU的关键方法，如果map里面的元素个数大于了缓存最大容量，则删除链表的顶端元素  
    @Override  
    public boolean removeEldestEntry(Map.Entry<K, V> eldest) {   
        System.out.println(eldest.getKey() + "=" + eldest.getValue());    
        return size() > capacity;  
    }    
}  
//测试类  
class Test{  

    public static void main(String[] args) throws Exception {  
	    //指定缓存最大容量为4  
	    Map<Integer,Integer> map = new LRULinkedHashMap<>(4);  
	    map.put(9,3);  
	    map.put(7,4);  
	    map.put(5,9);  
	    map.put(3,4);  
	    map.put(6,6);  
	    //总共put了5个元素，超过了指定的缓存最大容量  
	    //遍历结果  
        for(Iterator<Map.Entry<Integer,Integer>> it = map.entrySet().iterator(); it.hasNext(); ){  
            System.out.println(it.next().getKey());  
        }  
    }  
}  

输出结果：  
9=3
9=3
9=3
9=3
9=3
7
5
3
6

``` 
可见在超过指定的最大缓存容量的时候，果然按照LRU原则删除了元素。那么LinkedHashMap是如何按照LRU原则来保证元素访问顺序的呢？接下来的源码分析给出答案。  

_ _ _
### **数据结构及构造方法**  
LinkedHashMap中的节点类是Entry，继承自HashMap中的Node节点，在Node节点的基础上增加了before和after两个节点，定义如下：  
```java
static class Entry<K,V> extends HashMap.Node<K,V> {
    // 通过before及after为每个节点保存指向其前驱和后继的指针   
    Entry<K,V> before, after;

    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```
LinkedHashMap的底层数据结构与HashMap类似，只不过在HashMap的基础上使用一个双端链表维持插入节点的顺序或者访问节点的顺序。每个节点的结构如下图：   
<img src="/img/2017-2-20/pointStructure.png" width="300" height="300" alt="LinkedHashMap节点结构图" />
<center>图1：LinkedHashMap节点结构图</center>    

**这里尤其需要注意的是，在接下来介绍的LinkedHashMap的put、get、remove方法中涉及到的双端链表的操作，由于都是引用的更改，所以并没有影响到HashMap的底层数据结构：数组+链表+红黑树。LinkedHashMap只不过是将数组、链表、红黑树中的每个节点使用before和after指针串联了起来而已。**   

LinkedHashMap的重要字段有如下几个：  
```java
// 双向链表的首节点
transient LinkedHashMap.Entry<K,V> head;

// 双向链表的尾节点
transient LinkedHashMap.Entry<K,V> tail;

/**
 * The iteration ordering method for this linked hash map: <tt>true</tt>
 * for access-order, <tt>false</tt> for insertion-order.
 * false代表按插入顺序维持节点的顺序，true代表按LRU方式维持节点顺序，默认为false
 * @serial
 */
final boolean accessOrder;
```
LinkedHashMap的构造方法主要也是使用父类的构造方法并将accessOrder赋值，默认为false。accessOrder为final字段，值只能在构造方法中传入。构造方法如下：
```java
public LinkedHashMap(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor);
    accessOrder = false;
}

public LinkedHashMap(int initialCapacity) {
    super(initialCapacity);
    accessOrder = false;
}

public LinkedHashMap() {
    super();
    accessOrder = false; // 默认按照插入顺序维持节点顺序
}


public LinkedHashMap(Map<? extends K, ? extends V> m) {
    super();
    accessOrder = false;
    putMapEntries(m, false);
}

public LinkedHashMap(int initialCapacity,
                     float loadFactor,
                     boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}
```

_ _ _
### **put()操作**  
LinkedHashMap的put方法继承自HashMap，但为了实现LinkedHashMap维持节点顺序的特性，内部很多方法都自己实现了，下面我们以put方法开始说明。在这里只介绍LinkedHashMap重写的方法，关于HashMap中put方法的详细介绍，可以参考我的Java集合类学习之HashMap一文。    
```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //如果桶为空，调用newNode新建一个节点,LinkedHashMap重写了该方法
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //如果找到了节点
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            // LinkedHashMap重写了该方法
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    // LinkedHashMap重写了该方法
    afterNodeInsertion(evict);
    return null;
}
```

#### **重写的newNode()方法** 
从代码可以看到，如果插入K/V对时，桶中没有链表，那么使用newNode创建一个新节点，LinkedHashMap重写了该方法，其实现如下：
```java
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}

private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}
```
从上面代码可以看到，newNode()中方法中首先创建了一个以null为节点的Entry节点，然后调用linkNodeLast()方法将该结点添加到双端链表的尾节点。  

#### **重写的afterNodeInsertion(boolean evict)方法**
这个函数主要是用来在put操作之后进行可选的删除最近最少访问的节点。 afterNodeInsertion方法的evict参数如果为false，表示哈希表处于创建模式。只有在使用Map集合作为构造器创建LinkedHashMap或HashMap时才会为false，使用其他构造器创建的LinkedHashMap，之后再调用put方法，该参数均为true。LinkedHashMap的afterNodeInsertion()实现如下：
```java
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}
```
从上面可以看到，如果要进入到if语句块中需要同时满足三个条件：   
1. evict为true。只要不是构造方法中的插入Map集合，evict就为true，否则为false 
2. first!=null。表明表不为空，按理来说，当调用该方法时，哈希表不会为空 
3. removeEldestEntry()方法返回true。该方法删除删除最老的节点

LinkedHashMap的removeEldestEntry()方法的默认实现返回false，如下：  
```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```
所以上面就不会进入到if语句块中。removeElestEntry用于定义删除最老元素的规则。一旦需要删除最老节点，那么将会调用removeNode删除节点。 举个例子，如果一个链表只能维持100个元素，那么当插入了第101个元素时，以如下方式重写removeEldestEntry的话，那么将会删除最老的一个元素，如下：
```java
public boolean removeEldestEntry(Map.Entry<K,V> eldest) {
       return size() > 100;
}
```

#### **重写的afterNodeAccess(Node e)方法**
afterNodeAccess()在键值重复时，会调用该方法，其中参数e表示该节点。该方法用于再accessOrder为true时将节点移到最后，其实现如下：
```java
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    //如果accessOrder为true并且当前节点不是tail节点
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```
从上面代码可以看到，如果想进入到if语句块，那么必须同时满足两个条件：  
1. accessOrder为true。为true只能在三个参数的构造方法中指定accessOrder，表明按照LRU访问顺序管理节点。那么当键相同时，就相当于一次访问，所以可能需要将访问的节点移到双端链表的尾端。 
2. 如果当前节点不是尾节点。如果已经是尾节点，那么就无须移动。

从上面可以看到，一旦满足LinkedHashMap使用访问顺序管理链表时且当前节点不是尾节点，那么需要将节点移到尾节点，if语句块中的代码就是将一个节点从双端链表中移至尾端。  

_ _ _
### **get()操作** 
LinkedHashMap重写了HashMap的get方法，用于根据键得到值，如果哈希表中不包含该键，那么返回null，其实现如下：
```java
public V get(Object key) {
    Node<K,V> e;
    //如果哈希表中不存在该键，返回null
    if ((e = getNode(hash(key), key)) == null)
        return null;
    //如果accessOrder为true，即使用访问顺序
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
```
从上面代码可以看到，get()方法分为2步：  
1. 调用getNode()方法得到键对应的节点。如果节点为null，表明哈希表中不存在该键，那么返回null 
2. 如果哈希表中存在该键并且accessOrder为true，那么调用afterNodeAccess(e)将节点移到双端链表的尾部

_ _ _
### **remove()操作** 
LinkedHashMap的remove()方法根据键删除节点，如果哈希表中不存在键值，那么返回null。LinkedHashMap的remove()方法继承自HashMap的remove()方法，在将节点从链表或红黑树中移除后，调用afterNodeRemoval(Node e)方法，HashMap中afterNodeRemoval(Node e)方法默认实现为不做任何操作，LinkedHashMap重写了该方法，其实现如下：  
```java
// e表示待删除的节点
void afterNodeRemoval(Node<K,V> e) { // unlink
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    p.before = p.after = null;
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}
```  
从上面可以看到，afterNodeRemoval()方法主要就是将节点从双端链表中移除。  

关于LinkedHashMap的源码介绍就到这里。

(完)

参考文章：  
[LinkedHashMap源码分析](https://blog.csdn.net/qq_19431333/article/details/73927738)  
