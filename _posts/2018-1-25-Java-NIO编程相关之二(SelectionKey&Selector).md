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

#### **为什么要使用Selector模式？**  
传统的1:1线程模型对于每个到来的客户端都需要生成一个新的线程进行处理，生成多个线程及在线程之间切换的开销是不容忽视的，有些应用场景下使用1:1线程模型是不合适的，比如服务器需要同时支持大量的长期连接，比如说10000个连接以上，不过各个客户端并不会很频繁的发送太多的数据，想象下面这种场景：  

总公司的中心服务器要收集全国连锁便利店中各个收银机的交易信息，这种情况就很适合使用Java NIO通过Selector方式给我们提供的1:N线程模型，多路复用，采用少量的线程就可以完成任务，避免多余的开销。下面是一个线程使用Selector监听三个Channel事件的模型：   
<img src="/img/2018-1-25/overview-selectors.png" width="500" height="500" alt="使用selector监听三个channel的事件" />
<center>图1：使用selector监听三个channel的事件</center>   

#### **SelecionKey是什么、怎么用？**  
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
    //interestOps()方法返回值为Channel注册到Selector上时注册的感兴趣的操作  
    //比如SocketChannel注册到Selector上时注册对Read操作感兴趣，则其对应的SelectionKey.interestOps()方法返回值为1    
    public abstract int interestOps();  
    //更改当前SelectionKey的感兴趣的操作(在将Channel注册到Selector之后其interestOps还可以动态更改吗？)    
    public abstract SelectionKey interestOps(int ops);  
    //返回当前Channel上的就绪操作的集合，在进行select操作时会由Selector设置？  
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
     * 这个方法被调用有三种可能情况：  
     *     1、主动调用SelectionKey.cancel()方法
     *     2、channel关闭时调用此方法cancel与其关联的SelectionKey
     *     3、Selector关闭时关闭所有注册到其上的SelectionKey
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
#### **创建Selector**  
一般通过Selector.open()方法创建Selector类的实例对象，open方法所做的工作就是寻找到相应的SelectorProvider，创建平台相关的Selector实现，Windows下默认实现类为WindowsSelectorImpl，其UML图如下：  
<img src="/img/2018-1-25/selectorImpl.jpg" width="300" height="300" alt="selectors UML图" />
<center>图2：JDK1.8 selector UML图</center>  
下面介绍的内容都是基于WindowsSelectorImpl类实现的，首先看下它的创建过程：  
```java
public abstract class Selector implements Closeable {

    protected Selector() { }

    ...
}
public abstract class AbstractSelector extends Selector {

    private final Set<SelectionKey> cancelledKeys = new HashSet<SelectionKey>();

    protected AbstractSelector(SelectorProvider provider) {
        this.provider = provider;
    }

    protected final Set<SelectionKey> cancelledKeys() {
        return cancelledKeys;
    }
    ...
}

abstract class SelectorImpl extends AbstractSelector {

    // 保存有感兴趣的操作已经就绪的SelectionKey的集合
    protected Set<SelectionKey> selectedKeys;
    // 保存注册到Selector上的所有SelectionKey的集合
    protected HashSet<SelectionKey> keys;
    // 对于keys集合的包装，使得集合不能被更改
    private Set<SelectionKey> publicKeys;             // Immutable
    // 对于selectedKeys集合的包装，使得此集合上只能进行删除操作
    private Set<SelectionKey> publicSelectedKeys;     // Removal allowed, but not addition

    protected SelectorImpl(SelectorProvider sp) {
        super(sp);
        keys = new HashSet<SelectionKey>();
        selectedKeys = new HashSet<SelectionKey>();
        if (Util.atBugLevel("1.4")) {
            publicKeys = keys;
            publicSelectedKeys = selectedKeys;
        } else {
            publicKeys = Collections.unmodifiableSet(keys);
            publicSelectedKeys = Util.ungrowableSet(selectedKeys);
        }
    }

    public Set<SelectionKey> keys() {
        if (!isOpen() && !Util.atBugLevel("1.4"))
            throw new ClosedSelectorException();
        return publicKeys;
    }

    public Set<SelectionKey> selectedKeys() {
        if (!isOpen() && !Util.atBugLevel("1.4"))
            throw new ClosedSelectorException();
        return publicSelectedKeys;
    }
}

final class WindowsSelectorImpl extends SelectorImpl {
    // 设置select要轮询的Channel数组的初始容量
    private final int INIT_CAP = 8;
    // 由于windows下select系统调用支持的最大fd数量为1024，所以设置一个线程最多可管理的文件描述符数量为1024个
    // Should be INIT_CAP times a power of 2
    private final static int MAX_SELECTABLE_FDS = 1024;

    // 保存包含wakeUpSocketChannel在内的注册到当前Selector上的Channel数组
    private SelectionKeyImpl[] channelArray = new SelectionKeyImpl[INIT_CAP];

    /*
     持有一块堆外的内存空间，分配堆外内存空间的目的可能是为了不受GC影响；保存底层select系统调用所需要的文件描述符数据结构；
       很重要，实际select系统调用就是基于保存在其中的Channel对应的fd上进行的
    */
    private PollArrayWrapper pollWrapper;

    // 保存注册到当前Selector上的所有Channel
    private int totalChannels = 1;

    /* 保存当前帮助进行select操作的帮助线程数量
      为什么需要帮助线程呢？因为由于底层select系统调用限制，一个线程所能管理的fd数量最大为1024个，当我们需要处理更多的文件描述
      符的时候，就需要使用帮助线程调用新的select系统调用帮我们管理超出部分的文件描述符。
    */
    private int threadsCount = 0;

    // 保存帮助线程的集合
    private final List<SelectThread> threads = new ArrayList<SelectThread>();

    // 在创建WindowsSelectorImpl对象时会创建一个管道，就是通过这个管道实现的wakeUp操作
    private final Pipe wakeupPipe;

    // wakeupPipe管道两端对应的文件描述符
    private final int wakeupSourceFd, wakeupSinkFd;

    // close操作进行时用到的lock
    private Object closeLock = new Object();

    // 使用SubSelector类poll方法进行实际select操作，这是主线程使用的subSelector对象
    private final SubSelector subSelector = new SubSelector();

    private long timeout; //设置select调用的超时时间

    // Lock for interrupt triggering and clearing
    private final Object interruptLock = new Object();
    private volatile boolean interruptTriggered = false;

    WindowsSelectorImpl(SelectorProvider sp) throws IOException {
        super(sp);
        pollWrapper = new PollArrayWrapper(INIT_CAP);
        wakeupPipe = Pipe.open();
        wakeupSourceFd = ((SelChImpl)wakeupPipe.source()).getFDVal();

        // Disable the Nagle algorithm so that the wakeup is more immediate
        SinkChannelImpl sink = (SinkChannelImpl)wakeupPipe.sink();
        (sink.sc).socket().setTcpNoDelay(true);
        wakeupSinkFd = ((SelChImpl)sink).getFDVal();

        pollWrapper.addWakeupSocket(wakeupSourceFd, 0);
    }
    
    ...
}

/*分配一块堆外内存空间，保存Selecor类要用到的fd的集合，select要使用的fd数据结构如下：
   typedef struct {
     jint fd;
     jshort events;
   } pollfd;
   其实只需要6个字节，但分配空间时为每个fd分配了8个字节
*/
class PollArrayWrapper {
    
    //通过这个对象分配堆外内存空间
    private AllocatedNativeObject pollArray;
    //保存分配好的堆外内存空间的起始地址
    long pollArrayAddress;
    private static final short FD_OFFSET = 0;
    private static final short EVENT_OFFSET = 4;
    //为每个fd分配8个字节堆外内存空间
    static short SIZE_POLLFD = 8;
    private int size;

    //通过指定PollArrayWrapper中需要保存的fd个数来分配堆外内存空间
    PollArrayWrapper(int newSize) {
        int allocationSize = newSize * SIZE_POLLFD;
        this.pollArray = new AllocatedNativeObject(allocationSize, true);
        this.pollArrayAddress = this.pollArray.address();
        this.size = newSize;
    }

    //向PollArrayWrapper中添加新的文件描述符
    void addEntry(int index, SelectionKeyImpl ski) {
        this.putDescriptor(index, ski.channel.getFDVal());
    }

    //替换target中指定位置的fd
    void replaceEntry(PollArrayWrapper source, int sindex, PollArrayWrapper target, int tindex) {
        target.putDescriptor(tindex, source.getDescriptor(sindex));
        target.putEventOps(tindex, source.getEventOps(sindex));
    }

    //根据指定的位置在pollArray中放置文件描述符
    void putDescriptor(int i, int fd) {
        this.pollArray.putInt(SIZE_POLLFD * i + 0, fd);
    }

    ///根据指定的位置在pollArray中设置对应fd感兴趣的事件
    void putEventOps(int i, int event) {
        this.pollArray.putShort(SIZE_POLLFD * i + 4, (short)event);
    }

    void grow(int newSize) {
        PollArrayWrapper temp = new PollArrayWrapper(newSize);

        for(int i = 0; i < this.size; ++i) {
            this.replaceEntry(this, i, temp, i);
        }

        this.pollArray.free();
        this.pollArray = temp.pollArray;
        this.size = temp.size;
        this.pollArrayAddress = this.pollArray.address();
    }

    void free() {
        this.pollArray.free();
    }    

    int getEventOps(int i) {
        return this.pollArray.getShort(SIZE_POLLFD * i + 4);
    }

    int getDescriptor(int i) {
        return this.pollArray.getInt(SIZE_POLLFD * i + 0);
    }

    //把wakeUpSocket也交给pollArrayWrapper来管理
    void addWakeupSocket(int fdVal, int index) {
        this.putDescriptor(index, fdVal);
        this.putEventOps(index, Net.POLLIN);
    }
}
```
WindowsSelectorImpl构造函数中主要做了两件事:  
1. 创建PollArrayWrapper对象并设置初始容量为8，意味着在其中最多可以保存8个Channel对应的fd及其感兴趣的事件。  
2. 创建一个Pipe，并将pipe的SourceChannel对应的fd也交由PollArrayWrapper管理，这是为了实现Selector上的wakeUp方法，之后会具体介 绍。    

