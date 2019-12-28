---
title: ReentrantLock实现原理
date: 2019-12-28 10:29:21
tags: java
---

------
###  同步锁
<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用 java关键字synchronize 来做同步处理时，锁的获取和释放都是隐式的，实现的原理是通过编译后加上不同的机器指令来实现。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ReentrantLock 就是一个普通的java类，它是基于 AQS(AbstractQueuedSynchronizer)来实现同步锁。AQS 是 Java 并发包里实现锁、同步的一个重要的基础框架。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ReentrantLock是一个重入锁：一个线程获得了锁之后仍然可以反复的加锁，不会出现自己阻塞自己的情况。
<br>
<br>
### ReentrantLock
<br>
ReentrantLock 分为公平锁和非公平锁，可以通过构造方法来指定具体类型：

```
//默认非公平锁
public ReentrantLock() {
    sync = new NonfairSync();
}

//指定锁的类型   
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

ReentrantLock锁的使用:

```
Lock lock = new ReentrantLock();

try{
    lock.lock();
    //do bussiness        
}catch(InterruptedException e){
    e.printStackTrace();
}finally{
    lock.unlock();
}
``` 
默认一般使用非公平锁，它的效率和吞吐量都比公平锁高的多。
<br>
<br>
### 公平锁获取锁
<br>
首先看下获取公平锁的过程：
```
public void lock() {
    sync.lock();
}
``` 
sync的lock方法是一个抽象方法，具体是由其子类(FairSync)来实现的
```
// java.util.concurrent.locks.ReentrantLock.FairSync    
final void lock() {
    acquire(1);
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

```

第一步是尝试获取锁(tryAcquire(arg)), 如果该方法返回了True，则说明当前线程获取锁成功，就不用往后执行了；如果获取失败，就需要加入到等待队列中。这个也是由其子类(FairSync)实现：


``` 
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    // AQS中的state等于0表示目前没有其他线程获得锁，当前线程就可以尝试获取锁
    if (c == 0) {
        // AQS的队列中中是否有其他线程，如果有则不会尝试获取锁
        if (!hasQueuedPredecessors() &&
            //AQS中的state修改为1，成功即获取锁，
            compareAndSetState(0, acquires)) {
            // 获取成功则将当前线程置为获得锁的独占线程
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 锁已经被获取，判断获取锁的线程是否为当前线程
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        // 同一个锁最多能重入Integer.MAX_VALUE次，也就是2147483647
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

- 首先会判断 AQS 中的 state 是否等于 0，0 表示目前没有其他线程获得锁，当前线程就可以尝试获取锁。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; **注意:** 尝试之前会利用 hasQueuedPredecessors() 方法来判断 AQS 的队列中中是否有其他线程，如果有则不会尝试获取锁(这是公平锁特有的情况)。

- 如果队列中没有线程就利用 CAS 来将 AQS 中的 state 修改为1，也就是获取锁，获取成功则将当前线程置为获得锁的独占线程( setExclusiveOwnerThread(current))。

- 如果 state 大于 0 时，说明锁已经被获取了，则需要判断获取锁的线程是否为当前线程( ReentrantLock 支持重入)，是则需要将 state+1，并将值更新。
<br>
<br>
### 写入队列
<br>
如果 tryAcquire(arg) 获取锁失败，则需要用 addWaiter(Node.EXCLUSIVE) 将当前线程写入等待队列中。

写入之前需要将当前线程包装为一个 Node 对象( addWaiter(Node.EXCLUSIVE))。


```
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    // Pred指针指向尾节点Tail
    Node pred = tail;
    if (pred != null) {
        // 将New中Node的Prev指针指向Pred
        node.prev = pred;
        // 通过compareAndSetTail方法，完成尾节点的设置
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

首先判断队列是否为空，不为空时则将封装好的 Node 利用 CAS 写入队尾，如果Pred指针是Null（说明等待队列中没有元素）,或者当前Pred指针和Tail指向的位置不同（说明被别的线程已经修改）,就需要调用 enq(node) 来写入了。

```
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
 这个处理逻辑就相当于 自旋加上 CAS 保证一定能写入队列。
<br>
<br>
### 挂起等待线程
<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;上文解释了addWaiter方法，这个方法其实就是把对应的线程以Node的数据结构形式加入到双端队列里，返回的是一个包含该线程的Node。而这个Node会作为参数，进入到acquireQueued方法中。acquireQueued方法可以对排队中的线程进行“获锁”操作。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;总的来说，一个线程获取锁失败了，被放入等待队列，acquireQueued会把放入队列中的线程不断去获取锁，直到获取成功或者不再需要获取（中断）。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;写入队列之后需要将当前线程挂起(利用 acquireQueued(addWaiter(Node.EXCLUSIVE),arg))：

```
final boolean acquireQueued(final Node node, int arg) {
    // 标记是否成功拿到资源
    boolean failed = true;
    try {
        // 标记等待过程中是否中断过
        boolean interrupted = false;
        // 开始自旋，要么获取锁，要么中断
        for (;;) {
            // 获取当前节点的前驱节点
            final Node p = node.predecessor();
            // 如果p是头结点，说明当前节点在真实数据队列的首部，就尝试获取锁（头结点是虚节点）
            if (p == head && tryAcquire(arg)) {
               // 获取锁成功，头指针移动到当前node
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 说明p为头节点且当前没有获取到锁（可能是非公平锁被抢占了）或者是p不为头结点，这个时候就要判断当前node是否要被阻塞（被阻塞条件：前驱节点的waitStatus为-1），防止无限循环浪费资源。
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
- 首先会根据 node.predecessor() 获取到上一个节点是否为头节点，如果是则尝试获取一次锁，获取成功就万事大吉了。

- 如果不是头节点，或者获取锁失败，则会根据上一个节点的 waitStatus 状态来处理( shouldParkAfterFailedAcquire(p,node))。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; waitStatus 用于记录当前节点的状态，如节点取消、节点等待等。

- shouldParkAfterFailedAcquire(p,node) 返回当前线程是否需要挂起，如果需要则调用 parkAndCheckInterrupt()：


```
// java.util.concurrent.locks.AbstractQueuedSynchronizer
// 靠前驱节点判断当前线程是否应该被阻塞    
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    // 说明头结点处于唤醒状态
    if (ws == Node.SIGNAL)
        return true;
    // 通过枚举值我们知道waitStatus>0是取消状态
    if (ws > 0) {
        do {
             // 循环向前查找取消节点，把取消节点从队列中剔除
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // 设置前任节点等待状态为SIGNAL
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

parkAndCheckInterrupt主要用于挂起当前线程，阻塞调用栈，返回当前线程的中断状态。

```
// java.util.concurrent.locks.AbstractQueuedSynchronizer

private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

它是利用 LockSupport 的 part 方法来挂起当前线程的，直到被唤醒。
<br>
<br>
### 非公平锁获取锁
<br>
公平锁与非公平锁的差异主要在获取锁：<br>
- 公平锁就相当于买票，后来的人需要排到队尾依次买票，**不能插队**。<br>
- 而非公平锁则没有这些规则，是**抢占模式**，每来一个人不会去管队列如何，直接尝试获取锁。        
公平锁：

```
final void lock() {
    acquire(1);
}
```

非公平锁：


```
final void lock() {
    // 直接尝试获取锁
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

还要一个重要的区别是在尝试获取锁时 tryAcquire(arg)，非公平锁是不需要判断队列中是否还有其他线程，也是直接尝试获取锁：

```
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 没有通过!hasQueuedPredecessors()判断队列里是否有其他线程
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
<br>
<br>
### 释放锁
<br>
公平锁和非公平锁的释放流程都是一样的：

```
// java.util.concurrent.locks.ReentrantLock
public void unlock() {
    sync.release(1);
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer
public final boolean release(int arg) {
    // 如果返回true，说明该锁没有被任何线程持有
    if (tryRelease(arg)) {
        Node h = head;
        // 头结点不为空并且头结点的waitStatus不是初始化节点情况，解除线程挂起状态
        if (h != null && h.waitStatus != 0)
            //唤起被挂起的线程
            unparkSuccessor(h);
        return true;
    }
    return false;
}           

// ReentrantLock内部类sync实现的释放方法
protected final boolean tryRelease(int releases) {
    // 减少可重入次数
    int c = getState() - releases;
    // 当前线程不是持有锁的线程，抛出异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    // 如果持有线程全部释放，将当前独占锁所有线程设置为null，并更新state
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```
<br>
首先判断当前线程是否是获取锁的  线程，然后将AQS的state计数减1，由于是可重入锁所以需要将state减到0才认为完全释放锁，有多少次加锁操作就要有多少次解锁操作。释放之后要通过unparkSuccessor(h)来唤醒被挂起的线程。
<br>
<br>
### 总结
由于公平锁需要关心队列的情况，得按照队列里的先后顺序来获取锁，这样会造成大量的线程上下文切换，而非公平锁则没有这个限制。所以也就能解释非公平锁的效率会被公平锁更高。


    