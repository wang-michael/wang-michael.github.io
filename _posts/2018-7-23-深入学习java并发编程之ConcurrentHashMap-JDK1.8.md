---
layout:     post
title:      深入学习java并发编程之ConcurrentHashMap-JDK1.8
date:       2018-7-23
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 并发
    - JDK源码阅读
---

>本篇随笔主要记录了我阅读ConcurrentHashMap(Jdk1.8)源码期间的对于ConcurrentHashMap的一些实现上的个人理解，用于个人备忘，有不对的地方，请指出。  
  
_ _ _
### **前言**
相比于ConcurrentHashMap在JDK1.7中的实现，在JDK1.8中主要做了两方面的改进：    
1. 取消segments字段，直接采用transient volatile Node<K,V>[] table保存数据，采用table数组元素作为锁(使用CAS+Synchronized方式进行并发控制)，从而实现了对每一行数据进行加锁，减小了加锁粒度，进一步减少了并发冲突的概率。  
2. 类似于HashMap，将原先table数组＋单向链表的数据结构，变更为table数组＋单向链表＋红黑树的结构。对于hash表来说，最核心的能力在于将key hash之后能均匀的分布在数组中。如果hash之后散列的很均匀，那么table数组中的每个队列长度主要为0或者1。但实际情况并非总是如此理想，虽然ConcurrentHashMap类默认的加载因子为0.75，但是在数据量过大或者运气不佳的情况下，还是会存在一些队列长度过长的情况，如果还是采用单向列表方式，那么查询某个节点的时间复杂度为O(n)；因此，对于个数超过8(默认值)的列表，jdk1.8中采用了红黑树的结构，那么查询的时间复杂度可以降低到O(logN)，可以改进性能。  

下面就来具体了解下ConcurrentHashMap在JDK1.8中的具体实现。  

_ _ _
### **基础结构**
#### **键值对的存储**
```java
//ConcurrentHashMap类内部采用Node类存储键值对
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;  //采用volatile关键字修饰
    volatile Node<K,V> next;   //采用volatile关键字修饰

    Node(int hash, K key, V val, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.val = val;
        this.next = next;
    }

    public final K getKey()       { return key; }
    public final V getValue()     { return val; }
    public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
    public final String toString(){ return key + "=" + val; }
    public final V setValue(V value) {   //不支持修改value, 否则将会抛出异常
        throw new UnsupportedOperationException();
    }

    public final boolean equals(Object o) {
        Object k, v, u; Map.Entry<?,?> e;
        return ((o instanceof Map.Entry) &&
                (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                (v = e.getValue()) != null &&
                (k == key || k.equals(key)) &&
                (v == (u = val) || v.equals(u)));
    }
    //查找当前节点之后的链表,若是存在则返回相应的Node; 否则返回Null.
    Node<K,V> find(int h, Object k) {
        Node<K,V> e = this;
        if (k != null) {
            do {
                K ek;
                if (e.hash == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
            } while ((e = e.next) != null);
        }
        return null;
    }
}
```

#### **扩容操作用到的辅助节点类型**
```java
//ForwardingNode是Node的子类型
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;    //设置辅助扩容线程的下一段table

    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }

    Node<K,V> find(int h, Object k) {
        outer: for (Node<K,V>[] tab = nextTable;;) {
            Node<K,V> e; int n;
            if (k == null || tab == null || (n = tab.length) == 0 ||
                (e = tabAt(tab, (n - 1) & h)) == null)  //查找hash数组位置h处的Node
                return null;
            for (;;) {
                int eh; K ek;
                if ((eh = e.hash) == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))    //查找到key相同的Node
                    return e;
                if (eh < 0) {
                    if (e instanceof ForwardingNode) {
                        tab = ((ForwardingNode<K,V>)e).nextTable;   //递归查询下一个ForwardingNode
                        continue outer;
                    }
                    else
                        return e.find(h, k);   //查找链表
                }
                if ((e = e.next) == null)
                    return null;
            }
        }
    }
}
```

