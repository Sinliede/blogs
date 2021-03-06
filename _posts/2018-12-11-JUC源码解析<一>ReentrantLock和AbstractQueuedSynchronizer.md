## 1.前言
在了解j.u.c包的时候，总是会避不开AbstractQueuedSynchronizer这个抽象类，这个类是j.u.c包多线程操作的核心类，像CountDownLatch，ReentrantLoack，Semaphore等类都是以此类为基础实现的。本篇文章将尝试通过阅读AbstractQueuedSynchronizer的源码，去了解并发大师Doug Lea在解决高并发问题时的思路。

## 2.ReentrantLock
在处理高并发任务时，很多时候都会用到ReentrantLock对线程共有资源加锁，解决因多线程竞争共有资源而产生的问题。
ReentrantLock的典型使用使用方式Doug Lea在ThreadPoolExecutor中的addWaitor()方法中已经给出示范。

```java
final ReentrantLock mainLock = this.mainLock;
mainLock.lock();
try {
    // Recheck while holding lock.
    // Back out on ThreadFactory failure or if
    // shut down before lock acquired.
    int rs = runStateOf(ctl.get());

    if (rs < SHUTDOWN ||
        (rs == SHUTDOWN && firstTask == null)) {
        if (t.isAlive()) // precheck that t is startable
            throw new IllegalThreadStateException();
        workers.add(w);
        int s = workers.size();
        if (s > largestPoolSize)
            largestPoolSize = s;
        workerAdded = true;
    }
} finally {
    mainLock.unlock();
}
```
其中workers是一个由多个线程共有的HashSet，HashSet的add方法是线程不安全的，因此我们用ReentrantLock加锁，保证同一时刻只能有一个线程进行add操作。

看一下ReentrantLock的类结构

```java
    private final Sync sync;
    abstract static class Sync extents AbstractQueuedSynchronizer{ 
        abstract void lock();
    
        final boolean nonfairTryAcquire(int acquires) {...}
        
        ...
    }
    static final class NonfairSync extends Sync {...}
    
    static final class FairSync extends Sync { 
        final void lock() {acquire(1);}
        
        protected final boolean tryAcquire(int acquires) {...}
    }

    //ReentrantLock获取锁的方法
    public void lock() {
        sync.lock();
    }
    ...
```
可以看到ReentrantLock中存在一个继承了AbstractQueuedSynchronizer类的Sync对象，ReentrantLock的lock方法会调用sync的lock方法，sync的lock方法由两个实现类NonfairSync和FairSync实现。

## 3.AbstractQueuedSynchronizer
再来简单看一下AbstractQueuedSynchronizer的类结构。

```java
    //构建队列的节点类
    static final class Node{
        volatile int waitStatus;	//节点的等待状态，0代表初始化，-1代表等待唤起
        volatile Node prev;				//上一个节点
        volatile Node next;				//下一个节点
        volatile Thread thread;		//Node所在的线程
        Node nextWaiter;
    }
    private transient volatile Node head;	//头节点
    private transient volatile Node tail;	//尾节点
    private volatile int state;	//锁状态
```
可以看出AbstractQueuedSynchronizer中维护了一个由Node构成的双向链表，指明了head和tail节点，并且通过state来管理锁的状态，当state=0的时候代表锁未被持有，当state>0时代表锁已被持有。Node类中的prev, next以及AQS的head, tail, state对象都是volatile标示的，方便进行CAS操作。

## 4.ReentrantLock获取锁的源码分析
下图简单展示了ReentrantLock在执行lock方法时的调用过程。

<img src="https://github.com/Sinliede/Sinliede.github.io/blob/master/static/img/AQS-1.png?raw=true" width="50%" height="50%" />

