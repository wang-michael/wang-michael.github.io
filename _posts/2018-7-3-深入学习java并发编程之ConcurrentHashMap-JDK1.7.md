---
layout:     post
title:      深入学习java并发编程之ConcurrentHashMap-JDK1.7
date:       2018-7-3
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 并发
    - JDK源码阅读
---

>本篇随笔主要记录了我阅读ConcurrentHashMap(Jdk1.7)源码期间的对于ConcurrentHashMap的一些实现上的个人理解，用于个人备忘，有不对的地方，请指出。  
  
_ _ _
### **前言**
在并发编程中使用HashMap可能导致程序死循环，而使用线程安全的HashTable效率又非常低下，基于以上两个原因，便有了ConcurrentHashMap的登场机会。  

HashTable容器在激烈竞争的并发环境下表现效率低下的原因是所有访问HashTable的线程都必须竞争同一把锁，假如容器里有多把锁，每一把锁用来锁容器中一部分数据，那么当多线程访问容器里不同数据段的数据时，线程间就不会存在锁竞争，从而可以有效提升并发访问效率，这就是ConcurrentHashMap所使用的锁分段技术。首先将数据分成一段一段的存储，然后给每一段数据配一把锁，当一个线程访问其中一个段的数据的时候，其它段的数据也能被其它线程访问。  

下面是JDK1.7实现中ConcurrentHashMap数据结构图：  

<img src="/img/2018-7-3/concurrentHashMapStructure.png" width="500" height="500" alt="ConcurrentHashMap数据结构图" />
<center>图1：ConcurrentHashMap数据结构图</center>   

ConcurrentHashMap的key与value都不允许为null。  

ConcurrentHashMap中有些方法需要跨段，比如size()和containsValue()，它们可能需要锁定整个表而而不仅仅是某个段，这需要按顺序锁定所有段，操作完毕后，又按顺序释放所有段的锁。这里“按顺序”是很重要的，否则极有可能出现死锁。    

下面就来分析下ConcurrentHashMap的内部实现。    

_ _ _
### **ConcurrentHashMap初始化**
```java
// 可指定三个参数：初始容量、负载因子、并发级别
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS) // 最大并发级别65535
        concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1; // 使用的实际并发级别是ssize，一定是最接近concurrencyLevel的2的整数次幂
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    this.segmentShift = 32 - sshift; // hashcode最多为32位，计算段偏移量
    this.segmentMask = ssize - 1; // 计算段掩码
    if (initialCapacity > MAXIMUM_CAPACITY) 
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize; // 根据给定的初始容量和段数量计算每个段的容量
    if (c * ssize < initialCapacity)
        ++c;
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    while (cap < c)
        cap <<= 1;
    // create segments and segments[0] 仅先创建第0段，其它段用到的时候再创建
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize]; // 保存所有段的段数组
    // 使用UNSAFE类的putOrderedObject方法将第0段插入段数组中，这里禁止此写操作与之前的写操作之间的重排序有何意义呢?  
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0] 
    this.segments = ss;
}
```
初始化过程中需要注意的有以下几点：  
1. 我认为concurrencyLevel的意义可以理解为允许最多同时对ConcurrentHashMap做更新操作的段的数量，根据这个参数确定segment的数量
2. 上面计算后保存的段偏移量segmentShift和段掩码segmentMask是用来插入元素时定位元素所在的段的，具体如何定位接下来介绍
3. 向段数组中插入元素不是直接使用ss[0] = s0的形式，而是使用了UNSAFE.putOrderedObject(ss, SBASE, s0);经过查找相关资料，我认为其用处是通过在执行方法时插入putOrderedObject方法时插入StoreStore内存屏障，避免发生写操作重排序。对于这里的写操作没有采用putObjectVolatile方法的原因我认为是避免执行后插入StoreLoad屏障的开销(putOrderedObject方法是不会插入这个屏障的)；为什么这个开销可以避免呢？会不会引起读此变量时的内存可见性问题呢？答案是不会，因为put操作中有获取锁和释放锁的操作，JMM会保证获得锁到释放锁之间所有对象的状态更新都会在锁被释放之后更新到主存，从而保证这些变更对其他线程是可见的。 
   