#### **执行select**  
select操作是Selector中的重头戏，Selector类与select相关的对外供程序员调用的API有以下几种：  
```java
    //阻塞调用select，直到注册的Channel上有感兴趣的事件就绪时才返回
    abstract int select();
    //指定select操作超时时间
    abstract int select(long timeout):
    //不管是否有事件就绪，调用后立即返回
    abstract int selectNow():
    //获取在Selector上注册过的所有Channel对应的SelectionKey的集合
    abstract Set<SelectionKey> keys():
    //select操作过后更新的有事件就绪的SelectionKey的集合
    abstract Set<SelectionKey> selectedKeys():
```
为了更好的使用Selector提供给我们的API，最好理解其内部实现方式；下面就来看下select()方法的内部实现，具体分析见注释：  
```java
abstract class SelectorImpl extends AbstractSelector {

    //实际调用的是WindowsSelectorImpl中的doSelect方法
    protected abstract int doSelect(long timeout) throws IOException;

    private int lockAndDoSelect(long timeout) throws IOException {
        //对于一个Selector对象，同一时刻只允许一个线程进行真正的select操作
        synchronized (this) {
            if (!isOpen())
                throw new ClosedSelectorException();
            synchronized (publicKeys) {
                synchronized (publicSelectedKeys) {
                    return doSelect(timeout);
                }
            }
        }
    }
    ...
}

final class WindowsSelectorImpl extends SelectorImpl {

    protected int doSelect(long timeout) throws IOException {
        if (channelArray == null)
            throw new ClosedSelectorException();
        this.timeout = timeout; // set selector timeout
        //进行select操作之前先删除无效的SelectionKey
        processDeregisterQueue();
        if (interruptTriggered) {
            resetWakeupSocket();
            return 0;
        }
        // 根据fd数量计算需要的帮助线程的数量，计算后如果需要帮助线程的话，帮助线程就会启动并且阻塞在
        // 对startLock对象的wait方法调用上；
        adjustThreadsCount();
        finishLock.reset(); // 重置当前待结束的帮助线程的数量为threads.size()
        //由当前线程，也就是主线程，唤醒等待在startLock上的所有帮助线程，开启下一轮的poll操作。
        startLock.startThreads();
        // 主线程负责检测0-1023上fd的就绪操作
        try {
            //使用begin()-end()方法的目的是为了使得主线程interrupt()方法被调用后主线程能从select阻塞中返回。
            begin();
            try {
                subSelector.poll();
            } catch (IOException e) {
                finishLock.setException(e); // Save this exception
            }
            // 主线程select返回之后如果还有帮助线程没有从select中返回，主线程会阻塞等待这些帮助线程结束。  
            if (threads.size() > 0)
                finishLock.waitForHelperThreads();
          } finally {
              end();
          }        
        finishLock.checkForException();
        //select操作之后再次删除无效的SelectionKey
        processDeregisterQueue();
        //更新selectedKey集合，设置更新的selectKey的数量
        int updated = updateSelectedKeys();
        // 重置wakeUpSocket，准备进行下一轮select操作
        resetWakeupSocket();
        return updated;
    }

    // 在有新的Channel注册和解除注册之后，所需要的帮助线程的数量可能需要动态调整
    private void adjustThreadsCount() {
        //threadsCount初始为0，每当fd数量增加或减少1024个，threadsCount就加减1
        if (threadsCount > threads.size()) {
            // More threads needed. Start more threads.
            for (int i = threads.size(); i < threadsCount; i++) {
                //创建新的帮助线程进行select操作
                SelectThread newThread = new SelectThread(i);
                threads.add(newThread);
                //将新创建的帮助线程设置为主线程的守护线程
                newThread.setDaemon(true);
                newThread.start();
            }
        } else if (threadsCount < threads.size()) {
            // 删除无用的帮助线程
            for (int i = threads.size() - 1 ; i >= threadsCount; i--)
                threads.remove(i).makeZombie();
        }
    }

    // 代表一个用来进行select操作的帮助线程
    private final class SelectThread extends Thread {
        private final int index; // 线程编号，用来区分管理的是哪些fd，比如0代表管理的是1024-2047
        final SubSelector subSelector;用来进行select操作的SubSelector对象
        private long lastRun = 0; // last run number
        //用来区分当前帮助线程是否为无用线程的标志
        private volatile boolean zombie;
        // Creates a new thread
        private SelectThread(int i) {
            this.index = i;
            this.subSelector = new SubSelector(i);
            //确保当前线程启动后等待下一轮的select轮询操作
            this.lastRun = startLock.runsCounter;
        }
        void makeZombie() {
            zombie = true;
        }
        boolean isZombie() {
            return zombie;
        }
        public void run() {
            //while(true)循环的目的是为了使得在主线程不结束的情况下，帮助线程也不会退出
            //防止每次调用同一个Selector对象的select方法时创建帮助线程的开销
            while (true) { 
                // 可能阻塞，等待主线程中startLock.startThreads()执行后开启新一轮的select操作
                if (startLock.waitForStart(this))
                    return;
                // call poll()
                try {
                    每个帮助线程调用自己的subSelector对象进行select操作，检测自己管理的fd上是否有满足条件的就绪操作
                    subSelector.poll(index);
                } catch (IOException e) {
                    // Save this exception and let other threads finish.
                    finishLock.setException(e);
                }
                //记录当前帮助线程本轮select操作已经执行完毕，如果这个帮助线程是包括主线程和所有帮助线程在内的第一个从select
                //方法中返回的，它还要负责调用wakeUp方法唤醒其它阻塞在select操作上的线程                
                finishLock.threadFinished();
            }
        }
    }

    // 每一轮select操作结束后，所有帮助线程就会阻塞在startLock上等待进行下一轮的select操作
    private final StartLock startLock = new StartLock();

    /*
      为什么这里要使用一个startLock呢？我认为是为了减少每次select操作时创建线程的开销。
      我们通常会在主线程while循环中使用select操作：
          while(true) {
             selector.select();
             ...
          }
      帮助线程在启动时设置为了主线程的守护线程，帮助线程启动之后主线程不退出，帮助线程是不会退出的，并且每完成一次select操作就会进入休眠状态等待下一轮select操作的开始。

      每一轮select操作的开始是以主线程执行doSelect()方法中startLock.startThreads()作为标志的，这个方法执行后，主线程及所有帮助线程分别进行select操作检测其各自负责的Channel文件描述符上是否有感兴趣的事件就绪。
    */
    private final class StartLock {
        // runCounter变量加1，就代表下一轮select操作的开启
        // select方法调用一次，runsCounter就加一，超过long类型最大值怎么办？无影响。
        // 即使超过long类型最大值，只要runsCounter加1后的值与之前不一样就可以开启新一轮的select操作。
        private long runsCounter;
       // 方法调用后即开启新一轮的select操作
        private synchronized void startThreads() {
            runsCounter++; // next run
            notifyAll(); // 唤醒所有阻塞的帮助线程
        }
        // 帮助线程在最开始创建的时候或者完成一次select操作，就会阻塞在这个方法调用上，等待下一轮select操作的开始
        private synchronized boolean waitForStart(SelectThread thread) {
            while (true) {
                while (runsCounter == thread.lastRun) {
                    try {
                        startLock.wait();
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }
                if (thread.isZombie()) { // redundant thread
                    return true; // will cause run() to exit.
                } else {
                    // 通过设置thread.lastRun变量来保证这次运行之后，下一轮select操作开始之前，此帮助线程一直阻塞
                    thread.lastRun = runsCounter; 
                    return false; //   will cause run() to poll.
                }
            }
        }
    }

    // 主线程会阻塞在对finishLock的wait方法调用上，直到所有帮助线程select操作都结束。  
    private final FinishLock finishLock = new FinishLock();

    private final class FinishLock  {
        // 保存本轮select操作中尚未结束的帮助线程的数量
        private int threadsToFinish;

        // IOException which occured during the last run.
        IOException exception = null;

        // 每轮select操作开始前都要重置待结束帮助线程的数量
        private void reset() {
            threadsToFinish = threads.size(); // helper threads
        }

        /*
           在threadFinished()和waitForHelperThreads()方法中我们可以看到不管是主线程还是帮助线程，只要它是第一个结束的，就要
           唤醒其它线程，不管其检测的fd上是否有事件就绪，这是为什么呢？

           这是因为不管我们监测的描述符有多少，只有有一个fd上有感兴趣的操作就绪，就应该立即通知给方法调用者，所以只要有一个线程从
           select方法中返回了，就代表监测的fd中有事件就绪了，应该立即通知方法调用者，所以需要唤醒其它线程。  
        */


        // 每个帮助线程在一轮select操作结束之后会调用这个方法
        private synchronized void threadFinished() {
            if (threadsToFinish == threads.size()) { // finished poll() first
                // 如果当前线程是第一个结束的，需要唤醒其它线程，不管其检测的fd上是否有事件就绪
                wakeup();
            }
            threadsToFinish--;
            //如果主线程不是最后一个从select中返回的，主线程一定阻塞在对finishLock的wait方法的调用上
            //如果所有帮助线程都已结束，唤醒主线程继续执行
            if (threadsToFinish == 0) 
                notify();  
        }

        // 主线程调用这个方法等待所有帮助线程结束
        private synchronized void waitForHelperThreads() {
            if (threadsToFinish == threads.size()) {
                // 主线程是第一个结束的，需要唤醒其它线程，不管其检测的fd上是否有事件就绪
                wakeup();
            }
            while (threadsToFinish != 0) {
                try {
                    finishLock.wait();
                } catch (InterruptedException e) {
                    // Interrupted - set interrupted state.
                    Thread.currentThread().interrupt();
                }
            }
        }
        ...
    }
    ...
}
```