sync对象在lock()方法中调用AbstractQueuedSynchronizer的acquire()方法。代码如下

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```
acquire方法会首先调用tryAcquire()方法，尝试获取锁，返回true代表成功获取锁。tryAcquire()方法在AQS的实现类，也就是ReentrantLock中的Sync类中实现。

```java
protected final boolean tryAcquire(int acquires) {
    //获得当前线程的线程对象
    final Thread current = Thread.currentThread();
    int c = getState();
    //锁是否被持有
    if (c == 0) {
        //队列中是否有线程在等待获取锁
        if (!hasQueuedPredecessors() &&
            //CAS竞争获取锁
            compareAndSetState(0, acquires)) {
            //标记当前线程持有锁
            setExclusiveOwnerThread(current);
            //获取锁成功
            return true;
        }
    }
    //当前线程是否持有锁
    else if (current == getExclusiveOwnerThread()) {
        //锁重入，当前线程锁持有次数加1
        int nextc = c + acquires;
        //锁重入次数是否超出int最大值，超出报异常
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        //设置state
        setState(nextc);
        //持有锁成功
        return true;
    }
    return false;
}
```
上方代码是tryAcquire方法在FairSync类中的实现，FairSync顾名思义就是公平锁竞争。此处有两个细节。

1.判断nextc<0，当c等于int的最大值时，c+1就是int值的最小值，也就是负2的31次方减1，继续递增会使state再次等于0，而state=0代表锁未被持有，其他线程就可以持有锁，出现这种情况就会发生多个线程同时持有锁的情况，这和锁的目标是相悖的。

2.hasQueuedPredecessors方法判断是否有线程在等待获取锁，之后再讨论这个方法
下图展示了tryAcquire方法的调用过程

<img src="https://raw.githubusercontent.com/Sinliede/Sinliede.github.io/master/static/img/AQS-2.png" />

当tryAcquire返回false，也就是尝试获取锁失败的时候，执行acquireQueued(addWaiter(Node.EXCLUSIVE), arg))，首先来看addWaiter方法，也就是新增waiter节点。

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    //新引用指向当前的尾节点
    Node pred = tail;
    //尾节点tail不为null
    if (pred != null) {
        node.prev = pred;
        //CAS尝试将新节点node设置为尾节点tail
        if (compareAndSetTail(pred, node)) {
            //CAS成功，node节点已经是tail节点，将之前的尾节点的next指针指向当前的尾节点node节点
            pred.next = node;
            return node;
        }
    }
    //尾节点tail为null或者CAS竞争尾节点失败，enq入队
    enq(node);
    return node;
}
```
注意判断tail!=null的含义，tail==null会出现在以下可能的几种情况：
1.AQS初始化完成，没有任何线程获取过锁。
2.AQS初始化完成，有线程获取过锁，但通过tryAcquire方法直接获取到了锁。
这两种情况下，AQS的head节点和tail节点都是null。看下enq(node)方法

```java
private Node enq(final Node node) {
    //无限循环
    for (;;) {
        Node t = tail;
        //尾结点为null，上面讨论了tail为null那么head节点也一定为null
        if (t == null) { // Must initialize
            //CAS竞争设置新的头节点
            if (compareAndSetHead(new Node()))
                //CAS竞争头结点成功，让尾节点=头结点，此时头、尾节点都是空节点
                tail = head;
            } else {
            node.prev = t;
            //CAS尝试设置尾节点tail
            if (compareAndSetTail(t, node)) {
                //CAS成功，之前的尾节点，也就是现在的到时第二个节点的next指向当前尾节点node
                t.next = node;
                //返回已经入队的节点，结束循环
                return t;
            }
        }
    }
}
```
联系之前enq方法调用前AQS中node队列的状态，我们可以知道AQS中node队列的生成过程。下放的流程图中简化了第一次在addWaiter方法中竞争成为tail的逻辑，因为在enq方法中这段逻辑已经存在。注意当AQS的head和tail节点被初始化时，他们是没有封装Thread对象的

<img src="https://raw.githubusercontent.com/Sinliede/Sinliede.github.io/master/static/img/AQS-3.png" />

通过addWaiter方法，当前线程已经封装程node对象进入了等待队列。接下来执行acquireQueued方法。

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            //node的prev节点
            final Node p = node.predecessor();
            //如果node的prev节点是头节点，则尝试获取锁，
            if (p == head && tryAcquire(arg)) {
                //成功获取锁,设置头节点为当前节点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                //返回当前线程是否需要被中断
                return interrupted;
            }
            //如果node的prev节点不是头结点或者prev是头节点但node竞争锁失败,判断当前线程是否需要挂起
            if (shouldParkAfterFailedAcquire(p, node) &&
                //线程需要挂起，挂起当前线程。
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            //获取锁过程中发生异常，node节点出队
            cancelAcquire(node);
    }
}

private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```
需要注意以下几个细节：
1.在执行parkAndCheckInterrupt方法，LockSupport.park(this)使得当前线程挂起，不会返回。
2.在node的prev节点是head节点的时候执行了tryAcquire方法，这是因为在非公平锁NonFair的情况下，线程不会判断当前node队列中是否有线程在等待，而会直接tryAcquire尝试获取锁，这就会形成锁竞争，因此这里也使用tryAcquire方法，如果失败则继续等待。
3.如果成功获取锁，那么当前线程node成为head节点；因此当队列不为空的时候，head节点有封装的thread和未封装thread两种状态，这里回看下FairSync中tryAcquire时判断是否有线程节点在等待的方法hasQueuedPredecessors：

```java
public final boolean hasQueuedPredecessors() {
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
    ((s = h.next) == null || s.thread != Thread.currentThread());
}
```
这里我们看下在什么状态下队列没有thread在等待获取锁。
1.之前我们讨论了head和tail节点的状态，当head==tail节点时有两种可能,head==tail==null或者head==tail==new Node()。也就是队列中没有任何封装了thread的node节点，队列为空，因此head!=tail是队列有thread在等待的必要条件。
2.head.next==null或者head.next.thread!=Thread.currentThread()，首先第一个判断条件已经决定了node队列不是初始化的head==tail==new Node()状态，那么就有两种可能，一种是head.next==null，说明队列中的node经过排队获取锁，现在只有head节点在等待；第二种可能：head和next节点在初始化后，后序已经有thread在等待。注意FairSync在执行hasQueuedPredecessors前判断了state==0,说明当前的head节点已经释放锁，即将唤醒head.next节点，因此如果head.next节点的线程就是当前线程，说明可以执行CAS获取锁，当前线程不需要排队，否则当前线程需要排到队尾。

再看下判断当前线程是否需要挂起的shouldParkAfterFailedAcquire方法

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    //当前节点的prev节点是等待状态，那么当前节点应当等待（挂起）
    if (ws == Node.SIGNAL)
        return true;
    //prev节点的waitStatus>0，查看Node类中的waitStatus的各种状态,waitStatus只能是CANCELLED=1,那么当前节点的prev节点取消了排队，向前寻找未取消排队的节点，并链接上，node链表就去除掉了取消排队的节点
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        //将prev节点的waitStatus设置为SIGNAL等待状态，acquireQueued方法中的循环继续，再次进行shouldParkAfterFailedAcquire判断，prev节点的waitStatus状态就是SIGNAL状态，需要挂起，这段代码用来检查前置节点的状态
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```
下图是acquireQueued方法的简略执行过程