#### **类中成员变量**
```java
private transient volatile long baseCount; // 在ConcurrentHashMap内部使用了此变量来保存map中键值对个数

//当值为-1时, 代表数组正在被初始化;
//按照源码注释翻译，当值为-(1+扩容线程数), 代表数组正在被多个线程扩容。但是其实不是这样的，当线程进行扩容时，会根据resizeStamp函数生成一个基数戳rs，然后((rs<<RESIZE_STAMP_SHIFT)+n+1)这才是表示n个线程在扩容。
//当table为null时, 代表要初始化的容量大小; 否则代表下次要扩容的容量的阈值，即达到此容量后需要扩容
private transient volatile int sizeCtl;

//ConcurrentHashMap的最大容量 2^30
private static final int MAXIMUM_CAPACITY = 1 << 30;

//ConcurrentHashMap的默认容量 2^4
private static final int DEFAULT_CAPACITY = 16;

//hash值为-1处的节点代表forwarding node
static final int MOVED     = -1; 

//和key对应hash值进行与操作, 将hash值最高位置0，保证普通节点的hash值都是正数
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash

//用于生成当前数组对应的基数戳
private static int RESIZE_STAMP_BITS = 16;

//将基数戳左移的位数，保证左移后的基数戳为负值，然后再加上n+1,表示n个线程正在扩容
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

//表示最多能有多少个线程能够帮助进行扩容，因为sizeCtl只有低16位用于标识，所以最多只有2^16-1个线程帮助扩容
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

//数组位置中红黑树根节点的hash值为-2，小于0
static final int TREEBIN   = -2; 

//将HASH_BITS和普通节点的hash相与，将hash值最高位置0，从而保证普通节点的hash值都是>=0的
static final int HASH_BITS = 0x7fffffff; 

//扩容线程所负责的区间大小最低为16，避免发生大量的内存冲突
private static final int MIN_TRANSFER_STRIDE = 16;

//用于扩容过程中，指示原数组下一个分割区间的上界位置
private transient volatile int transferIndex;

//只有当数组处于扩容过程时，nextTable才不为null;否则其他时刻，nextTable为null;
//nextTable主要用于扩容过程中指向扩容后的新数组
private transient volatile Node<K,V>[] nextTable;

//节点数组，用于存储键值对，当第一次插入时进行初始化。
transient volatile Node<K,V>[] table;

private transient volatile CounterCell[] count erCells;// 可方便的计算hashmap中所有元素的个数，性能大大优于jdk1.7中的size()方法
```
对于sizeCtl，还需要着重记录一下其作用：
* 0：默认值
* -1：代表哈希表正在进行初始化
* 大于0：相当于 HashMap 中的 threshold，表示阈值
* 小于-1：代表有多个线程正在进行扩容

#### **构造方法**
```java
//默认构造方法
public ConcurrentHashMap() {
}

//用户自定义初始化容量作为参数
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));   //对用户输入的初始化容量修剪为2^n次方, 
    this.sizeCtl = cap; // 将sizeCtl初始化为map的初始容量
}

public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}

// 兼容JDK1.7的构造方法
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```

_ _ _
### **put方法实现**
```java
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS; // 得到的hash值最高位为0，确保普通节点的hash值一定是个正数
}

public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode()); // 经过spread方法处理后的hash值一定是个正数
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0) // 如果此时tab为空，则进行初始化，使用CAS操作保证多线程初始化的线程安全
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { // 表中对应位置值为null，则尝试使用CAS操作插入数据
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED) // 找到的节点是一个ForwardingNode节点
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 针对首个节点进行加锁操作，而不是segment，进一步减少线程冲突
            synchronized (f) {
                if (tabAt(tab, i) == f) { // 确保加锁前与加锁后tab[i]所代表的节点并未改变
                    if (fh >= 0) { // hash值为正数，代表是一个普通的节点
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) { // 遍历结束后binCount代表的是链表长度
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) { // 红黑树根节点hash值小于0
                        Node<K,V> p;
                        binCount = 2;
                        // 插入红黑树中
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD) // 是否需要将链表转化为红黑树
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount); //调用addCount函数，将容器大小加1，并判断是否需要进行扩容
    return null;
}
```

