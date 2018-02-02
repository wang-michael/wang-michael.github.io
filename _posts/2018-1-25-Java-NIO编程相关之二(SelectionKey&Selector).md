---
layout:     post
title:      Java-NIO编程相关之二(SelectionKey&Selector)
date:       2018-1-25
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Java基础 
    - JDK源码阅读
---

>本文主要记录了个人在学习NIO相关知识时一些思考过程，主要用于备忘，错误难免，敬请指出！
  
_ _ _
### **前言**

Java NIO API中包含三个核心组件：Channels、 Buffers、 Selector,其余组件有些只不过是为了这三个组件能更好的结合使用而设计的一些工具类，本文主要记录对于Selector相关知识的学习。    

_ _ _
### **Selector**  
**Select模型的意义并不在于消除阻塞，而是在于消除阻塞之后支持的多路复用机制，非阻塞读写很容易实现，但没有Select模型，如何实现类似Selector的1：N线程模型高效的处理非阻塞读写就会非常复杂。**   

**问题**：为什么Java Selector要求channel工作在非阻塞模式下呢？  
我的理解：os内部select系统调用首先会对要读写的设备进行轮询，若轮询时没有设备可读写，则调用select的进程进入睡眠状态(并不会一直轮询)，当有设备变得可读写时，相应设备会唤醒睡眠进程，从而继续往下执行。这就实现了select的当有一个文件描述符可操作时就立即唤醒执行的基本原理。Java Selector本质上还是基于底层os的select系统调用实现的，在select轮询监测设备的时候不能陷入阻塞，所以Java Selector要求Channel工作在非阻塞模式下。   

