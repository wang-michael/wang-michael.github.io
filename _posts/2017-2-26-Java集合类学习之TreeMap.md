---
layout:     post
title:      Java集合类学习之TreeMap
date:       2017-2-26
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Java基础 
    - JDK源码阅读
---

>本篇随笔主要记录了我阅读TreeMap(JDK1.8)源码期间的对于TreeMap的一些实现上的个人理解，用于个人备忘，有不对的地方，请指出。  
  
_ _ _
### **前言**
TreeMap将其内部所有节点组织成了一颗红黑树，从而使得检索、插入、删除操作平均和最差时间都是O(lgn)。如果程序中对存储的数据集合的操作经常需要得到一些有序的结果，比如查找大于指定元素的所有节点，那么应该使用TreeMap。理解TreeMap的核心在于理解其内部数据结构的实现，其提供的一些操作相关的API仅是在红黑树数据结构基础之上进行的一些增删改查的操作而已，所以本篇博客重点记录下TreeMap内部实现红黑树的过程。关于红黑树数据结构的基本操作过程，可以参考我的另一篇博客：[数据结构之红黑树](https://wang-michael.github.io/2017/02/15/%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E4%B9%8B%E7%BA%A2%E9%BB%91%E6%A0%91/)   

_ _ _
### **TreeMap内部红黑树的实现**
TreeMap中用于描述红黑树的成员变量如下：  
```java
// 红黑树树根
private transient Entry<K,V> root;

// Red-black mechanics，红黑机制
private static final boolean RED   = false;
private static final boolean BLACK = true;

static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left;
    Entry<K,V> right;
    Entry<K,V> parent;
    boolean color = BLACK;
}
```  
红黑树节点对应的类为Entry，其包含了6个部分内容：key(键)、value(值)、left(左孩子)、right(右孩子)、parent(父节点)、color(颜色)
Entry节点根据key进行排序，Entry节点包含的内容为value。  

#### **左旋**
```java
/** 
 * From CLR
 * 目的是把p的右孩子转到p的原来的位置上 
*/
private void rotateLeft(Entry<K,V> p) {
    if (p != null) {
        Entry<K,V> r = p.right;
        p.right = r.left;
        if (r.left != null)
            r.left.parent = p;
        r.parent = p.parent;
        if (p.parent == null)
            root = r;
        else if (p.parent.left == p)
            p.parent.left = r;
        else
            p.parent.right = r;
        r.left = p;
        p.parent = r;
    }
}
```
#### **右旋**
```java
/** 
 * From CLR 
 * 目的是把p的左孩子转到p的原来的位置上 
 */
private void rotateRight(Entry<K,V> p) {
    if (p != null) {
        Entry<K,V> l = p.left;
        p.left = l.right;
        if (l.right != null) l.right.parent = p;
        l.parent = p.parent;
        if (p.parent == null)
            root = l;
        else if (p.parent.right == p)
            p.parent.right = l;
        else p.parent.left = l;
        l.right = p;
        p.parent = l;
    }
}
```
#### **插入操作**
```java
public V put(K key, V value) {
    Entry<K,V> t = root;
    if (t == null) {
        compare(key, key); // type (and possibly null) check

        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
    int cmp;
    Entry<K,V> parent;
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator;
    if (cpr != null) {
        // 指定了comparator之后按照二叉搜索树的算法进行搜索
        do {
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value); // 如果要插入的key值已经存在，直接更新其value即可
        } while (t != null);
    }
    else {
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")  // 默认key一定是Comparable的子类
            Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    // 挂接新插入的节点
    Entry<K,V> e = new Entry<>(key, value, parent);
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;
    // 进行插入之后的颜色修正操作
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}
```

#### **插入修正操作**
```java
/** From CLR */
private void fixAfterInsertion(Entry<K,V> x) {
    x.color = RED; // 默认新插入的节点颜色为红色
    // 当父节点的颜色为红色时存在红红冲突，需要进行颜色修正操作
    while (x != null && x != root && x.parent.color == RED) {
        // 如果父节点是祖父节点的左孩子
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            Entry<K,V> y = rightOf(parentOf(parentOf(x))); // 找到x的叔父节点
            if (colorOf(y) == RED) { // x的叔父节点为红色
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {
                if (x == rightOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateLeft(x);
                }
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateRight(parentOf(parentOf(x)));
            }
        } else { // 父节点是祖父节点的右孩子
            Entry<K,V> y = leftOf(parentOf(parentOf(x))); // 找到x的叔父节点
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {
                if (x == leftOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateRight(x);
                }
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateLeft(parentOf(parentOf(x)));
            }
        }
    }
    root.color = BLACK;
}
```
#### **删除操作**
```java
/**
 * Delete node p, and then rebalance the tree.
 */
private void deleteEntry(Entry<K,V> p) {
    modCount++;
    size--;

    // If strictly internal, copy successor's element to p and then make p
    // point to successor.如果p左右孩子均不为空，找到其替代节点与其替换key和value
    if (p.left != null && p.right != null) {
        Entry<K,V> s = successor(p);
        p.key = s.key;
        p.value = s.value;
        p = s;
    } // p has 2 children

    // Start fixup at replacement node, if it exists.
    Entry<K,V> replacement = (p.left != null ? p.left : p.right);

    if (replacement != null) { // 左孩子或者右孩子其中之一不为空
        // Link replacement to parent 将替代节点挂接到被删除节点原来位置上
        replacement.parent = p.parent;
        if (p.parent == null)
            root = replacement;
        else if (p == p.parent.left)
            p.parent.left  = replacement;
        else
            p.parent.right = replacement;

        // Null out links so they are OK to use by fixAfterDeletion.
        p.left = p.right = p.parent = null;

        // Fix replacement
        if (p.color == BLACK) // 被删节点为黑色，进行删除后修正
            fixAfterDeletion(replacement);
    } else if (p.parent == null) { // return if we are the only node.
        root = null;
    } else { //  No children. Use self as phantom replacement and unlink.
        if (p.color == BLACK)  // 被删节点没有孩子，且为黑色，进行删除后修正
            fixAfterDeletion(p);

        if (p.parent != null) {
            if (p == p.parent.left)
                p.parent.left = null;
            else if (p == p.parent.right)
                p.parent.right = null;
            p.parent = null;
        }
    }
}
// 找到待删除节点的替代节点
static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
    if (t == null)
        return null;
    else if (t.right != null) { // 寻找t的直接中序后继节点(右最小)
        Entry<K,V> p = t.right;
        while (p.left != null)
            p = p.left;
        return p;
    } else { // 这种情况何时出现？  
        Entry<K,V> p = t.parent;
        Entry<K,V> ch = t;
        while (p != null && ch == p.right) {
            ch = p;
            p = p.parent;
        }
        return p;
    }
}
```

#### **删除后修正**
```java
/** From CLR */
private void fixAfterDeletion(Entry<K,V> x) {
    while (x != root && colorOf(x) == BLACK) { // 检测是否为双黑情况
        if (x == leftOf(parentOf(x))) { // x是左孩子，右孩子则对称
            Entry<K,V> sib = rightOf(parentOf(x)); // 找到x的兄弟节点sib

            if (colorOf(sib) == RED) { // 兄弟节点为红色
                setColor(sib, BLACK);
                setColor(parentOf(x), RED);
                rotateLeft(parentOf(x));
                sib = rightOf(parentOf(x));
            }

            if (colorOf(leftOf(sib))  == BLACK &&
                colorOf(rightOf(sib)) == BLACK) { // 兄弟节点为黑色且有两个黑色子节点
                setColor(sib, RED);
                x = parentOf(x);
            } else { // 兄弟节点为黑色且子节点有红色，兄弟节点两子节点共三种情况：两孩子全红，左红右黑，左黑右红
                if (colorOf(rightOf(sib)) == BLACK) { // 兄弟节点右孩子为黑色，左孩子为红色，需要进行两次旋转
                    setColor(leftOf(sib), BLACK);
                    setColor(sib, RED);
                    rotateRight(sib);
                    sib = rightOf(parentOf(x));
                }
                setColor(sib, colorOf(parentOf(x))); 
                setColor(parentOf(x), BLACK);
                setColor(rightOf(sib), BLACK);
                rotateLeft(parentOf(x));
                x = root;
            }
        } else { // symmetric
            Entry<K,V> sib = leftOf(parentOf(x));

            if (colorOf(sib) == RED) {
                setColor(sib, BLACK);
                setColor(parentOf(x), RED);
                rotateRight(parentOf(x));
                sib = leftOf(parentOf(x));
            }

            if (colorOf(rightOf(sib)) == BLACK &&
                colorOf(leftOf(sib)) == BLACK) {
                setColor(sib, RED);
                x = parentOf(x);
            } else {
                if (colorOf(leftOf(sib)) == BLACK) {
                    setColor(rightOf(sib), BLACK);
                    setColor(sib, RED);
                    rotateLeft(sib);
                    sib = leftOf(parentOf(x));
                }
                setColor(sib, colorOf(parentOf(x)));
                setColor(parentOf(x), BLACK);
                setColor(leftOf(sib), BLACK);
                rotateRight(parentOf(x));
                x = root;
            }
        }
    }

    setColor(x, BLACK);
}
```

_ _ _
### **TreeMap基于红黑树实现的对外API**
这里只以ceilingKey方法为例进行介绍，其余方法不再赘述，都是基于红黑树数据结构进行的操作。  

ceilingKey(K key)的作用是“返回大于/等于key的最小的键值对所对应的KEY，没有的话返回null”，它的代码如下：
```java
public K ceilingKey(K key) {
    return keyOrNull(getCeilingEntry(key));
}
static <K,V> K keyOrNull(TreeMap.Entry<K,V> e) {
    return (e == null) ? null : e.key;
}

final Entry<K,V> getCeilingEntry(K key) {
    Entry<K,V> p = root;
    while (p != null) {
        int cmp = compare(key, p.key);
        if (cmp < 0) { // key比当前节点的key值小
            if (p.left != null)
                p = p.left;
            else
                return p;
        } else if (cmp > 0) { // key比当前节点的key值大
            if (p.right != null) {
                p = p.right;
            } else {
                Entry<K,V> parent = p.parent;
                Entry<K,V> ch = p;
                // 回退到之前找到的大于等于给定key值的节点，不存在则返回null (parent为null)
                while (parent != null && ch == parent.right) { 
                    ch = parent;
                    parent = parent.parent;
                }
                return parent;
            }
        } else
            return p;
    }
    return null;
}
final int compare(Object k1, Object k2) {
    return comparator==null ? ((Comparable<? super K>)k1).compareTo((K)k2)
        : comparator.compare((K)k1, (K)k2);
}
```

(完)