**putOrderedObject方法的写入效率比volatile写更高效，因为它们的写入其实只是更新的本地缓存，没有立即回写至主内存，所以它们的操作结果也不是立即可见的，另外它还附带禁止对该操作前面的其他写操作的重排序，除非后面存在volatile写或者synchronize块或者ReentrantLock之类的锁的获取-释放语义导致它们会被立即刷新回主存使其立即对其他处理器可见，否则从理论上来说，其可见性很可能将会被无限期的推迟，当然这最终还是取决于平台的缓存一致性策略。**  

顺便说下UNSAFE.putOrderedObject(ss, SBASE, s0)的使用方法。其中第一个参数是要放入参数的对象，这里是一个segment数组对象ss；第二个参数是从此对象起始内存地址向后偏移多少个字节来放入指定对象，由于这里插入的是第一个元素，所以只需要偏移数组的头部长度即可，一般是16或者24个字节；第三个参数是要放入的对象，这里是s0。  

SBASE是在ConcurrentHashMap类初始化过程中确定的：  
```java
// Unsafe mechanics
private static final sun.misc.Unsafe UNSAFE;
private static final long SBASE;
private static final int SSHIFT;
private static final long TBASE;
private static final int TSHIFT;
private static final long HASHSEED_OFFSET;
private static final long SEGSHIFT_OFFSET;
private static final long SEGMASK_OFFSET;
private static final long SEGMENTS_OFFSET;

static {
    int ss, ts;
    try {
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class tc = HashEntry[].class;
        Class sc = Segment[].class;
        TBASE = UNSAFE.arrayBaseOffset(tc);
        SBASE = UNSAFE.arrayBaseOffset(sc);
        ts = UNSAFE.arrayIndexScale(tc);
        ss = UNSAFE.arrayIndexScale(sc);
        HASHSEED_OFFSET = UNSAFE.objectFieldOffset(
            ConcurrentHashMap.class.getDeclaredField("hashSeed"));
        SEGSHIFT_OFFSET = UNSAFE.objectFieldOffset(
            ConcurrentHashMap.class.getDeclaredField("segmentShift"));
        SEGMASK_OFFSET = UNSAFE.objectFieldOffset(
            ConcurrentHashMap.class.getDeclaredField("segmentMask"));
        SEGMENTS_OFFSET = UNSAFE.objectFieldOffset(
            ConcurrentHashMap.class.getDeclaredField("segments"));
    } catch (Exception e) {
        throw new Error(e);
    }
    if ((ss & (ss-1)) != 0 || (ts & (ts-1)) != 0)
        throw new Error("data type scale not a power of two");
    SSHIFT = 31 - Integer.numberOfLeadingZeros(ss);
    TSHIFT = 31 - Integer.numberOfLeadingZeros(ts);
}
```
确定SBASE的方法是UNSAFE.arrayBaseOffset(sc);此方法的作用是确定某个类的数组对象的头部大小，64位机器上，数组对象的对象头占用24个字节，启用压缩之后占用16个字节。之所以比普通对象占用内存多是因为需要额外的空间存储数组的长度。  

UNSAFE.arrayIndexScale(sc)的作用是确定数组中每个元素需要占用的字节数。  

_ _ _
### **ConcurrentHashMap定位Segment方式**
以get方法为例，定位Segment的过程如下：  
```java
int h = hash(key);
long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u))

private int hash(Object k) {
    int h = hashSeed;

    if ((0 != h) && (k instanceof String)) {
        return sun.misc.Hashing.stringHash32((String) k);
    }

    h ^= k.hashCode();

    // Spread bits to regularize both segment and index locations,
    // using variant of single-word Wang/Jenkins hash.
    h += (h <<  15) ^ 0xffffcd7d;
    h ^= (h >>> 10);
    h += (h <<   3);
    h ^= (h >>>  6);
    h += (h <<   2) + (h << 14);
    return h ^ (h >>> 16);
}
```
这个过程中需要注意的是：  
1. 定位segment的元素的hash值是在其hashCode的基础上又使用Wang/Jenkins hash的变种算法对元素的hashCode进行了一次再散列。之所以进行再散列，目的是减少散列冲突，使元素能够均匀的分布在不同的Segment上，从而提高容器的存取效率。假如散列的质量差到极点，那么所有的元素都在一个Segment中，不仅存取元素缓慢，分段锁也会失去意义。  
2. 假设concurrencyLevel为16，那么segmentShift为28，segmentMask为15，确定段所在位置的方式是(h >>> segmentShift) & segmentMask，可见只有hash值的高4位参与到了确定段所在位置的计算当中去，如果这里的hash值使用的不是再散列之后的hashCode而是普通的hashCode，那么只要两个元素的hashCode的高四位相同，就会被定位到一个段中去，导致元素不能均匀的分到不同的段中去，分段的意义也就不大了。通过再hash使得hashCode中每一位的值都可能成为高四位的值，让数字的每一位都参加到散列运算当中，从而减少散列冲突。  