os中select()内部实现管理可参考：[Linux:select()详解和实现原理](http://www.cnblogs.com/sky-heaven/p/7205491.html)    

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
        //创建PollArrayWrapper对象并设置初始容量为8
        pollWrapper = new PollArrayWrapper(INIT_CAP);

        //创建用来进行wakeUp操作的Pipe，并将其SourceChannel FD交由pollWrapper保存。
        wakeupPipe = Pipe.open();
        wakeupSourceFd = ((SelChImpl)wakeupPipe.source()).getFDVal();

        // Disable the Nagle algorithm so that the wakeup is more immediate
        SinkChannelImpl sink = (SinkChannelImpl)wakeupPipe.sink();
        //禁用nagle算法，使得wakeUp操作更快
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
1. 创建PollArrayWrapper对象并设置初始容量为8，意味着初始情况下在其中最多可以保存8个Channel对应的fd及其感兴趣的事件。  
2. 创建一个Pipe，并将pipe的SourceChannel对应的fd也交由PollArrayWrapper管理，这是为了实现Selector上的wakeUp方法，之后会具体介绍wakeUp实现方式。    

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
对于select方式的底层实现Java实际上是通过调用native方法进而调用底层系统调用来实现的，比如Windows上使用select方式，linux上调用epoll系统调用。为了更好的使用Selector提供给我们的API，最好理解其内部实现方式；下面就以windows下JDK1.8为例来看下select()方法的内部实现，具体分析见注释：  
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
            //如果wakeUp被调用过，则读出pipe中用来wakeUp的数据，本次select操作立即返回
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
        final SubSelector subSelector;//用来进行select操作的SubSelector对象
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
                    // 每个帮助线程调用自己的subSelector对象进行select操作，检测自己管理的fd上是否有满足条件的就绪操作
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
接下来对代码注释中涉及的几个重点内容做进一步的解释。  
1. **为什么要使用帮助线程？**由于windows下select系统调用可管理的fd的最大数量为1024，当注册到单个Selector上的fd数量超过1024的时候就需要使用另一个线程，也就是所谓的帮助线程进行另一个select系统调用管理超出部分的fd。具体示意图如下：  <img src="/img/2018-1-25/selectHelperThreads.jpg" width="500" height="500" alt="引入帮助线程帮助管理fd示意图" /><center>图3：引入帮助线程帮助管理fd示意图</center>  
2. **wakeUp是怎么唤醒的？怎么唤醒多个帮助线程的？**根据API文档中所述，如果A线程调用selector.select()方法陷入了阻塞，此时在B线程上调用selector.wakeUp()就可以唤醒阻塞的A线程；如果B线程调用selector.wakeUp()时没有任何线程阻塞在select()方法上，那么下一个调用selector.select()方法的线程不会阻塞，立即返回。在上面对select()源码进行分析的过程中我们也可以看到，对于每一轮select操作，主线程和帮助线程会分别对其管理的fd调用select系统调用监听事件就绪状态，只要有一个线程监听的fd中有事件就绪，就会调用wakeUp()方法唤醒其它所有的线程，这与我们的理解是相符的，下面就来看下wakeUp唤醒阻塞线程的原理：  
```java
final class WindowsSelectorImpl extends SelectorImpl {

    public Selector wakeup() {
        synchronized (interruptLock) {
            if (!interruptTriggered) {
                setWakeupSocket();
                interruptTriggered = true;
            }
        }
        return this;
    }

    private void setWakeupSocket() {
        setWakeupSocket0(wakeupSinkFd);
    }

    private native void setWakeupSocket0(int wakeupSinkFd);

    ...
}
```
可以看到实际wakeUp方法实际调用的是native的setWakeupSocket0()方法，这个方法做了什么呢？其源码如下：    
```c
JNIEXPORT void JNICALL
Java_sun_nio_ch_WindowsSelectorImpl_setWakeupSocket0(JNIEnv *env, jclass this, jint scoutFd) {
    /* Write one byte into the pipe */
    const char byte = 1;
    send(scoutFd, &byte, 1, 0);
}
```
内容很简单，向pipe中写入了一个字节。为什么向pipe中写入一个字节就可以唤醒阻塞的线程呢？因为我们在创建WindowsSelectorImpl对象时，同时创建了一个pipe，将pipe的sourceFd交由PollArrayWrapper管理，每新开一个帮助线程时，其管理的第一个文件描述符也是pipe的sourceFd(如图3所示)，当使用pipe.sinkFd向pipe中写入一个字节时，所有线程都会检测到pipe。sourceFd上的可读事件就绪，当然就会从阻塞中返回，这就是wakeUp方法的实现原理。**将一个阻塞在select操作上的线程从阻塞状态中唤醒有三种方式**：调用selector.wakeUp()、调用selector.close()、在阻塞线程上调用Thread.interrupt方法，其实selector.close()与Thread.interrupt()方法原理也是调用wakeUp()方法唤醒阻塞线程。多个线程同时调用一个selector上的select()方法陷入阻塞，只有一个线程是由于调用native select方法等待感事件就绪陷入阻塞，其余线程是由于要获取的锁被占用而陷入阻塞。wakeUp()方法唤醒的是由于调用native select方法陷入阻塞的线程。  

3. **startLock与finishLock的作用？**用来进行主线程与帮助线程中的同步，主线程通过startLock来开启新一轮的select操作，帮助线程完成一轮select操作之后会阻塞在startLock上等待进行下一轮select操作。主线程执行结束之后会阻塞在finishLock上等待所有帮助线程执行结束之后才解除阻塞向下执行。  

#### **更新selected Key**    
对于每个Selector类对象，其内部维持了三个SelectionKey相关的集合：  
* key set：通过selector.keys()方法获取，保存了注册到当前selector上的所有channel对应的SelectionKey的集合。返回的set是不可更改的，即对此set上进行的add、remove及其衍生操作都会抛出UnsupportedOperationException。当一个Channel注册到Selector上时，其对应的SelectionKey会被隐式的添加到key set集合中。被cancelled的SelectionKey会在select操作中从Selector的key set中隐式移除。      
* selected-key：通过selectedKeys()方法获取，保存那些至少有一个感兴趣的事件已经就绪的Channel对应的SelectionKey，selected-key set中保存的SelectionKey是key set的子集。selected-key set是不能向其中手动添加数据，即对此set上进行的add及其衍生操作都会抛出UnsupportedOperationException。在select操作执行之后，有事件就绪的SelectionKey会被加入到selected Key集合。可以使用set.remove或者set.iterator.remove将处理过的key从selected key中删除，并且如果不主动删除处理过的SelectionKey，这个Key就会一直存在在selected key set中。  
* cancelled-key：此集合保存selectionKey被cancel但是其对应的channel还没有从selector上解除注册的SelectionKey的集合，此集合不能被直接获取，通常其是key set的子集。一个SelectionKey被添加到cancelled-key set集合中可能有两种情况：一是对应的channel被关闭，二是在SelectionKey上调用cancel方法，其实本质上只有一种，因为关闭channel时也是调用的SelectionKey上的cancel方法。被cancelled的SelectionKey对应的channel会在下次select操作中从selector中解除注册。      

在一个SelectionKey的cancel方法被调用之后，如果调用时其对应的selector上没有select操作正在进行，那么等到下次select操作时，这个key才会真正的从selector中移除；如果调用时有select操作正在进行，可能在本次、也可能在下一次select操作中将key真正的从selector中移除。  

将一个SelectionKey加入到CanceledSet之后，真正的移除工作做了这几件事情：1、从selector的key-set中移除此key 2、从selector的selected-key set中移除此key 3、从此key对应的channel中保存的SelectionKey数组中移除此key 4、从selector的cancelled-key set中移除此key 5、设置此key为invalid，调用其isValid()返回值为false。  

对于程序员来讲，相比于select操作的内部实现，更为关注的是select操作执行后的结果，通常也就是更新后的selectedKey集合，通过这个集合可以知道哪些channel有事件就绪，进行相应处理操作。下面就来通过源码看一下select操作执行之后具体是如何更新selected-key集合的：     
```java
final class WindowsSelectorImpl extends SelectorImpl {

    private long updateCount = 0;    

    protected int doSelect(long timeout) throws IOException {
        ...    
        subSelector.poll();//监听管理的fd集合是否有就绪事件发生
        ...
        int updated = updateSelectedKeys();//更新selectedKeySet
        ...
        return updated;//返回与上次select操作相比selectionKey就绪状态有更新的数量
    }

    private int updateSelectedKeys() {
        //初始情况下为0，每调用一次updateCount就加一
        updateCount++;
        int numKeysUpdated = 0;
        //遍历主线程及所有帮助线程中包含的就绪fd，计算出与上次select相比就绪状态发生变化的fd总数
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
        // 这三个数组保存了select系统调用返回的结果的集合，是poll0函数中的入参
        // 调用后三个数组中第一个元素的值代表就绪fd的个数
        // 第一个元素之外的其它元素代表具体就绪的fd
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

        //poll0返回之后readFds中保存读就绪的fd，writeFds中保存写就绪的fd，exceptFds保存有例外数据就绪的fd
        private native int poll0(long pollAddress, int numfds,
             int[] readFds, int[] writeFds, int[] exceptFds, long timeout);

        private int processSelectedKeys(long updateCount) {         
            /*先处理可读的fd，之后是可写的fd，之后是有例外数据的fd。
              如果一个SelectionKey之前就存在于selected-key set中并未被移除，之前就绪为可读可写，本次select操作后仍然可读可写，
              则 processSelectedKeys()方法处理过程中其fd就绪状态变化为 读写->读->读写就绪，返回numKeysUpdated为1.
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
                /*
                 fdMap的key值为注册到此selector上channel对应的fd，value值为包装为MapEntry的SelectionKey,具体包装方式如下：
                  private final static class MapEntry {
                     SelectionKeyImpl ski;
                     long updateCount = 0;
                     long clearedCount = 0;
                     MapEntry(SelectionKeyImpl ski) {
                          this.ski = ski;
                     }
                  }
                  这样包装的目的是为了根据native select调用返回的就绪fd能快速找到其对应的SelectionKey
                */
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

                /*
                  情况1：channel A 在第一次select操作中监测到读就绪，之后在selected-key set中对此fd不做任何处理，第二次select后没有监
                        测到此Channel就绪，则这次select操作返回的就绪状态有更新的fd数量中不将此fd记录在内。  
                  情况2：channel A 在第一次select操作中监测到读就绪，之后在selected-key set中对此fd不做任何处理，第二次select后仍然监
                        测到此Channel读就绪，与上次select监测到的状态相同，则这次select操作返回的就绪状态有更新的fd不将此fd记录在内。
                  情况3：channel A 在第一次select操作中监测到读就绪，之后在selected-key set中对此fd不做任何处理，第二次select后监
                        测到此Channel写就绪，与上次select监测到的状态不同，则这次select操作返回的就绪状态有更新的fd将此fd记录在内。
                  情况4：channel A 注册对读感兴趣，channel A 在第一次select操作中监测到读就绪，之后在selected-key set中对此fd处理后将
                        此fd从中删除，第二次监测到channel写就绪，由于此Channel注册时对写就绪不感兴趣，调用前channel A中readyOps为读就绪，调用后channel A readyOps设置为0，这次select操作返回的就绪状态有更新的fd不将此fd记录在内。    
                */
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
                    if (me.clearedCount != updateCount) {//情况4会进入这个if中
                        sk.channel.translateAndSetReadyOps(rOps, sk);//执行完这一句之后，情况4 readyOps变为0
                        if ((sk.nioReadyOps() & sk.nioInterestOps()) != 0) {//情况4不能进入这个if中
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
**select()调用后返回值(和上次select相比就绪fd数量有更新的数量)的计算方法**：  

1、调用时当前channel对应的selection Key不在selected-key set中，则会清除selection key中之前的就绪信息，判断当前channel的就绪事件与其注册的感兴趣的事件是否有交集，如果有则设置其readyOps为此交集，并使select返回值加1。  

2、调用时当前channel对应的selection Key在selected-key set中，首先记录selection key中保存的旧的就绪信息，之后清除selection key中之前的旧的就绪信息，判断当前channel的就绪事件与其注册的感兴趣的事件是否有交集，如果有则设置其readyOps为此交集；若没有交集，则select返回值不变。之后判断现在的readyOps与之前的readyOps相比是否增加了新的就绪操作(比如由读变为写就绪、或由读就绪变为读写就绪)，如果增加了的话就使select返回值加1，如果没增加新的就绪操作的话(比如原来读就绪，现在仍然为读就绪，或者原来为读写就绪，现在仅为写就绪)返回值不变。**需要注意的一种情况是如果channel之前是读写就绪的，下一次select操作后仍然是读写就绪的，第二次select操作这个channel会不会计入到select返回值的计算中呢？**答案是会的，因为其就绪操作在源代码WindowsSelectorImpl内部类SubSelector.processSelectedKeys()中的变化过程为读写就绪->读就绪->读写就绪，由读就绪变为读写就绪的过程中认为增加了新的就绪操作，所以返回值会加1。   

JDK官方API文档中关于select()方法有这样一段话让我感觉很迷惑：         
<img src="/img/2018-1-25/JDKAPISelect.jpg" width="800" height="800" alt="JDK API select操作相关介绍" />
<center>图4：JDK API select操作相关介绍</center>  

按照我的理解，这种说法的意思是对于之前已经保存在selected-key set中的SelectionKey，在本次select操作中SelectionKey之前含有的就绪信息会被保留，新的就绪信息会在原有基础上增加，这与我上面所说的select()返回值计算中第2点是矛盾的。按照这种理解方式，如果一个SocketChannel之前select操作中监测到读写就绪，之后在selected-key set中保留其对应的SelectionKey，下一次select操作中此SocketChannel仅仅写就绪，但是我们通过SelectionKey获得的Channel的就绪信息还会是读写就绪。实际是不是这样呢？经我测试这种理解方式是不对的，第二次select操作会更新就绪信息由读写就绪变为写就绪。测试Demo如下：    
```java
public class SelectorServer {

    private static final String SERVERIP = "localhost";
    private static final int PORT = 8000;
    private static Selector selector;

    public static void selectorCommonUseStyle() throws IOException, InterruptedException {
        selector = Selector.open();
        ServerSocketChannel channel = ServerSocketChannel.open();
        channel.configureBlocking(false);
        channel.bind(new InetSocketAddress(SERVERIP, PORT));
        channel.register(selector, SelectionKey.OP_ACCEPT);

        while(true) {
            System.out.println("准备select");            
            int readyChannels = selector.select();
            System.out.println("readyChannels: " + readyChannels);
            Set<SelectionKey> selectedKeys = selector.selectedKeys();
            Iterator<SelectionKey> keyIterator = selectedKeys.iterator();

            while(keyIterator.hasNext()) {
                SelectionKey key = keyIterator.next();
                if(key.isAcceptable()) {
                    System.out.println(key.channel() + "acceptable!");
                    SocketChannel socketChannel = ((ServerSocketChannel) key.channel()).accept();
                    socketChannel.configureBlocking(false);
                    //socketChannel对应的SelectionKey一旦被加入selected-key set，就不会移除
                    socketChannel.register(selector, SelectionKey.OP_READ | SelectionKey.OP_WRITE);
                    keyIterator.remove();
                }
                if (key.isConnectable()) {                    
                    System.out.println(key.channel() + "isConnectable!");
                }
                if (key.isReadable()) {                    
                    System.out.println(key.channel() + "isReadable");
                    ByteBuffer src = ByteBuffer.allocate(1024);
                    ((SocketChannel) key.channel()).read(src);
                    src.flip();
                    byte[] temp = new byte[1024];
                    src.get(temp, 0, src.limit());
                    System.out.println(new String(temp));
                }
                if (key.isWritable()) {                   
                    System.out.println(key.channel() + "isWritable");
                }
            }
            Thread.sleep(5000);
            System.out.println("-----------------------------");
        }
    }

    public static void main(String[] args) throws IOException, InterruptedException {
        selectorCommonUseStyle();
    }
}

public class SocketChannelClient {

    private static final String SERVERIP = "localhost";
    private static final int PORT = 8000;

    public static void main(String[] args) throws IOException, InterruptedException {
        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.connect(new InetSocketAddress(InetAddress.getByName(SERVERIP), PORT));
        socketChannel.configureBlocking(false);

        ByteBuffer src = ByteBuffer.allocate(1024);
        byte[] dst = new byte[1024];

        src.put("ping".getBytes());
        src.flip();//写模式切换到读模式

        Thread.sleep(11000);
        while (src.position() < src.limit()) {
            socketChannel.write(src);
        }

        /*加上这段代码会有连续两次的readyChannels为1，SocketChannel可读可写的输出
        Thread.sleep(6000);
        src.clear();
        src.put("ping".getBytes());
        src.flip();//写模式切换到读模式
        while (src.position() < src.limit()) {
            socketChannel.write(src);
        }
        */

        int num;
        src.clear();//读模式切换到写模式
        while ((num = socketChannel.read(src)) != -1) {
            if (num > 0) {
                src.flip();
                src.get(dst, 0, src.limit());
                System.out.println("收到回复:" + new String(dst));
            }
        }

    }
}

/*
SelectorServer输出结果如下：  
准备select
readyChannels: 1
sun.nio.ch.ServerSocketChannelImpl[/127.0.0.1:8000]acceptable!
-----------------------------
准备select
readyChannels: 1
java.nio.channels.SocketChannel[connected local=/127.0.0.1:8000 remote=/127.0.0.1:1041]isWritable
-----------------------------
准备select
readyChannels: 0
java.nio.channels.SocketChannel[connected local=/127.0.0.1:8000 remote=/127.0.0.1:1041]isWritable
-----------------------------
准备select
readyChannels: 1
java.nio.channels.SocketChannel[connected local=/127.0.0.1:8000 remote=/127.0.0.1:1041]isReadable
ping                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            
java.nio.channels.SocketChannel[connected local=/127.0.0.1:8000 remote=/127.0.0.1:1041]isWritable
-----------------------------
准备select
readyChannels: 0
java.nio.channels.SocketChannel[connected local=/127.0.0.1:8000 remote=/127.0.0.1:1041]isWritable
-----------------------------
*/
```
由SelectorServer输出的结果可以看出注册在selector上的SocketChannel对应的SelectionKey虽然一直在selected-key set中，但其就绪状态经历了写就绪->读写就绪->写就绪的变化，并且由于由读写就绪切换到写就绪的变化过程中每个状态相比于上一个状态并没有产生新的就绪操作，所以readyChannels数量并没有增加。  

**使用SelectionKey interestOps(int ops)方法更改注册到selector上的感兴趣事件何时生效？**

如果调用interestOps(int ops)方法时有select操作正在运行，本次select操作中不一定能检测到感兴趣事件的改动，但下次select操作一定可以检测到。  

**在Selected-key Set中的SelectionKey对应的Channel不一定是真正有就绪操作发生的。**这可能有以下几种情况：
* 有可能前一次select操作中监测到某个Channel有感兴趣的事件就绪，加入到selected-key set之后不对其做任何处理，第二次select操作之后实际上此Channel没有任何事件就绪，但其仍在selected-key set中保存，并且其中保存的就绪信息仍是第一次select时储存的就绪信息。  
* 在一次select过程中，某个SelectionKey被加入到了selected-key set中，但是在select过程中此selectionKey上的cancel方法已经被调用，这个key实际是无效的。
* 在一次select过程中，某个SelectionKey被加入到了selected-key set中，但是在select过程中此selectionKey对应的Channel被关闭(实际也是调用了selectionKey上的cancel方法)，这个虽然在selected-key set中，但实际是无效的。

#### **关闭Selector**    
首先看下selector.close()在源代码中的执行流程：  
```java
public abstract class Selector implements Closeable {

    public abstract void close() throws IOException;

}

public abstract class AbstractSelector extends Selector {

    public final void close() throws IOException {
        //如果之前close()方法已经被调用过，则直接返回
        boolean open = selectorOpen.getAndSet(false);
        if (!open)
            return;
        implCloseSelector();
    }

    protected abstract void implCloseSelector() throws IOException;
    ...
}

public abstract class SelectorImpl extends AbstractSelector {

    public void implCloseSelector() throws IOException {
        //如果有线程正阻塞在对selector方法的调用上，则会被wakeUp()方法唤醒
        this.wakeup();
        synchronized(this) {
            Set var2 = this.publicKeys;
            synchronized(this.publicKeys) {
                Set var3 = this.publicSelectedKeys;
                synchronized(this.publicSelectedKeys) {
                    this.implClose();
                }
            }

        }
    }

    protected abstract void implClose() throws IOException;
    ...
}

final class WindowsSelectorImpl extends SelectorImpl {
    //进行资源清理操作
    protected void implClose() throws IOException {
        Object var1 = this.closeLock;
        synchronized(this.closeLock) {
            if(this.channelArray != null && this.pollWrapper != null) {
                Object i = this.interruptLock;
                synchronized(this.interruptLock) {
                    this.interruptTriggered = true;
                }

                this.wakeupPipe.sink().close();
                this.wakeupPipe.source().close();

                //解除所有注册到当前selector上的Channel
                for(int var7 = 1; var7 < this.totalChannels; ++var7) {
                    if(var7 % 1024 != 0) {
                        this.deregister(this.channelArray[var7]);
                        SelectableChannel t = this.channelArray[var7].channel();
                        if(!t.isOpen() && !t.isRegistered()) {
                            ((SelChImpl)t).kill();
                        }
                    }
                }

                //释放pollWrapper所占有的堆外内存空间
                this.pollWrapper.free();
                this.pollWrapper = null;
                this.selectedKeys = null;
                this.channelArray = null;
                //退出所有的帮助线程
                Iterator var8 = this.threads.iterator();

                while(var8.hasNext()) {
                    WindowsSelectorImpl.SelectThread var9 = (WindowsSelectorImpl.SelectThread)var8.next();
                    var9.makeZombie();
                }

                this.startLock.startThreads();
            }

        }
    }
    ...
}
```
可以看出，close()方法所做的主要工作有两部分，一是唤醒阻塞在Selector.select()方法上的所有线程，二是进行必要的资源清理工作。资源清理时close()方法带来的边际效应是将注册到其上的SelectionKey从对应的Channel中删除。    

一个Selector被关闭后，除了wakeUp()和close()方法之外的任何其他方法调用都会抛出ClosedSelectorException。实际上selector关闭后调用wakeUp方法和close方法也不会有任何作用。  

#### **Selector线程安全问题**    
Selector本身应用于多线程情况下是线程安全的，但是通过其获取的Set本身并不是线程安全的。比如多线程调用同一个selector上的selectedKeys()方法对获取的set做remove操作，此时就需要注意线程安全问题。  

#### **Channel在Selector上的注册**  
Channel.register()方法在执行时如果其对应的Selector上的select或close方法正在执行，register方法会阻塞到这两个方法执行完毕之后才能执行。AbstractChannel及其子类注册过程都是下面的代码：      
```java
public abstract class AbstractSelectableChannel extends SelectableChannel {

    public final SelectionKey register(Selector sel, int ops, Object att) throws ClosedChannelException {
        synchronized (regLock) {
            if (!isOpen())
                throw new ClosedChannelException();
            if ((ops & ~validOps()) != 0)
                throw new IllegalArgumentException();
            if (blocking)
                throw new IllegalBlockingModeException();
            SelectionKey k = findKey(sel);
            //如果之前注册过本次相当于更改SelectionKey的ops和att属性
            if (k != null) {
                k.interestOps(ops);
                k.attach(att);
            }
            if (k == null) {
                // New registration
                synchronized (keyLock) {
                    if (!isOpen())
                        throw new ClosedChannelException();
                    //在对应Selector上注册Channel
                    k = ((AbstractSelector)sel).register(this, ops, att);
                    //将注册后获得的SelectionKey在Channel中保存一份
                    addKey(k);
                }
            }
            return k;
        }
    }
    ...
}

public abstract class SelectorImpl extends AbstractSelector {

    protected final SelectionKey register(AbstractSelectableChannel ch, int ops, Object attachment) {
        if(!(ch instanceof SelChImpl)) {
            throw new IllegalSelectorException();
        } else {
            SelectionKeyImpl k = new SelectionKeyImpl((SelChImpl)ch, this);
            k.attach(attachment);
            Set var5 = this.publicKeys;
            //需要获得publicKeys上的锁才能继续执行，Selector.select()执行时也要获得publicKeys上的锁
            synchronized(this.publicKeys) {
                this.implRegister(k);
            }

            k.interestOps(ops);
            return k;
        }
    }
    ...
}

final class WindowsSelectorImpl extends SelectorImpl {

    //在Selector上的实际注册操作
    protected void implRegister(SelectionKeyImpl ski) { 
        Object var2 = this.closeLock;
        synchronized(this.closeLock) {
            if(this.pollWrapper == null) {
                throw new ClosedSelectorException();
            } else {
                this.growIfNeeded();
                this.channelArray[this.totalChannels] = ski;
                ski.setIndex(this.totalChannels);
                this.fdMap.put(ski);
                this.keys.add(ski);
                this.pollWrapper.addEntry(this.totalChannels, ski);
                ++this.totalChannels;
            }
        }
    }
    ...
}
```
对于AbstractSelectableChannel.isRegistered()方法，只要当前Channel注册到了任何一个Selector上，就会返回true。需要注意的是有可能当Channel所有的SelectionKey都被Cancel之后调用此方法仍然返回true，这是因为cancel一个SelectionKey并不会立即解除Selector与Channel之间的注册关系，而是要等到Selector上的下一次select操作进行或者Selector被关闭才会解除。  

关于Selector使用更详细的示例可参考：[java-nio-server](https://github.com/wang-michael/java-nio-server)    
源代码forked by jjenkov，在其基础上做了一些修改，使用Java NIO原生API实现了一个简单的HTTP服务器。  

最后附上一张Selector工作流程图：  
<img src="/img/2018-1-25/selectorwork.jpg" width="800" height="800" alt="Selector工作流程图" />
<center>图5：Selector工作流程图</center>  

参考文章：  
[《Java 源码分析》：Java NIO 之 Selector(第一部分Selector.open())](http://blog.csdn.net/u010412719/article/details/52809669)  
[Java NIO Tutorial](http://tutorials.jenkov.com/java-nio/index.html)   

（完）