initTable方法实现如下：
```java
/**
 * Initializes table, using the size recorded in sizeCtl.
 */
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { // 使用CAS操作防止表被初始化多次
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc; // 将sizeCtl设置为扩容阈值
            }
            break;
        }
    }
    return tab;
}
```
为何红黑树根节点hash值小于0？？？  因为红黑树根节点是一个TreeBin类型的节点，构造函数中设置了其hash值为-2; 实际节点是TreeBin中的first节点。   
```java
static final int TREEBIN   = -2; // hash for roots of trees

static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K,V> root;
    volatile TreeNode<K,V> first;
    ...

    /**
     * Creates bin with initial set of nodes headed by b.
     */
    TreeBin(TreeNode<K,V> b) {
        super(TREEBIN, null, null, null);
        this.first = b;
        ...
    }
}        
```

_ _ _
### **扩容过程实现**
首先需要介绍一下，ForwardingNode 这个节点类型：
```java
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        //注意这里
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }
    Node<K,V> find(int h, Object k) { // 何时会用到这个方法？？？是不是get方法时候定位到的节点是一个ForwardingNode节点时会用到？？？  
        // loop to avoid arbitrarily deep recursion on forwarding nodes
        outer: for (Node<K,V>[] tab = nextTable;;) {
            Node<K,V> e; int n;
            if (k == null || tab == null || (n = tab.length) == 0 ||
                (e = tabAt(tab, (n - 1) & h)) == null)
                return null;
            for (;;) {
                int eh; K ek;
                if ((eh = e.hash) == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
                if (eh < 0) {
                    if (e instanceof ForwardingNode) {
                        tab = ((ForwardingNode<K,V>)e).nextTable;
                        continue outer;
                    }
                    else
                        return e.find(h, k);
                }
                if ((e = e.next) == null)
                    return null;
            }
        }
    }
}
```
这个节点内部保存了一 nextTable 引用，它指向一张 hash 表。在扩容操作中，我们需要对每个桶中的结点进行分离和转移，如果某个桶结点中所有节点都已经迁移完成了（已经被转移到新表 nextTable 中了），那么会在原 table 表的该位置挂上一个 ForwardingNode 结点，说明此桶已经完成迁移。  

ForwardingNode 继承自 Node 结点，并且它唯一的构造函数将构建一个键，值，next 都为 null 的结点，反正它就是个标识，无需那些属性。但是 hash 值却为 MOVED（-1）。  

所以，我们在 putVal 方法中遍历整个 hash 表的桶结点，如果遇到 hash 值等于 MOVED，说明已经有线程正在扩容 rehash 操作，整体上还未完成，不过我们要插入的桶的位置已经完成了所有节点的迁移。  

由于检测到当前哈希表正在扩容，于是让当前线程去协助扩容。  
```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        //返回一个 16 位长度的扩容校验标识
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            //sizeCtl 如果处于扩容状态的话
            //前 16 位是数据校验标识，后 16 位是当前正在扩容的线程总数
            //这里判断校验标识是否相等，如果校验符不等或者扩容操作已经完成了(即sc == rs + 1或者transferIndex <= 0)，直接退出循环，不用协助它们扩容了
            // sizeCtl 为 (rs<<RESIZE_STAMP_SHIFT)+n+1 表示有n个线程在帮助扩容
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            //否则调用 transfer 帮助它们进行扩容
            //sc + 1 标识增加了一个线程进行扩容
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```
下面我们看这个稍显复杂的 transfer 方法，我们分几个部分来细说。  
```java
//第一部分
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        //计算单个线程允许处理的最少table桶首节点个数，不能小于 16
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; 
        //刚开始扩容，初始化 nextTab 
        if (nextTab == null) {
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            //transferIndex 指向最后一个桶，方便从后向前遍历 
            transferIndex = n;
        }
        int nextn = nextTab.length;
        //定义 ForwardingNode 用于标记迁移完成的桶
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
```
如何保证nextTab的初始化由单线程执行？  

所有调用transfer的方法（例如helperTransfer、addCount)几乎都预先判断了nextTab!=null,而nextTab只会在transfer方法中初始化，保证了第一个进来的线程初始化之后其他线程才能进入。  