_ _ _
### **ConcurrentHashMap put操作**
```java
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>
        implements ConcurrentMap<K, V>, Serializable {

    public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key); // 对元素进行再hash
        int j = (hash >>> segmentShift) & segmentMask; // 确定元素应该放在哪个段
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            s = ensureSegment(j); // 确保段存在
        return s.put(key, hash, value, false); // 向段中放置元素的实际方法
    }

    static final class Segment<K,V> extends ReentrantLock implements Serializable {
        final V put(K key, int hash, V value, boolean onlyIfAbsent) {
            HashEntry<K,V> node = tryLock() ? null :
                scanAndLockForPut(key, hash, value); // 获取锁
            V oldValue;
            try {
                HashEntry<K,V>[] tab = table;
                // 与将元素散列到segmemnt时使用的算法不同，目的是使散列到一个segment的元素散列到不同entry中
                int index = (tab.length - 1) & hash; 
                HashEntry<K,V> first = entryAt(tab, index); 
                for (HashEntry<K,V> e = first;;) {
                    if (e != null) { // 从首节点之后向后查找，找到则更新
                        K k;
                        if ((k = e.key) == key ||
                            (e.hash == hash && key.equals(k))) {
                            oldValue = e.value;
                            if (!onlyIfAbsent) {
                                e.value = value; // HashEntry中的value声明为volatile的，保证对其更新立即可见
                                ++modCount;
                            }
                            break;
                        }
                        e = e.next;
                    }
                    else { // 散列到的HashEntry中不存在元素
                        if (node != null)
                            node.setNext(first); 
                        else
                            node = new HashEntry<K,V>(hash, key, value, first);
                        int c = count + 1;
                        if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                            rehash(node); // 在插入元素之前判断是否需要rehash，避免了无用rehash操作(即rehash之后可能并没有元素插入)
                        else
                            setEntryAt(tab, index, node);
                        ++modCount;
                        count = c;
                        oldValue = null;
                        break;
                    }
                }
            } finally {
                unlock();
            }
            return oldValue;
        }
    }

    // 尝试获取锁的过程中扫描出一个包含给定key值的节点，如果没有找到这样的节点则新创建一个并返回。  
    // 这个方法返回的时候保证锁已经被获取到了
    // 之所以在获取锁的过程中对整个链表进行遍历，主要目的是希望遍历的链表被CPU cache所缓存，为后续实际put过程中的链表遍历操作提升性能。
    private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
        HashEntry<K,V> first = entryForHash(this, hash);
        HashEntry<K,V> e = first;
        HashEntry<K,V> node = null;
        int retries = -1; // negative while locating node
        while (!tryLock()) { 
            HashEntry<K,V> f; // to recheck first below
            if (retries < 0) { // 先从头结点开始向后遍历，直到找到节点或遍历到最后
                if (e == null) {
                    if (node == null) // speculatively create node
                        node = new HashEntry<K,V>(hash, key, value, null);
                    retries = 0;
                }
                else if (key.equals(e.key))
                    retries = 0;
                else
                    e = e.next;
            }
            else if (++retries > MAX_SCAN_RETRIES) { // 如果尝试次数已经到达最大值
                lock(); // 阻塞获取锁
                break;
            }
            else if ((retries & 1) == 0 &&
                     (f = entryForHash(this, hash)) != first) { // 尝试获取锁的过程中判断上次获取到锁的更新操作是否改变了first节点
                e = first = f; // re-traverse if entry changed 如果first节点更新了，则重新遍历
                retries = -1;
            }
        }
        // 有可能出现的是上次更新操作插入的节点的key值与本次想要插入的key值相同，但是本次查找并未找到此key值，反而返回了一个新创建的节点
        return node; // 如果找到了包含指定值的节点，返回值为null，否则返回新创建的尚未挂接到entry数组上的节点
    }

    // 通过在执行方法时插入putOrderedObject方法时插入StoreStore内存屏障，避免发生写操作重排序。
    static final <K,V> void setEntryAt(HashEntry<K,V>[] tab, int i,
                                       HashEntry<K,V> e) {
        UNSAFE.putOrderedObject(tab, ((long)i << TSHIFT) + TBASE, e);
    }
}
```