为什么要使用帮助线程？  
wakeUp是怎么唤醒的？怎么唤醒多个帮助线程的？  
FdMap介绍？SubSelector介绍？SubSelector poll processSelectedKeys processFDSet介绍？  
selector.close()？
channel.register()？


update操作及SubSelector上select操作介绍：  
```java
final class WindowsSelectorImpl extends SelectorImpl {

    private int updateSelectedKeys() {
        updateCount++;
        int numKeysUpdated = 0;
        numKeysUpdated += subSelector.processSelectedKeys(updateCount);
        for (SelectThread t: threads) {
            numKeysUpdated += t.subSelector.processSelectedKeys(updateCount);
        }
        return numKeysUpdated;
    }

    // SubSelector for the main thread
    private final SubSelector subSelector = new SubSelector();

    private final class SubSelector {
        private final int pollArrayIndex; // starting index in pollArray to poll
        // These arrays will hold result of native select().
        // The first element of each array is the number of selected sockets.
        // Other elements are file descriptors of selected sockets.
        private final int[] readFds = new int [MAX_SELECTABLE_FDS + 1];
        private final int[] writeFds = new int [MAX_SELECTABLE_FDS + 1];
        private final int[] exceptFds = new int [MAX_SELECTABLE_FDS + 1];

        private SubSelector() {
            this.pollArrayIndex = 0; // main thread
        }

        private SubSelector(int threadIndex) { // helper threads
            this.pollArrayIndex = (threadIndex + 1) * MAX_SELECTABLE_FDS;
        }

        private int poll() throws IOException{ // poll for the main thread
            return poll0(pollWrapper.pollArrayAddress,
                         Math.min(totalChannels, MAX_SELECTABLE_FDS),
                         readFds, writeFds, exceptFds, timeout);
        }

        private int poll(int index) throws IOException {
            // poll for helper threads
            return  poll0(pollWrapper.pollArrayAddress +
                     (pollArrayIndex * PollArrayWrapper.SIZE_POLLFD),
                     Math.min(MAX_SELECTABLE_FDS,
                             totalChannels - (index + 1) * MAX_SELECTABLE_FDS),
                     readFds, writeFds, exceptFds, timeout);
        }

        private native int poll0(long pollAddress, int numfds,
             int[] readFds, int[] writeFds, int[] exceptFds, long timeout);

        private int processSelectedKeys(long updateCount) {
            int numKeysUpdated = 0;
            numKeysUpdated += processFDSet(updateCount, readFds,
                                           PollArrayWrapper.POLLIN,
                                           false);
            numKeysUpdated += processFDSet(updateCount, writeFds,
                                           PollArrayWrapper.POLLCONN |
                                           PollArrayWrapper.POLLOUT,
                                           false);
            numKeysUpdated += processFDSet(updateCount, exceptFds,
                                           PollArrayWrapper.POLLIN |
                                           PollArrayWrapper.POLLCONN |
                                           PollArrayWrapper.POLLOUT,
                                           true);
            return numKeysUpdated;
        }

        /**
         * Note, clearedCount is used to determine if the readyOps have
         * been reset in this select operation. updateCount is used to
         * tell if a key has been counted as updated in this select
         * operation.
         *
         * me.updateCount <= me.clearedCount <= updateCount
         */
        private int processFDSet(long updateCount, int[] fds, int rOps,
                                 boolean isExceptFds)
        {
            int numKeysUpdated = 0;
            for (int i = 1; i <= fds[0]; i++) {
                int desc = fds[i];
                if (desc == wakeupSourceFd) {
                    synchronized (interruptLock) {
                        interruptTriggered = true;
                    }
                    continue;
                }
                MapEntry me = fdMap.get(desc);
                // If me is null, the key was deregistered in the previous
                // processDeregisterQueue.
                if (me == null)
                    continue;
                SelectionKeyImpl sk = me.ski;

                // The descriptor may be in the exceptfds set because there is
                // OOB data queued to the socket. If there is OOB data then it
                // is discarded and the key is not added to the selected set.
                if (isExceptFds &&
                    (sk.channel() instanceof SocketChannelImpl) &&
                    discardUrgentData(desc))
                {
                    continue;
                }

                if (selectedKeys.contains(sk)) { // Key in selected set
                    if (me.clearedCount != updateCount) {
                        if (sk.channel.translateAndSetReadyOps(rOps, sk) &&
                            (me.updateCount != updateCount)) {
                            me.updateCount = updateCount;
                            numKeysUpdated++;
                        }
                    } else { // The readyOps have been set; now add
                        if (sk.channel.translateAndUpdateReadyOps(rOps, sk) &&
                            (me.updateCount != updateCount)) {
                            me.updateCount = updateCount;
                            numKeysUpdated++;
                        }
                    }
                    me.clearedCount = updateCount;
                } else { // Key is not in selected set yet
                    if (me.clearedCount != updateCount) {
                        sk.channel.translateAndSetReadyOps(rOps, sk);
                        if ((sk.nioReadyOps() & sk.nioInterestOps()) != 0) {
                            selectedKeys.add(sk);
                            me.updateCount = updateCount;
                            numKeysUpdated++;
                        }
                    } else { // The readyOps have been set; now add
                        sk.channel.translateAndUpdateReadyOps(rOps, sk);
                        if ((sk.nioReadyOps() & sk.nioInterestOps()) != 0) {
                            selectedKeys.add(sk);
                            me.updateCount = updateCount;
                            numKeysUpdated++;
                        }
                    }
                    me.clearedCount = updateCount;
                }
            }
            return numKeysUpdated;
        }
    }
}
```