总体来说这部分代码还是比较简单的，主要完成的是对单个线程能处理的最少桶结点个数的计算和一些属性的初始化操作。  
```java
//第二部分，并发扩容控制的核心
boolean advance = true;
boolean finishing = false;
//i 指向当前桶，bound 指向当前线程需要处理的桶结点的区间下限
for (int i = 0, bound = 0;;) {
       Node<K,V> f; int fh;
       //这个 while 循环的目的就是通过 --i 遍历当前线程所分配到的桶结点
       //一个桶一个桶的处理
       while (advance) {
           int nextIndex, nextBound;
           if (--i >= bound || finishing)
               advance = false;
           //transferIndex <= 0 说明已经没有需要迁移的桶了
           else if ((nextIndex = transferIndex) <= 0) {
               i = -1;
               advance = false;
           }
           //更新 transferIndex
           //为当前线程分配任务，处理的桶结点区间为（nextBound,nextIndex）
           else if (U.compareAndSwapInt(this, TRANSFERINDEX, nextIndex,nextBound = (nextIndex > stride ? nextIndex - stride : 0))) {
               bound = nextBound;
               i = nextIndex - 1;
               advance = false;
           }
       }
       //当前线程所有任务完成
       if (i < 0 || i >= n || i + n >= nextn) { 
           int sc;
           if (finishing) {
               nextTable = null;
               table = nextTab;
               sizeCtl = (n << 1) - (n >>> 1);
               return;
           }
           if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
               if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                   return;
               finishing = advance = true;
               i = n; 
           }
       }
       //待迁移桶为空，那么在此位置 CAS 添加 ForwardingNode 结点标识该桶已经被处理过了
       else if ((f = tabAt(tab, i)) == null)
           advance = casTabAt(tab, i, null, fwd);
       //如果扫描到 ForwardingNode，说明此桶已经被处理过了，跳过即可  为什么会出现这种情况呢？？？  
       else if ((fh = f.hash) == MOVED)
           advance = true;
```
多个线程同时对tab进行扩容的核心思想就是为每个线程在数组中分配其需要工作的扩容区间，每个新参加进来扩容的线程必然先进 while 循环的最后一个判断条件中去领取自己需要迁移的桶的区间即（nextBound,nextIndex）。然后 i 指向区间的最后一个位置，表示迁移操作从后往前的做。接下来的几个判断就是实际的迁移结点操作了。等我们大致介绍完成第三部分的源码再回来对各个判断条件下的迁移过程进行详细的叙述。  

举例来说，数组长度为48，第一个扩容线程分配处理的区间是32到47，第二个就为16到31，由数组从后向前逐渐给新到来的线程分配空间进行处理。    