_ _ _
### **ConcurrentHashMap get操作**
```java
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>
        implements ConcurrentMap<K, V>, Serializable {

    public V get(Object key) {
        Segment<K,V> s; // manually integrate access methods to reduce overhead
        HashEntry<K,V>[] tab;
        int h = hash(key);
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE; // 确定所在段在段数组中的偏移量
        // 使用getObjectVolatile确保每次从主内存中加载此变量
        if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
            (tab = s.table) != null) {
            // 遍历segment下的entry数组
            for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                     (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
                 e != null; e = e.next) {
                K k;
                if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                    return e.value;
            }
        }
        return null;
    }
}
```
对于Segments数组和HashEntry数组，通过UNSAFE.getObjectVolatile方法实现数组内元素的volatile特性;  

对于HashEntry内的value值，通过将其声明为volatile保证对其进行的更新对于接下来的读操作可见。  

get到的元素恰好被删除了怎么办？如果此节点引用已经被获取到了，那么不影响此次get操作的结果，因为remove操作仅是将此被删除的HashEntry节点的前驱节点的next执行它的next节点，对其内部的key、value值并没有做任何改动。  

get与containsKey两个方法几乎完全一致：他们都没有使用锁，而是通过Unsafe对象的getObjectVolatile()方法提供的原子读语义，来获得Segment以及对应的链表，然后对链表遍历判断是否存在key相同的节点以及获得该节点的value。但由于遍历过程中其他线程可能对链表结构做了调整，因此get和containsKey返回的可能是过时的数据(比如获得的结果为null，实际在遍历过程中有其它线程恰好插入了这个元素)，这一点是ConcurrentHashMap在弱一致性上的体现。如果要求强一致性，那么必须使用Collections.synchronizedMap()方法。   

_ _ _
### **ConcurrentHashMap rehash及size操作**
在扩容的时候，首先会创建一个容量是原来容量两倍的数组，然后将原数组里的元素进行再散列后插入到新的数组里。为了高效，ConcurrentHashMap不会对整个容器进行扩容，而只对某个segment进行扩容。具体扩容操作代码如下：  
```java
/**
 * Segments are specialized versions of hash tables.  This
 * subclasses from ReentrantLock opportunistically, just to
 * simplify some locking and avoid separate construction.
 */
static final class Segment<K,V> extends ReentrantLock implements Serializable {

    /**
     * Doubles size of table and repacks entries, also adding the
     * given node to new table
     * 扩容之后会加入传入的node节点
     */
    @SuppressWarnings("unchecked")
    private void rehash(HashEntry<K,V> node) {
        
        HashEntry<K,V>[] oldTable = table;
        int oldCapacity = oldTable.length;
        int newCapacity = oldCapacity << 1;
        threshold = (int)(newCapacity * loadFactor);
        HashEntry<K,V>[] newTable =
            (HashEntry<K,V>[]) new HashEntry[newCapacity]; // 扩容
        int sizeMask = newCapacity - 1;
        for (int i = 0; i < oldCapacity ; i++) { // 重新散列
            HashEntry<K,V> e = oldTable[i];
            /**
             *Doug Lea为rehash做了一定的优化，避免让所有的节点都进行复制操作：由于扩容是基于2的幂指来操作，假设扩容前某HashEntry对应到Segment中  数组的index为i，数组的容量为capacity，那么扩容后该HashEntry对应到新数组中的index只可能为i或者i+capacity，因此大多数HashEntry节点在扩容前后index可以保持不变。基于此，rehash方法中会定位第一个后续所有节点在扩容后index都保持不变的节点，然后将这个节点之前的所有节点重排即可。
             */
            if (e != null) {
                HashEntry<K,V> next = e.next;
                int idx = e.hash & sizeMask;
                if (next == null)   //  Single node on list
                    newTable[idx] = e;
                else { // Reuse consecutive sequence at same slot
                    HashEntry<K,V> lastRun = e;
                    int lastIdx = idx;
                    // 定位第一个后续所有节点在扩容后index都保持不变的节点
                    for (HashEntry<K,V> last = next;
                         last != null;
                         last = last.next) {
                        int k = last.hash & sizeMask;
                        if (k != lastIdx) {
                            lastIdx = k;
                            lastRun = last;
                        }
                    }                    
                    newTable[lastIdx] = lastRun;
                    // Clone remaining nodes 将这个节点之前的所有节点重排
                    for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                        V v = p.value;
                        int h = p.hash;
                        int k = h & sizeMask;
                        HashEntry<K,V> n = newTable[k];
                        newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                    }
                }
            }
        }
        int nodeIndex = node.hash & sizeMask; // add the new node 插入新节点
        node.setNext(newTable[nodeIndex]);
        newTable[nodeIndex] = node;
        table = newTable;
    }
}
```  