WindowsSelectorImpl类成员变量FdMap对应的内部类实现及其作用：    
```java
final class WindowsSelectorImpl extends SelectorImpl {

    private final FdMap fdMap = new FdMap();

    // Maps file descriptors to their indices in  pollArray
    private final static class FdMap extends HashMap<Integer, MapEntry> {
        static final long serialVersionUID = 0L;
        private MapEntry get(int desc) {
            return get(new Integer(desc));
        }
        private MapEntry put(SelectionKeyImpl ski) {
            return put(new Integer(ski.channel.getFDVal()), new MapEntry(ski));
        }
        private MapEntry remove(SelectionKeyImpl ski) {
            Integer fd = new Integer(ski.channel.getFDVal());
            MapEntry x = get(fd);
            if ((x != null) && (x.ski.channel == ski.channel))
                return remove(fd);
            return null;
        }
    }

    // class for fdMap entries
    private final static class MapEntry {
        SelectionKeyImpl ski;
        long updateCount = 0;
        long clearedCount = 0;
        MapEntry(SelectionKeyImpl ski) {
            this.ski = ski;
        }
    }
}
```
WindowsSelectorImpl类成员变量StartLock对应的内部类实现及其作用：  
```java
final class WindowsSelectorImpl extends SelectorImpl {

    // Helper threads wait on this lock for the next poll.
    private final StartLock startLock = new StartLock();

    private final class StartLock {
        // A variable which distinguishes the current run of doSelect from the
        // previous one. Incrementing runsCounter and notifying threads will
        // trigger another round of poll.
        private long runsCounter;
       // Triggers threads, waiting on this lock to start polling.
        private synchronized void startThreads() {
            runsCounter++; // next run
            notifyAll(); // wake up threads.
        }
        // This function is called by a helper thread to wait for the
        // next round of poll(). It also checks, if this thread became
        // redundant. If yes, it returns true, notifying the thread
        // that it should exit.
        private synchronized boolean waitForStart(SelectThread thread) {
            while (true) {
                while (runsCounter == thread.lastRun) {
                    try {
                        startLock.wait();
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }
                if (thread.isZombie()) { // redundant thread
                    return true; // will cause run() to exit.
                } else {
                    thread.lastRun = runsCounter; // update lastRun
                    return false; //   will cause run() to poll.
                }
            }
        }
    }
}
```
WindowsSelectorImpl类成员变量FinishLock对应的内部类实现及其作用：  
```java
final class WindowsSelectorImpl extends SelectorImpl {

    // Main thread waits on this lock, until all helper threads are done
    // with poll().
    private final FinishLock finishLock = new FinishLock();

    private final class FinishLock  {
        // Number of helper threads, that did not finish yet.
        private int threadsToFinish;

        // IOException which occured during the last run.
        IOException exception = null;

        // Called before polling.
        private void reset() {
            threadsToFinish = threads.size(); // helper threads
        }

        // Each helper thread invokes this function on finishLock, when
        // the thread is done with poll().
        private synchronized void threadFinished() {
            if (threadsToFinish == threads.size()) { // finished poll() first
                // if finished first, wakeup others
                wakeup();
            }
            threadsToFinish--;
            if (threadsToFinish == 0) // all helper threads finished poll().
                notify();             // notify the main thread
        }

        // The main thread invokes this function on finishLock to wait
        // for helper threads to finish poll().
        private synchronized void waitForHelperThreads() {
            if (threadsToFinish == threads.size()) {
                // no helper threads finished yet. Wakeup them up.
                wakeup();
            }
            while (threadsToFinish != 0) {
                try {
                    finishLock.wait();
                } catch (InterruptedException e) {
                    // Interrupted - set interrupted state.
                    Thread.currentThread().interrupt();
                }
            }
        }

        // sets IOException for this run
        private synchronized void setException(IOException e) {
            exception = e;
        }

        // Checks if there was any exception during the last run.
        // If yes, throws it
        private void checkForException() throws IOException {
            if (exception == null)
                return;
            StringBuffer message =  new StringBuffer("An exception occured" +
                                       " during the execution of select(): \n");
            message.append(exception);
            message.append('\n');
            exception = null;
            throw new IOException(message.toString());
        }
    }
}
```
WindowsSelectorImpl类成员变量SubSelector对应的内部类实现及其作用：
```java
final class WindowsSelectorImpl extends SelectorImpl {

    private final SubSelector subSelector = new SubSelector();

    private final class SubSelector {
        private final int pollArrayIndex; // starting index in pollArray to poll
        // These arrays will hold result of native select().
        // The first element of each array is the number of selected sockets.
        // Other elements are file descriptors of selected sockets.
        private final int[] readFds = new int [MAX_SELECTABLE_FDS + 1];
        private final int[] writeFds = new int [MAX_SELECTABLE_FDS + 1];
        private final int[] exceptFds = new int [MAX_SELECTABLE_FDS + 1];

        private SubSelector() {
            this.pollArrayIndex = 0; // main thread
        }

        private SubSelector(int threadIndex) { // helper threads
            this.pollArrayIndex = (threadIndex + 1) * MAX_SELECTABLE_FDS;
        }

        private int poll() throws IOException{ // poll for the main thread
            return poll0(pollWrapper.pollArrayAddress,
                         Math.min(totalChannels, MAX_SELECTABLE_FDS),
                         readFds, writeFds, exceptFds, timeout);
        }

        private int poll(int index) throws IOException {
            // poll for helper threads
            return  poll0(pollWrapper.pollArrayAddress +
                     (pollArrayIndex * PollArrayWrapper.SIZE_POLLFD),
                     Math.min(MAX_SELECTABLE_FDS,
                             totalChannels - (index + 1) * MAX_SELECTABLE_FDS),
                     readFds, writeFds, exceptFds, timeout);
        }

        private native int poll0(long pollAddress, int numfds,
             int[] readFds, int[] writeFds, int[] exceptFds, long timeout);

        private int processSelectedKeys(long updateCount) {
            int numKeysUpdated = 0;
            numKeysUpdated += processFDSet(updateCount, readFds,
                                           PollArrayWrapper.POLLIN,
                                           false);
            numKeysUpdated += processFDSet(updateCount, writeFds,
                                           PollArrayWrapper.POLLCONN |
                                           PollArrayWrapper.POLLOUT,
                                           false);
            numKeysUpdated += processFDSet(updateCount, exceptFds,
                                           PollArrayWrapper.POLLIN |
                                           PollArrayWrapper.POLLCONN |
                                           PollArrayWrapper.POLLOUT,
                                           true);
            return numKeysUpdated;
        }

        /**
         * Note, clearedCount is used to determine if the readyOps have
         * been reset in this select operation. updateCount is used to
         * tell if a key has been counted as updated in this select
         * operation.
         *
         * me.updateCount <= me.clearedCount <= updateCount
         */
        private int processFDSet(long updateCount, int[] fds, int rOps,
                                 boolean isExceptFds)
        {
            int numKeysUpdated = 0;
            for (int i = 1; i <= fds[0]; i++) {
                int desc = fds[i];
                if (desc == wakeupSourceFd) {
                    synchronized (interruptLock) {
                        interruptTriggered = true;
                    }
                    continue;
                }
                MapEntry me = fdMap.get(desc);
                // If me is null, the key was deregistered in the previous
                // processDeregisterQueue.
                if (me == null)
                    continue;
                SelectionKeyImpl sk = me.ski;

                // The descriptor may be in the exceptfds set because there is
                // OOB data queued to the socket. If there is OOB data then it
                // is discarded and the key is not added to the selected set.
                if (isExceptFds &&
                    (sk.channel() instanceof SocketChannelImpl) &&
                    discardUrgentData(desc))
                {
                    continue;
                }

                if (selectedKeys.contains(sk)) { // Key in selected set
                    if (me.clearedCount != updateCount) {
                        if (sk.channel.translateAndSetReadyOps(rOps, sk) &&
                            (me.updateCount != updateCount)) {
                            me.updateCount = updateCount;
                            numKeysUpdated++;
                        }
                    } else { // The readyOps have been set; now add
                        if (sk.channel.translateAndUpdateReadyOps(rOps, sk) &&
                            (me.updateCount != updateCount)) {
                            me.updateCount = updateCount;
                            numKeysUpdated++;
                        }
                    }
                    me.clearedCount = updateCount;
                } else { // Key is not in selected set yet
                    if (me.clearedCount != updateCount) {
                        sk.channel.translateAndSetReadyOps(rOps, sk);
                        if ((sk.nioReadyOps() & sk.nioInterestOps()) != 0) {
                            selectedKeys.add(sk);
                            me.updateCount = updateCount;
                            numKeysUpdated++;
                        }
                    } else { // The readyOps have been set; now add
                        sk.channel.translateAndUpdateReadyOps(rOps, sk);
                        if ((sk.nioReadyOps() & sk.nioInterestOps()) != 0) {
                            selectedKeys.add(sk);
                            me.updateCount = updateCount;
                            numKeysUpdated++;
                        }
                    }
                    me.clearedCount = updateCount;
                }
            }
            return numKeysUpdated;
        }
    }
}
```


#### **唤醒阻塞的select**  
abstract Selector wakeup():

#### **关闭selector**  
abstract void close():

对于select方式的底层实现Java实际上是通过调用native方法进而调用底层系统调用来实现的，比如linux上调用epoll系统调用。  