<img src="https://raw.githubusercontent.com/Sinliede/Sinliede.github.io/master/static/img/AQS-4.png" />

回到文章开始，ReentrantLock.lock()获取锁，如果获取到则继续执行被锁的代码块，如果没有获取到则进去node队列等待获取锁，当前线程在lock()中挂起，等待被唤起；获取到锁的线程执行被锁的代码块，然后执行ReentrantLock.unlock()方法。unlock方法代码如下

```java
public void unlock() {
    //因为当前线程可能不止一次获取锁，因此每次解锁都释放1
    sync.release(1);
}

public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        //将当前持有锁的线程置为null
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}

private void unparkSuccessor(Node node) {
    //node节点是head节点
    int ws = node.waitStatus;
    //将head节点的waitStatus设置为0
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    Node s = node.next;
    //如果head.next==null代表队列为空，head.next.waitStatus>0代表head.next节点取消了等待
    if (s == null || s.waitStatus > 0) {
        s = null;
        //从队尾开始向前寻找到第一个waitStatus<=0也就是在等待的节点，这里包含waitStatus==0是考虑到该节点没有next节点或者其next节点尚未设置其waitStatus=Node.SIGNAL，参见shouldParkAfterFailedAcquire方法
    for (Node t = tail; t != null && t != node; t = t.prev)
        if (t.waitStatus <= 0)
            s = t;
    }
    //如果寻找到了等待节点，唤起该节点的线程，该节点继续执行accquireQueued方法中的循环来获取锁
    if (s != null)
        LockSupport.unpark(s.thread);
}
```
unlock()方法调用sync.release(1)，因为ReentrantLock是可重入的锁，当前线程可以多次获取锁，因此这里每次unlock释放1，直到state=0,锁被完全释放，执行unparkSuccessor(head)方法唤起head的next节点。注意在判断是否执行unparkSuccessor方法的判断条件中有head.waitStatus != 0，考量head节点waitStatus的可能情况:

1.队列刚刚初始化，head=tail=new Node(), head.waitStatus==tail.waitStatus==0,此时链表为空，不需要唤起head.next;

2.有节点在挂起等待，那么该节点执行过shouldParkAfterFailedAcquire方法，其prev节点的waitStatus一定是SIGNAL，依次类推，head节点的waitStatus应该是Node.SIGNAL，需要唤起head.next;

3.node队列的节点已经依次获取锁，只剩head节点，因为我们是在唤醒后续线程的unparkSuccessor方法中将head设置为0，因此此时的head.waitStatus应该是Node.SIGNAL，需要唤起head.next

下图是release方法的简易逻辑

<img src="https://raw.githubusercontent.com/Sinliede/Sinliede.github.io/master/static/img/AQS-5.png" />

## 5.小结
通过阅读代码的方式我们了解了下ReentrantLock和AQS在多线程并发过程中加锁和解锁的原理。AQS的锁竞争过程可以大致理解为一个排队过程，就像我们排队在ATM取钱一样。大家都来取钱的时候发现ATM机并没有被占用，那么大家就抢，抢到的先用ATM，抢不到的要排队等。排队的时候要按照规则，不能插队，只能抢当队尾，入队的时候要看看前面的人是不是还想用ATM，不想用的把他踢出去，知道前面一个是想用ATM的人，这样依次排队形成一个队列。为了防止队列中的人等ATM等的太累（线程消耗太多资源），可以先让他睡会儿（挂起），正在用ATM的人用好后叫醒后面还想用ATM的人，把不想用的人踢出队列。这样我们就可以依次来使用ATM（获得锁）了。

Doug Lea大神的代码非常简洁，其中包含了大量的CAS操作以及对各种情况的判断，个人水品有限，如果文章中有谬误，还请留言指正。