如果要统计整个ConcurrentHashMap里元素的大小，就必须统计所有Segment里元素的大小后求和，即将每个Segment内的变量count相加。但问题是虽然相加时可以获取每个Segment的count的最新值，但是可能累加前使用的count发生了变化，那么统计结果就不准了。所以最安全的做法就是在统计size的时候把所有Segment的put、remove、clean方法全部锁住，但是这种做法显然非常低效。  

因为在累加count操作过程中，之前累加过的count发生变化的几率非常小，所以ConcurrentHashMap的做法是先尝试3次通过不锁住Segment的方式来统计各个Segment的大小，如果统计的过程中，容器的count发生了变化，再采用加锁的方式来统计所有Segment的大小。    

那么ConcurrenHashMap是如何判断在统计的时候容器是否发生了变化呢？使用modCount变量，在put、remove、clean方法里操作元素前都会将变量modCount进行加1，那么在统计size前后比较modCount是否发生变化，从而得知容器的大小是否发生变化。  

我想这里即使第一次无锁统计与第二次无锁统计的modCount值相同，但还是有可能出现第二次统计时累加前使用的count发生了变化，统计的结果不准确，只是相比于一次统计将概率变小了。  

具体代码如下：  
```java
public int size() {
    // Try a few times to get accurate count. On failure due to
    // continuous async changes in table, resort to locking.
    final Segment<K,V>[] segments = this.segments;
    int size;
    boolean overflow; // true if size overflows 32 bits
    long sum;         // sum of modCounts
    long last = 0L;   // previous sum
    int retries = -1; // first iteration isn't retry
    try {
        for (;;) {
            if (retries++ == RETRIES_BEFORE_LOCK) { // 在锁住所有段之前先尝试无锁统计
                for (int j = 0; j < segments.length; ++j) // 锁住当前存在的所有段
                    ensureSegment(j).lock(); // force creation
            }
            sum = 0L;
            size = 0;
            overflow = false;
            for (int j = 0; j < segments.length; ++j) {
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount; // Segment中的modCount和count变量都是非volatile的，这里如何在无锁统计时保证其可见性呢？？？
                    int c = seg.count;
                    if (c < 0 || (size += c) < 0)
                        overflow = true;
                }
            }
            if (sum == last) // 比较本次统计modCount结果与上次统计结果是否相同
                break;
            last = sum;
        }
    } finally {
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();
        }
    }
    return overflow ? Integer.MAX_VALUE : size;
}

static final <K,V> Segment<K,V> segmentAt(Segment<K,V>[] ss, int j) {
    long u = (j << SSHIFT) + SBASE;
    return ss == null ? null :
        (Segment<K,V>) UNSAFE.getObjectVolatile(ss, u); // 在这里getObjectVolatile可以保证其内部成员变量例如modCount获取到的值也是最新的吗？？？
}
```

