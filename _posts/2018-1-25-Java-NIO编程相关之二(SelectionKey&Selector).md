---
layout:     post
title:      Java-NIO编程相关之二(SelectionKey&Selector)
date:       2018-1-25
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Java基础
---

>本文主要记录了个人在学习NIO相关知识时一些思考过程，主要用于备忘，错误难免，敬请指出！
  
_ _ _
### **前言**

Java NIO API中包含三个核心组件：Channels、 Buffers、 Selector,其余组件有些只不过是为了这三个组件能更好的结合使用而设计的一些工具类，本文主要记录对于Selector相关知识的学习。    

_ _ _
### **Selector**  
**Select模型的意义并不在于消除阻塞，而是在于消除阻塞之后支持的多路复用机制，非阻塞读写很容易实现，但没有Select模型，如何实现类似Selector的1：N线程模型高效的处理非阻塞读写就会非常复杂。**   

**问题**：为什么select要求channel工作在非阻塞模式下呢？
Selector内部原理实际是在做一个对所注册的channel的轮询访问，不断的轮询(目前就这一个算法)，一旦轮询到一个channel有所注册的事情发生，比如数据来了，他就会站起来报告，交出一把钥匙，让我们通过这把钥匙来读取这个channel的内容。轮询多个Channel的过程中要求不能阻塞在对某个Channel的询问上，所以要求注册到Selector上的Channel都工作在非阻塞模式下？  

socketChannel可以注册connect,read,write;
serverSocketChannel只能注册对accept事件感兴趣；

非阻塞读写：内核中没有数据准备好，立即返回，不会等待数据准备好之后才返回。  
**问题**:什么情况下connect、Read、write在select过程才被认为是准备好呢？  

#### **Selector是什么、怎么用？**
传统的1:1线程模型对于每个到来的客户端都需要生成一个新的线程进行处理，生成多个线程及在线程之间切换的开销是不容忽视的，有些应用场景下使用1:1线程模型是不合适的，比如服务器需要同时支持大量的长期连接，比如说10000个连接以上，不过各个客户端并不会很频繁的发送太多的数据，想象下面这种场景：  

总公司的中心服务器要收集全国连锁便利店中各个收银机的交易信息，这种情况就很适合使用Java NIO通过Selector方式给我们提供的1:N线程模型，多路复用，采用少量的线程就可以完成任务，避免多余的开销。下面是一个线程使用Selector监听三个Channel事件的模型：   
<img src="/img/2017-11-29/overview-selectors.png" width="500" height="500" alt="selectors UML图" />
<center>图4：使用selector监听三个channel的事件</center>   

