---
layout:     post
title:      深入学习java并发编程之ConcurrentLinkedQueue-JDK1.8
date:       2018-7-29
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 并发
    - JDK源码阅读
---

>本篇随笔主要记录了我阅读ConcurrentLinkedQueue(Jdk1.8)源码期间的对于ConcurrentLinkedQueue的一些实现上的个人目前的理解，膜拜Doug Lea大神的**无锁并发安全队列**的实现，肯定有很多地方理解的不到位，有待继续深入。  
  
_ _ _
### **前言**
通常使用CAS实现并发安全的方式是CAS操作更新一个volatile状态变量，比如下面的并发安全的堆栈实现：  
```java
public class ConcurrentStack<E> {
    AtomicReference<Node<E>> head = new AtomicReference<Node<E>>();
    public void push(E item) {
        Node<E> newHead = new Node<E>(item);
        Node<E> oldHead;
        do {
            oldHead = head.get();
            newHead.next = oldHead;
        } while (!head.compareAndSet(oldHead, newHead));
    }
    public E pop() {
        Node<E> oldHead;
        Node<E> newHead;
        do {
            oldHead = head.get();
            if (oldHead == null) 
                return null;
            newHead = oldHead.next;
        } while (!head.compareAndSet(oldHead,newHead));
        return oldHead.item;
    }
    static class Node<E> {
        final E item;
        Node<E> next;
        public Node(E item) { this.item = item; }
    }
}
```
上述代码中push() 方法观察当前最顶的节点，构建一个新节点放在堆栈上，然后，如果最顶端的节点在初始观察之后没有变化，那么就安装新节点。如果 CAS 失败，意味着另一个线程已经修改了堆栈，那么过程就会重新开始。  

堆栈只需要使用CAS更新一个head指针即可，但对于队列来说，需要更新头尾两个指针，但CAS不支持对两个以上的指针的原子性条件更新，所以，要构建一个非阻塞的链表就需要找到一种方式，可以用 CAS 更新多个指针，同时不会让数据结构处于不一致的状态。  

下面就来看看ConcurrentLinkedQueue是如何解决这个问题的。  

_ _ _
### **基础结构**
#### **链表节点存储**
典型的链表数据结构设计，内部提供了CAS更新存储的值item和下一个节点引用next的方法。  
```java
private static class Node<E> {
    volatile E item;
    volatile Node<E> next;

    /**
     * Constructs a new node.  Uses relaxed write because item can
     * only be seen after publication via casNext.
     */
    Node(E item) {
        UNSAFE.putObject(this, itemOffset, item);
    }

    boolean casItem(E cmp, E val) {
        return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
    }

    void lazySetNext(Node<E> val) {
        UNSAFE.putOrderedObject(this, nextOffset, val);
    }

    boolean casNext(Node<E> cmp, Node<E> val) {
        return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
    }

    // Unsafe mechanics

    private static final sun.misc.Unsafe UNSAFE;
    private static final long itemOffset;
    private static final long nextOffset;

    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = Node.class;
            itemOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("item"));
            nextOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("next"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```
#### **类中成员变量**
```java
private transient volatile Node<E> head;// 链表头结点

private transient volatile Node<E> tail;// 链表尾节点
```

#### **构造方法**
```java
/**
 * Creates a {@code ConcurrentLinkedQueue} that is initially empty.
 */
public ConcurrentLinkedQueue() {
    head = tail = new Node<E>(null); // 初始指向一个Dummy节点
}

/**
 * Creates a {@code ConcurrentLinkedQueue}
 * initially containing the elements of the given collection,
 * added in traversal order of the collection's iterator.
 *
 * @param c the collection of elements to initially contain
 * @throws NullPointerException if the specified collection or any
 *         of its elements are null
 */
public ConcurrentLinkedQueue(Collection<? extends E> c) {
    Node<E> h = null, t = null;
    for (E e : c) {
        checkNotNull(e);
        Node<E> newNode = new Node<E>(e);
        if (h == null)
            h = t = newNode;
        else {
            t.lazySetNext(newNode);
            t = newNode;
        }
    }
    if (h == null)
        h = t = new Node<E>(null);
    head = h;
    tail = t;
}
```

_ _ _
### **插入数据方法offer()**
```java
/**
 * Inserts the specified element at the tail of this queue.
 * As the queue is unbounded, this method will never return {@code false}.
 *
 * @return {@code true} (as specified by {@link Queue#offer})
 * @throws NullPointerException if the specified element is null
 */
public boolean offer(E e) {
    checkNotNull(e);
    final Node<E> newNode = new Node<E>(e);

    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {
            // p确实是最后一个节点
            if (p.casNext(null, newNode)) {                
                if (p != t) // 标记1
                    // 失败就失败了，总会有线程来更新tail节点，这么做是为了尽量减少对tail节点的CAS更新，提高效率
                    // 并不是每次节点入队后都更新tail节点，而是当tail节点和尾节点的距离大于1时才更新tail节点
                    casTail(t, newNode);  
                return true;
            }
            // Lost CAS race to another thread; re-read next
        }
        else if (p == q) // 标记2
            // We have fallen off list.  If tail is unchanged, it
            // will also be off-list, in which case we need to
            // jump to head, from which all live nodes are always
            // reachable.  Else the new tail is a better bet.
            p = (t != (t = tail)) ? t : head;
        else
            // Check for tail updates after two hops. 
            p = (p != t && t != (t = tail)) ? t : q; // 标记3
    }
}
```
先来看下标记1中的情况何时出现，首先线程1调用了offer方法，单线程的情况下完成了一次插入，插入后链表情况如下：  
<img src="/img/2018-7-29/step1.png" width="500" height="500" alt="线程1插入一个数据之后" />
<center>图1：线程1插入一个数据之后</center>  