_ _ _
### **ConcurrentHashMap remove操作**
直接看代码：
```java
public V remove(Object key) {
    int hash = hash(key);
    Segment<K,V> s = segmentForHash(hash); // 找到所在段
    return s == null ? null : s.remove(key, hash, null);
}

 /**
 * Remove; match on key only if value null, else match both.
 */
final V remove(Object key, int hash, Object value) {
    if (!tryLock()) // 对段加锁
        scanAndLock(key, hash);
    V oldValue = null;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> e = entryAt(tab, index);
        HashEntry<K,V> pred = null;
        while (e != null) {
            K k;
            HashEntry<K,V> next = e.next;
            if ((k = e.key) == key ||
                (e.hash == hash && key.equals(k))) {
                V v = e.value;
                if (value == null || value == v || value.equals(v)) { 
                    // 链表中删除Entry
                    if (pred == null)
                        setEntryAt(tab, index, next);
                    else
                        pred.setNext(next);
                    ++modCount;
                    --count;
                    oldValue = v;
                }
                break;
            }
            pred = e;
            e = next;
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```

_ _ _
### **ConcurrentHashMap 遍历操作**
通常可以使用ConcurrentHashMap的keySet()、values()、entrySet()方式对ConcurrentHashMap进行遍历，其内部原理大致相同，都是基于底层的map进行操作，对iterator进行的操作(如删除元素)都会反映到底层的map上面。不同于HashMap的是，ConcurrentHashMap并没有使用fail-fast机制，我的理解是因为ConcurrentHashMap本就是为了线程并发操作而设计的类，仅需支持弱一致性即可。      

这里仅以keySet()方法为例介绍其内部实现：  
```java
abstract class HashIterator {
    int nextSegmentIndex;
    int nextTableIndex;
    HashEntry<K,V>[] currentTable;
    HashEntry<K, V> nextEntry;
    HashEntry<K, V> lastReturned;

    HashIterator() {
        nextSegmentIndex = segments.length - 1;
        nextTableIndex = -1;
        advance();
    }

    /**
     * Set nextEntry to first node of next non-empty table
     * (in backwards order, to simplify checks).
     * 方法作用是确定下一个要遍历的Entry所在的位置  
     */
    final void advance() {
        for (;;) {
            if (nextTableIndex >= 0) {
                if ((nextEntry = entryAt(currentTable,
                                         nextTableIndex--)) != null)
                    break;
            }
            else if (nextSegmentIndex >= 0) {
                Segment<K,V> seg = segmentAt(segments, nextSegmentIndex--);
                if (seg != null && (currentTable = seg.table) != null)
                    nextTableIndex = currentTable.length - 1;
            }
            else
                break;
        }
    }

    // 方法中并没有实现fail-fast机制
    final HashEntry<K,V> nextEntry() {
        HashEntry<K,V> e = nextEntry;
        if (e == null)
            throw new NoSuchElementException();
        lastReturned = e; // cannot assign until after null check
        if ((nextEntry = e.next) == null)
            advance();
        return e;
    }

    public final boolean hasNext() { return nextEntry != null; }
    public final boolean hasMoreElements() { return nextEntry != null; }

    public final void remove() {
        if (lastReturned == null)
            throw new IllegalStateException();
        ConcurrentHashMap.this.remove(lastReturned.key);
        lastReturned = null;
    }
}

final class KeyIterator
    extends HashIterator
    implements Iterator<K>, Enumeration<K>
{
    public final K next()        { return super.nextEntry().key; }
    public final K nextElement() { return super.nextEntry().key; }
}

final class KeySet extends AbstractSet<K> {
    public Iterator<K> iterator() {
        return new KeyIterator();
    }
    public int size() { // 基于底层ConcurrentHashMap进行操作
        return ConcurrentHashMap.this.size();
    }
    public boolean isEmpty() {
        return ConcurrentHashMap.this.isEmpty();
    }
    public boolean contains(Object o) {
        return ConcurrentHashMap.this.containsKey(o);
    }
    public boolean remove(Object o) {
        return ConcurrentHashMap.this.remove(o) != null;
    }
    public void clear() {
        ConcurrentHashMap.this.clear();
    }
}

public Set<K> keySet() {
    Set<K> ks = keySet;
    return (ks != null) ? ks : (keySet = new KeySet());
}
```

顺便再回忆下迭代器设计模式的好处：  
1. 统一集合类的遍历方式
2. 可以同时对集合进行多次遍历


(完)

参考文章：  
[ConcurrentHashMap原理分析](https://my.oschina.net/hosee/blog/639352)   
[ConcurrentHashMap总结](https://my.oschina.net/hosee/blog/675884)  