对于Selector与Channel，一个selector上可能注册多个channel，一个channel也可能注册到多个selector上，为了确定其一一对应关系，引入了SelectionKey。每个Channel中保存了一个SelectionKey的集合，保存了这个Channel都注册到了哪些selector上等相关信息，Selector中也有相应SelectionKey的集合保存了有哪些channel注册到当前Selector上等相关信息。通过这种方式使用SelectionKey可以解除Channel与Selector类之间的耦合，使得操作更加灵活，SelectionKey类具体源码如下：  
```java
public abstract class SelectionKey {

    public static final int OP_READ = 1 << 0;
    public static final int OP_WRITE = 1 << 2;
    public static final int OP_CONNECT = 1 << 3;
    public static final int OP_ACCEPT = 1 << 4;

    /**
     * Constructs an instance of this class.
     */
    protected SelectionKey() { }

    // -- Channel and selector operations --

    public abstract SelectableChannel channel();
    public abstract Selector selector();
    public abstract boolean isValid();
    public abstract void cancel();

    // -- Operation-set accessors --

    public abstract int interestOps();
    public abstract SelectionKey interestOps(int ops);
    public abstract int readyOps();

    // -- Operation bits and bit-testing convenience methods --    

    public final boolean isReadable() {
        return (readyOps() & OP_READ) != 0;
    }

    public final boolean isWritable() {
        return (readyOps() & OP_WRITE) != 0;
    }

    public final boolean isConnectable() {
        return (readyOps() & OP_CONNECT) != 0;
    }

    public final boolean isAcceptable() {
        return (readyOps() & OP_ACCEPT) != 0;
    }

    public final Object attach(Object ob) {
        return attachmentUpdater.getAndSet(this, ob);
    }

    public final Object attachment() {
        return attachment;
    }

}

public abstract class AbstractSelectionKey
    extends SelectionKey
{

    /**
     * Initializes a new instance of this class.
     */
    protected AbstractSelectionKey() { }

    private volatile boolean valid = true;

    public final boolean isValid() {
        return valid;
    }

    void invalidate() {                                 // package-private
        valid = false;
    }

    /**
     * Cancels this key.
     *
     * <p> If this key has not yet been cancelled then it is added to its
     * selector's cancelled-key set while synchronized on that set.  </p>
     */
    public final void cancel() {
        // Synchronizing "this" to prevent this key from getting canceled
        // multiple times by different threads, which might cause race
        // condition between selector's select() and channel's close().
        synchronized (this) {
            if (valid) {
                valid = false;
                ((AbstractSelector)selector()).cancel(this);
            }
        }
    }
}

package sun.nio.ch;

public class SelectionKeyImpl extends AbstractSelectionKey {
    final SelChImpl channel;
    public final SelectorImpl selector;
    private int index;
    private volatile int interestOps;
    private int readyOps;

    SelectionKeyImpl(SelChImpl ch, SelectorImpl sel) {
        this.channel = ch;
        this.selector = sel;
    }

    public SelectableChannel channel() {
        return (SelectableChannel)this.channel;
    }

    public Selector selector() {
        return this.selector;
    }

    public int interestOps() {
        this.ensureValid();
        return this.interestOps;
    }

    public SelectionKey interestOps(int ops) {
        this.ensureValid();
        return this.nioInterestOps(ops);
    }

    public int readyOps() {
        this.ensureValid();
        return this.readyOps;
    }

    public void nioReadyOps(int ops) {
        this.readyOps = ops;
    }

    public int nioReadyOps() {
        return this.readyOps;
    }
	
    ...
}

```
SelectionKey保存了channel与selector的对应关系，其实际实现类是SelectionKeyImpl，可以看到其构造方法是package可见，也就是说我们不能直接通过new来创建一个SelectionKey对象，通常情况下其获取方式为：  
```java
Selector selector = Selector.open();
ServerSocketChannel channel = ServerSocketChannel.open();
channel.configureBlocking(false);
channel.bind(new InetSocketAddress(SERVERIP, PORT));
//通过向一个selector中注册channel获取SelectionKey
SelectionKey key = channel.register(selector, SelectionKey.OP_ACCEPT);
```
在向selector中注册某个channel时通常还要指定监听的channel上的事件，比如上面代码中指定要监听ServerSocketChannel的accept事件，selector会保存在其上注册的所有channel对应的selectionKey集合，当监听事件发生时，我们可以通过selectionKey判断是哪个channel的哪个具体事件发生来做出相应处理，这也是SelectionKey的通常使用方式。
```java
while(true) {
    System.out.println("准备select");
    int readyChannels = selector.select();
    if(readyChannels == 0) continue;
    Set<SelectionKey> selectedKeys = selector.selectedKeys();
    Iterator<SelectionKey> keyIterator = selectedKeys.iterator();

    while(keyIterator.hasNext()) {

        SelectionKey key = keyIterator.next();

        if(key.isAcceptable()) {
            System.out.println("acceptable!");
            SocketChannel socketChannel = ((ServerSocketChannel) key.channel()).accept();
            new Thread(new SocketChannelThread(socketChannel)).start();
        } else if (key.isConnectable()) {
            // a connection was established with a remote server.

        } else if (key.isReadable()) {
            // a channel is ready for reading

        } else if (key.isWritable()) {
            // a channel is ready for writing
        }

        keyIterator.remove();
    }
}
```
对于select方式的底层实现Java实际上是通过调用native方法进而调用底层系统调用来实现的，比如linux上调用epoll系统调用。  