可见插入后tail节点并没有指向新插入的节点，之后线程2插入另一个数据，插入前链表情况如下：  
<img src="/img/2018-7-29/step2.png" width="500" height="500" alt="线程2插入一个数据之前" />
<center>图2：线程2插入一个数据之前</center>  

可见 q = p.next 并不为空，执行到标记3所在的代码处，之后将q赋值给p，准备开始下一次循环，执行后情况如下：  
<img src="/img/2018-7-29/step3.png" width="500" height="500" alt="线程2插入一个数据" />
<center>图3：线程2插入一个数据</center>  

在下一次循环中由于 q = p.next 为空，执行if内代码插入新的节点，并且由于 p != t 成立，所以更新尾节点为新插入的节点：  
<img src="/img/2018-7-29/step4.png" width="500" height="500" alt="线程2插入一个数据后" />
<center>图4：线程2插入一个数据后</center>  

由上面插入过程中可以看到，由于代码中标记1处判断的存在，导致并不是每次节点入队后都更新tail节点，而是当tail节点和尾节点的距离大于1时才更新tail节点，这样可以提升入队效率。  

标记3处的代码会检查p是否在原来的基础上做过移动，以及现在的尾节点是否等于最新的尾节点，根据这两个条件判断是将p移动到它的next节点还是直接将其移动到最新的尾节点(如果尾节点被别的线程更新过了，直接将其移动到尾节点比一次次的next移动效率要高)。  

offer方法代码中标记1处与标记3处的情况都已经简单分析过了，标记2处代码执行的逻辑是如果尾节点更新过了，将p指向尾节点，否则将p指向头结点，那么标记2处的情况什么时候会出现呢？每次循环之前明明已执行过 q = p.next, 怎么会出现 p == q 的情况呢？  

接下来在对poll方法分析中找到答案。          

_ _ _
### **数据出队方法poll()**
```java
public E poll() {
    restartFromHead:
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {
            E item = p.item;

            if (item != null && p.casItem(item, null)) {
                // Successful CAS is the linearization point
                // for item to be removed from this queue.
                if (p != h) // hop two nodes at a time
                    updateHead(h, ((q = p.next) != null) ? q : p);
                return item;
            }
            else if ((q = p.next) == null) {
                updateHead(h, p);
                return null;
            }
            else if (p == q) // 标记1
                continue restartFromHead;
            else
                p = q;
        }
    }
}

final void updateHead(Node<E> h, Node<E> p) {
    if (h != p && casHead(h, p))
        h.lazySetNext(h); // 就是这里将之前出队的head节点的引用指向了自己
}
```
poll执行之前假设初始队列如下：
<img src="/img/2018-7-29/step5.png" width="500" height="500" alt="初始队列" />
<center>图5：初始队列</center>  

之后线程1单线程情况下进行了一次poll操作，结果如下：
<img src="/img/2018-7-29/step6.png" width="500" height="500" alt="线程1 poll一次之后" />
<center>图6：线程1 poll一次之后</center>  

之后线程2进行了另一次poll操作，操作后结果如下：  
<img src="/img/2018-7-29/step7.png" width="500" height="500" alt="线程2 poll一次之后" />
<center>图7：线程2 poll一次之后</center>  

可见被删除的之前的head节点的next指向了本身。  

图7也回答了我们offer方法中标记2处代码出现的情况，即offer方法执行时获取了tail指针，等待向后执行，之后线程切换，等线程再切换回来的时候发现由于这次offer线程的插入速度太慢，而之前指向的tail节点已经被删除，所以 虽然 q = p.next,但实际上 p和q 指向的还是同一个节点。上面poll代码中标记1处的情况也与此类似，不再赘述。  

_ _ _
### **size()方法与contains()方法**
size方法与contains方法都需要对队列进行遍历，接下来就来看看其内部遍历是如何进行的：  
```java
public int size() {
    int count = 0;
    for (Node<E> p = first(); p != null; p = succ(p))
        if (p.item != null)
            // Collection.size() spec says to max out
            if (++count == Integer.MAX_VALUE)
                break;
    return count;
}

public boolean contains(Object o) {
    if (o == null) return false;
    for (Node<E> p = first(); p != null; p = succ(p)) {
        E item = p.item;
        if (item != null && o.equals(item))
            return true;
    }
    return false;
}

Node<E> first() { // 返回头结点，与peek方法原理相同，不同之处仅仅在于返回的是Node节点而不是其中的元素
    restartFromHead:
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {
            boolean hasItem = (p.item != null);
            if (hasItem || (q = p.next) == null) {
                updateHead(h, p);
                return hasItem ? p : null;
            }
            else if (p == q)
                continue restartFromHead;
            else
                p = q;
        }
    }
}

final Node<E> succ(Node<E> p) { // 返回节点p的下一个节点
    Node<E> next = p.next;
    return (p == next) ? head : next; // 如果p.next == p,证明节点p已经过时，返回新的头结点即可
}
```

_ _ _
### **总结**
ConcurrentLinkedQueue是并发大师Doug Lea 根据Michael-Scott提出的非阻塞链接队列算法的基础上修改而来，它是一个基于链表的无界线程安全队列，它采用先入先出的规则对节点进行排序，当我们添加一个节点的时候，它会添加到队列的尾部；当我们获取一个元素的时，它会返回队列头部的元素。它通过使用head和tail引用延迟更改的方式，减少CAS操作，在满足线程安全的前提下，提高了队列的操作效率。  

(完)  

参考文章：  
[非阻塞算法简介](https://www.ibm.com/developerworks/cn/java/j-jtp04186/)    
[ConcurrentLinkedQueue源码分析](https://www.jianshu.com/p/7816c1361439)   