```java
//第三部分
else {
    //
    synchronized (f) {
        if (tabAt(tab, i) == f) {
            Node<K,V> ln, hn;
            //链表的迁移操作
            if (fh >= 0) {
                int runBit = fh & n;
                Node<K,V> lastRun = f;
                //整个 for 循环为了找到整个桶中最后连续的 fh & n 不变的结点
                for (Node<K,V> p = f.next; p != null; p = p.next) {
                    int b = p.hash & n;
                    if (b != runBit) {
                        runBit = b;
                        lastRun = p;
                    }
                }
                if (runBit == 0) {
                    ln = lastRun;
                    hn = null;
                }
                else {
                    hn = lastRun;
                    ln = null;
                }
                //如果fh&n不变的链表的runbit都是0，则nextTab[i]内元素ln前逆序，ln及其之后顺序
                //否则，nextTab[i+n]内元素全部相对原table逆序
                //这是通过一个节点一个节点的往nextTab添加
                for (Node<K,V> p = f; p != lastRun; p = p.next) {
                    int ph = p.hash; K pk = p.key; V pv = p.val;
                    if ((ph & n) == 0)
                        ln = new Node<K,V>(ph, pk, pv, ln);
                    else
                        hn = new Node<K,V>(ph, pk, pv, hn);
                }
                //把两条链表整体迁移到nextTab中
                setTabAt(nextTab, i, ln);
                setTabAt(nextTab, i + n, hn);
                //将原桶标识位已经处理
                setTabAt(tab, i, fwd);
                advance = true;
            }
            //红黑树的复制算法，不再赘述
            else if (f instanceof TreeBin) {
                TreeBin<K,V> t = (TreeBin<K,V>)f;
                TreeNode<K,V> lo = null, loTail = null;
                TreeNode<K,V> hi = null, hiTail = null;
                int lc = 0, hc = 0;
                for (Node<K,V> e = t.first; e != null; e = e.next) {
                    int h = e.hash;
                    TreeNode<K,V> p = new TreeNode<K,V>(h, e.key, e.val, null, null);
                    if ((h & n) == 0) {
                        if ((p.prev = loTail) == null)
                            lo = p;
                        else
                            loTail.next = p;
                    loTail = p;
                    ++lc;
                    }
                    else {
                        if ((p.prev = hiTail) == null)
                            hi = p;
                        else
                            hiTail.next = p;
                    hiTail = p;
                    ++hc;
                    }
                }
                ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :(hc != 0) ? new TreeBin<K,V>(lo) : t;
                hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :(lc != 0) ? new TreeBin<K,V>(hi) : t;
                setTabAt(nextTab, i, ln);
                setTabAt(nextTab, i + n, hn);
                setTabAt(tab, i, fwd);
                advance = true;
           }

```
那么至此，有关迁移的几种情况已经介绍完成了，下面我们整体上把控一下整个扩容和迁移过程。  

首先，每个线程进来会先领取自己的任务区间，然后开始 --i 来遍历自己的任务区间，对每个桶进行处理。如果遇到桶的头结点是空的，那么使用 ForwardingNode 标识该桶已经被处理完成了。如果遇到已经处理完成的桶(**为什么会遇到已经处理完成的桶呢？各个线程的处理区间不是互不冲突吗？？？**)，直接跳过进行下一个桶的处理。如果是正常的桶，对桶首节点加锁，正常的迁移即可，迁移结束后依然会将原表的该位置标识位已经处理。  

当 i < 0，说明本线程处理速度够快的，整张表的最后一部分已经被它处理完了，现在需要看看是否还有其他线程在自己的区间段还在迁移中。这是退出的逻辑判断部分：
```java
while (advance) {
    int nextIndex, nextBound;
    if (--i >= bound || finishing)
        advance = false;
    else if ((nextIndex = transferIndex) <= 0) { 
        // transferIndex代表已经分配出去的下界，比如数组中第16个元素到最后一个元素已经分配给了帮助线程进行处理，那么transferIndex就为15
        i = -1;
        advance = false;
    }
    else if (U.compareAndSwapInt
             (this, TRANSFERINDEX, nextIndex,
              nextBound = (nextIndex > stride ?
                           nextIndex - stride : 0))) {
        bound = nextBound;
        i = nextIndex - 1;
        advance = false;
    }
}
if (i < 0 || i >= n || i + n >= nextn) { // i < 0可能是因为你上述while循环中的第一个else if导致的，那何时会出现i >= n 或者 i + n >= nextn 呢？？？
    int sc;
    if (finishing) {
        nextTable = null;
        table = nextTab;
        sizeCtl = (n << 1) - (n >>> 1);
        return;
    }
    if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
        if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
            return;
        finishing = advance = true;
        i = n; // recheck before commit
    }
}
```
finnish 是一个标志，如果为 true 则说明整张表的迁移操作已经全部完成了，我们只需要重置 table 的引用并将 nextTable 赋为空即可。否则，CAS 式的将 sizeCtl 减一，表示当前线程已经完成了任务，退出扩容操作。

如果退出成功，那么需要进一步判断是否还有其他线程仍然在执行任务。
```java
if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
   return;
```
我们说过 resizeStamp(n) 返回的是对 n 的一个数据校验标识，占 16 位。而 RESIZE_STAMP_SHIFT 的值为 16，那么位运算后，整个表达式必然在右边空出 16 个零。也正如我们所说的，sizeCtl 的高 16 位为数据校验标识，低 16 为表示正在进行扩容的线程数量。

(resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2 表示当前只有一个线程正在工作，相对应的，如果 (sc - 2) == resizeStamp(n) << RESIZE_STAMP_SHIFT，说明当前线程就是最后一个还在扩容的线程，那么会将 finishing 标识为 true，并在下一次循环中退出扩容方法。

这一块的难点在于对 sizeCtl 的各个值的理解，关于它的深入理解，这里推荐一篇文章。
[着重理解位操作](http://wuzhaoyang.me/2016/09/05/java-collection-map-2.html)  

看到这里，真的为 Doug Lea 精妙的设计而折服，针对于多线程访问问题，不但没有拒绝式得将他们阻塞在门外，反而邀请他们来帮忙一起工作。  

_ _ _
### **addCount方法实现**
回到我们之前分析的 putVal 方法。接着前文的分析，当我们根据 hash 值，找到对应的桶结点，如果发现该结点为 ForwardingNode 结点，表明当前的哈希表正在扩容和 rehash，于是将本线程送进去帮忙扩容。否则如果是普通的桶结点，于是锁住该桶，分链表和红黑树的插入一个节点，具体插入过程类似 HashMap，此处不再赘述。  

当我们成功的添加完成一个结点，最后是需要判断添加操作后是否会导致哈希表达到它的阈值，并针对不同情况决定是否需要进行扩容，还有 CAS 式更新哈希表实际存储的键值对数量。这些操作都封装在 addCount 这个方法中，当然 putVal 方法的最后必然会调用该方法进行处理。下面我们看看该方法的具体实现，该方法主要做两个事情。一是更新 baseCount，二是判断是否需要扩容。  

addCount函数的源代码如下：  
```java
//如果数组太小并且没有扩容，那么启动扩容。如果正在扩容，帮忙一起扩容。
//每次扩容后检查占用率是否需要进行再一次扩容，因为扩容滞后于添加元素。
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;

    //baseCount更新失败，则使用counterCells,调用 fullAddCount 将这些失败的结点包装成一个 CounterCell 对象，保存在 CounterCell 数组中。
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {  //利用CAS操作更新baseCount
        CounterCell a; long v; int m;
        //baseCount更新失败,CAS更新CounterCell数组的元素值＋x，uncontended表示更新CounterCell的争用
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
             //如果这也失败，说明这个数组元素也被争用了。必须要动粗了
             //fullAddCount实现思想同LongAdder，这个类也是1.8加入的。
             //作用就是将x加到counterCells数组中或baseCount中
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        //统计元素的个数
        s = sumCount();
    }

    if (check >= 0) {  //判断是否需要扩容
        Node<K,V>[] tab, nt; int n, sc;
        //元素个数>扩容阈值，并且tab不为空
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);  //生成一个基数戳
            //如果正在扩容
            if (sc < 0) {
         			//本轮扩容结束或没有桶可分配，线程离开
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                // sc + 1表示扩容线程＋1
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))  //将sizeCtl加1，表示新增加一个线程进行辅助操作
                    transfer(tab, nt);
            }
            //第一个监测到要扩容的线程进来，设置 (rs << RESIZE_STAMP_SHIFT) + 2
            //表示现在只有一个线程在扩容，也就是当前进来的线程
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))  //基数戳rs<<RESIZE_STAMP_SHIFT变为负数，然后+2，赋值给sizeCtl，代表有一个线程将要扩容，此后，每增加一个线程辅助扩容，将sizeCtl值加1.
                transfer(tab, null);
            //统计个数，继续循环检测
            s = sumCount();
        }
    }
}
```  
待分析：addCount方法，tryPreSize方法。  

_ _ _
### **get方法实现**
```java
// get方法整体不加锁
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0) //说明该节点位置为红黑树节点
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```
如果get时节点hash值为-1怎么办？？？  

_ _ _
### **size方法实现**
```java
public int size() {
    long n = sumCount();  //调用内部sumCount方法
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;  
}
```

(完)
参考文章：
[为并发而生的 ConcurrentHashMap（Java 8）](https://www.jianshu.com/p/e99e3fcface4)  