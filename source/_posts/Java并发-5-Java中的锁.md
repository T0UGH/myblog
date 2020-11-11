---
title: '[Java并发][5][Java中的锁]'
date: 2020-11-11 17:21:10
tags:
    - Java
    - 并发
categories:
    - Java并发
---
## 第5章 Java中的锁



### 5.1 Lock接口



**锁**是用来**控制多个线程访问共享资源**的方式，一般来说，一个锁能够防止多个线程同时访问共享资源。



在Lock接口出现之前，Java程序是靠synchronized关键字实现锁功能的，而Java SE 5之后，并发包中新增了**Lock接口**（以及相关实现类）用来实现锁功能，它**提供了与synchronized关键字类似的同步功能**，只是**在使用时需要显式地获取和释放锁**。虽然它缺少了（通过synchronized块或者方法所提供的）隐式获取释放锁的便捷性，但是却**更灵活**，**扩展性更好**。



Lock的使用非常简单

````java
Lock lock = new ReentrantLock();
lock.lock();
try{
    
}finally{
    lock.unlock();
}
````

在finally块中释放锁，目的是**保证在获取到锁之后，最终能够被释放**。



**Lock接口的API如下表所示。**

| 方法名称                                             | 描述                                                         |
| ---------------------------------------------------- | ------------------------------------------------------------ |
| void lock()                                          | **获取锁**：调用该方法当前线程将会获取锁，当锁获取后，从该方法返回 |
| void lockInterruptibly() throws InterruptedException | **可中断地获取锁**：该方法可以响应中断，即在锁的获取中可以中断当前线程 |
| boolean tryLock()                                    | **尝试非阻塞获取锁**：获取到立刻返回ture，获取不到立刻返回false，不会等待 |
| boolean tryLock(long time, TimeUnit unit)            | **超时获取锁**：当前线程在以下三种情况下会返回 1、当前线程在超时时间内获取了锁。2、当前线程在超时时间内被中断。3、超时时间结束，返回false |
| void unlock()                                        | **释放锁**                                                   |
| Condition newCondition()                             | 获取等待通知组件：该组件和当前的锁绑定                       |



### 5.2 队列同步器



**队列同步器**`AbstractQueuedSynchronizer`（以下简称同步器），是用来**构建锁**或者其他同步组件的**基础框架.**



**同步器是实现锁的关键**，在锁的实现中聚合同步器，利用同步器实现锁的语义。可以这样理解二者之间的关系：

- **锁是面向使用者的**，它定义了使用者与锁交互的接口（比如可以允许两个线程并行访问），**隐藏了实现细节**；
- **同步器面向的是锁的实现者**，它**简化了锁的实现方式**，屏蔽了同步状态管理、线程的排队、等待与唤醒等底层操作。
- **锁和同步器很好地隔离了使用者和实现者所需关注的领域。**



在正式介绍同步器之前，我们要先了解**共享锁**和**独占锁**

- 独占锁就是在同一时刻只能有一个线程获取到锁，而其他获取锁的线程只能处于同步队列中等待，只有获取锁的线程释放了锁，后继的线程才能够获取锁
- 而共享锁允许同一时刻多个线程获取到锁



#### 5.2.1 队列同步器的接口与示例



**同步器的设计**是基于**模板方法**这种设计模式的，也就是说，

- **使用者**需要**继承同步器**并**重写指定的方法**，
- 随后**将同步器组合**在自定义同步组件的实现中，并**调用同步器提供的模板方法**，
- 而这些**模板方法将会调用使用者重写的方法**。



下面介绍继承后，我们需要重写的5个方法，如下（不重写也行，同步器里提供了默认的空实现）

| 方法名称                                    | 描述                                                         |
| ------------------------------------------- | ------------------------------------------------------------ |
| protected boolean tryAcquire(int arg)       | 独占式获取同步状态：实现该方法需要查询当前状态并判断同步状态是否符合预期，然后再进行CAS设置同步状态 |
| protected boolean tryRelease(int arg)       | 独占式释放同步状态：等待获取同步状态的线程有机会获取同步状态 |
| protected int tryAcquireShared(int arg)     | 共享式获取同步状态，返回大于等于0的值，表示获取成功，反之，获取失败 |
| protected boolean tryReleaseShared(int arg) | 共享式释放同步状态                                           |
| protected boolean isHeldExclusively()       | 判断当前同步器是否在独占模式下被线程占用                     |



下面再介绍用于**访问或者修改同步状态的方法**，这三个方法同步器已经帮我们实现好了，它们是线程安全的，我们在重写上面5个方法时可能会用到它们来修改同步状态

| 方法名称                              | 描述                                |
| ------------------------------------- | ----------------------------------- |
| getState()                            | 获取当前同步状态                    |
| setState()                            | 设置当前同步状态                    |
| compareAndSet(int expect, int update) | 使用CAS设置当前状态，该方法是原子的 |



下面介绍实现一个锁时，**锁可以调用同步器提供的各种方法**，也就是前文提到的**模版方法**

| 方法名称                                           | 描述                                                         |
| -------------------------------------------------- | ------------------------------------------------------------ |
| void acquire(int arg)                              | 独占式获取同步状态：如果当前线程获取同步状态成功，则由该方法返回，否则将进入同步队列等待 |
| void acquireInterruptibly(int arg)                 | 与acquire方法相同，但是该方法可以响应中断。如果当前线程在同步队列中收到了中断，则抛出InterruptedException并返回 |
| boolean tryAcquireNanos(int arg, long nanos)       | 在acquireInterruptibly的基础上增加了超时限制，如果当前线程在超时时间内没有获得同步状态，则返回false，否则返回true |
| void acqureShared(int arg)                         | 共享式的获取同步状态，如果当前线程未获取到同步状态，将进入同步队列等待 |
| void acquireSharedInterruptibly(int arg)           | 共享式的获取同步状态，并且可以响应中断                       |
| boolean tryAcquireSharedNanos(int arg, long nanos) | 在acquireSharedInterruptibly(int arg)基础上增加了超时限制    |
| boolean release(int arg)                           | 独占式释放同步状态，该方法释放同步状态之后，还会从同步队列中将头结点唤醒 |
| boolean releaseShared(int arg)                     | 共享式的释放同步状态                                         |
| `Collection<Thread>getQueuedThreads()`             | 获取等待在同步队列上的线程集合                               |



下面我们实现一个独占锁，来体会一下同步器的工作原理

````java
package ThreadPoolSample;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.AbstractQueuedSynchronizer;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;

public class Mutex implements Lock {

    //静态内部类来继承AQS，并且重写几个方法
    private static class MutexSynchronizer extends AbstractQueuedSynchronizer{
        
        //假设独占锁有人获取时，state为1
        //没人获取时，state为0
        
        //判断当前是否有线程获取独占锁
        @Override
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        //独占地获取同步状态，获取失败直接返回false
        @Override
        protected boolean tryAcquire(int arg) {
            if(compareAndSetState(0, 1)){
                this.setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        //独占地释放同步状态
        @Override
        protected boolean tryRelease(int arg) {
            if(getState() == 0)
                throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        //返回一个新的条件(暂时没学到)
        Condition newCondition(){
            return new ConditionObject();
        }
    }

    //以聚合的方式使用AQS的子类
    private final MutexSynchronizer synchronizer = new MutexSynchronizer();

    //下面这些方法的实现全部简单的调用synchrnizer的对应方法即可
    @Override
    public void lock() {
        synchronizer.acquire(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        synchronizer.acquireInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return synchronizer.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return synchronizer.tryAcquireNanos(1, unit.toNanos(time));
    }

    @Override
    public void unlock() {
        synchronizer.release(1);
    }

    @Override
    public Condition newCondition() {
        return synchronizer.newCondition();
    }
}

````



由此可以看出，有了AQS之后，我们要实现一个可靠的自定义同步组件变得简单了



#### 5.2.2 队列同步器的实现分析



##### 5.2.2.1 同步队列



**同步器**依赖内部的**同步队列**（一个FIFO双向队列）来**完成同步状态的管理**

- 当前线程获取同步状态**失败时**，同步器会将**当前线程**以及等待状态等信息**构造成为一个节点（Node）**并将其
  **加入同步队列的尾部**，同时会**阻塞当前线程**
- 当**同步状态被释放**时，**会把首节点中的线程唤醒**，使其再次尝试获取同步状态

![](https://i.loli.net/2020/10/07/tMhHBKiOXaesElI.png)

同步器中有一个头节点和一个尾节点，**当新节点入队时，插入到尾节点**。这个操作必须是**线程安全**的。同步器使用了**CAS方法**：`compareAndSetTail(Node expect, Node update)`来**保证**这个操作的**线程安全**。

![](https://i.loli.net/2020/10/07/6FAXf8aSwbT9nu4.png)



而**当某个节点释放锁然后出队时，则不需要保证线程安全**。因为同步器定义了一个规则：当头节点出队时，只有头结点的后继节点有资格获取锁。因此不需要考虑线程安全，因为没有其他节点有资格和头结点的后继节点竞争。

![](https://i.loli.net/2020/10/07/r4ZEX8ziL6gWRUk.png)



##### 5.2.2.2 独占式同步状态获取与释放



我们先给出独占式获取和释放同步状态的步骤，然后分别解释每一步

- 在获取同步状态时，同步器维护一个同步队列，获取状态失败的线程都会被加入到队列中并在队列中进行自旋；
- 移出队列（或停止自旋）的条件是前驱节点为头节点且前驱节点释放了同步状态。
- 在释放同步状态时，同步器调用tryRelease(int arg)方法释放同步状态，然后唤醒头节点的后继节点。



**独占式获取同步状态的流程图**如下

![](https://i.loli.net/2020/10/07/tbZx3rkTzEy4lnQ.png)



该方法的源码如下

````java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
````

- 首先调用`tryAcquire`尝试**非阻塞的获取同步状态**，**成功则直接返回**
- 假如上一步失败，则调用`addWaiter`将这个线程**加入到同步队列中**
- 然后调用`acquireQueued`，这个方法是个死循环，**自旋地获取同步状态**
- `selfInterrupt()`应该是自中断，我也不知道有啥用，先跳过吧



接着我们看看`addWaiter`方法，它用一种线程安全的方式将当前节点加入到同步队列

```java
private Node addWaiter(Node mode) {
    //先初始化一个节点，用当前线程来初始化，mode如果为EXCLUSIZE则这个变量为null，应该没啥用
    Node node = new Node(Thread.currentThread(), mode);
    
    
    // Try the fast path of enq; backup to full enq on failure
    
    //获取尾节点
    Node pred = tail;
    
    //如果尾节点不为空
    if (pred != null) {
        //设置新节点的前驱为尾节点
        node.prev = pred;
        //调用CAS方法设置尾节点为当前节点
        if (compareAndSetTail(pred, node)) {
            //如果成功则设置老尾节点的后继为新节点
            pred.next = node;
            //然后返回。
            return node;
        }
    }
    //假如尾节点为空，则调用enq(node)方法，进一步addWaiter()
    enq(node);
    return node;
}


private Node enq(final Node node) {
    //一个死循环
    for (;;) {
        //首先获取尾节点
        Node t = tail;
        //如果尾节点为空
        if (t == null) { // Must initialize
            //使用CAS设置头结点为一个新节点
            if (compareAndSetHead(new Node()))
                //然后设置尾节点是头结点
                tail = head;
        //如果尾节点不为空
        } else {
            //设置当前节点的前驱是尾节点
            node.prev = t;
            //使用CAS设置尾节点是当前节点
            if (compareAndSetTail(t, node)) {
                //老尾节点的后继设置为当前节点
                t.next = node;
                //返回老尾节点
                return t;
            }
        }
    }
}
```



然后再看看acquireQueued()方法是怎样自旋地获取同步状态的

````java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        //死循环
        for (;;) {
            //获取当前节点的前驱节点
            final Node p = node.predecessor();
            //如果前驱节点是头结点并且获取同步状态成功
            if (p == head && tryAcquire(arg)) {
                //设置头结点为当前节点
                setHead(node);
                //释放老头节点的引用，方便GC
                p.next = null; // help GC
                //没有失败
                failed = false;
                return interrupted;
            }
            //没看懂，先跳过
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        //如果失败了，就取消获取，不重要，也跳过吧
        if (failed)
            cancelAcquire(node);
    }
}
````



最后我们看看独占式地释放同步状态的源码

````java
public final boolean release(int arg) {
    //如果尝试释放同步状态成功
    if (tryRelease(arg)) {
        //获取头结点
        Node h = head;
        //如果头结点不为null并且头结点当前的等待状态不是0（这个就别管了）
        if (h != null && h.waitStatus != 0)
            //唤醒它的后继节点
            unparkSuccessor(h);
        return true;
    }
    return false;
}
````





##### 5.2.2.3 共享式同步状态获取与释放



![](https://i.loli.net/2020/10/07/bH2asvrqASZBuVg.png)

- 左半部分，共享式访问资源时，其他共享式的访问均被允许，而独占式访问被阻塞
- 右半部分是独占式访问资源时，同一时刻其他访问均被阻塞。



下面来看共享式获取同步状态的源码

````java
public final void acquireShared(int arg) {
    //先尝试非阻塞地获取共享式同步状态，如果失败
    if (tryAcquireShared(arg) < 0)
        //就自旋地获取共享式同步状态
        doAcquireShared(arg);
}

//自旋地获取共享式同步状态
private void doAcquireShared(int arg) {
    //先将当前线程加入到同步队列中并且状态是共享态
    final Node node = addWaiter(Node.SHARED);

    boolean failed = true;
    
    try {
        boolean interrupted = false;
    
        //死循环
        for (;;) {
            //获取当前节点的前驱节点
            final Node p = node.predecessor();
            //如果这个前驱节点是头结点
            if (p == head) {
                //尝试获取同步状态
                int r = tryAcquireShared(arg);
                //如果获取成功
                if (r >= 0) {
                    //设置头结点并且传播共享态获取成功这件事
                    setHeadAndPropagate(node, r);
                    //将老头结点的后继设置为null
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            //我也不知道这个地方
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private void setHeadAndPropagate(Node node, int propagate) {
   	//记录一下老头结点     
    Node h = head; // Record old head for check below
    //设置传入的节点为头结点
    setHead(node);
    /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
    //如果传播值大于0或者老头节点为空或者老头节点的等待状态小于0
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        //获取传入节点的后继
        Node s = node.next;
        //如果后继为空或者后继是共享型的
        if (s == null || s.isShared())
            //调用doReleaseShared()来释放同步状态
            doReleaseShared();
    }
}
````



下面是共享式释放同步状态的源码

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```



##### 5.2.2.4 独占式超时获取同步状态



````java
private boolean doAcquireNanos(int arg, long nanosTimeout) throws InterruptedException {
    
    //如果传入的超时纳秒小于0直接返回就行了
    if (nanosTimeout <= 0L)
        return false;
    
    //截至日期
    final long deadline = System.nanoTime() + nanosTimeout;
    //新建一个独占式节点
    final Node node = addWaiter(Node.EXCLUSIVE);
    
    boolean failed = true;
    
    try {
        //死循环
        for (;;) {
            //先获取当前节点的前驱
            final Node p = node.predecessor();
            //如果这个前驱是头结点并且获取同步状态成功
            if (p == head && tryAcquire(arg)) {
                //设置头结点为当前节点
                setHead(node);
                //help GC
                p.next = null; // help GC
                failed = false;
                return true;
            }
            //如果获取失败，则更新nanasTimeout的值
            nanosTimeout = deadline - System.nanoTime();
            
            //如果已经小于0，说明当前已经超时了，则直接返回false
            if (nanosTimeout <= 0L)
                return false;
            
            //如果nanosTimeout小于定于spinForTimeoutThread时，将不会使该线程进行超时等待，而是进入快速的自旋
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            
            //如果线程被设置了中断，则抛出异常来响应这个中断
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
````



##### 5.2.2.5 自定义同步组件——TwinsLock



本节我们再设计一个简单的同步组件来加深理解，需求如下：

- 设计一个同步工具，该工具在同一时刻，只允许至多两个线程同时访问
- 超过两个线程的访问将被阻塞
- 我们将这个同步工具命名为TwinsLock。



设计过程如下

- 首先，**确定访问模式**。TwinsLock能够在同一时刻支持多个线程的访问，这显然是**共享式访问**，因此，要
  求TwinsLock必须重写tryAcquireShared(int args)方法和tryReleaseShared(int args)方法
- 其次，**定义资源数**。TwinsLock在同一时刻允许至多两个线程的同时访问，表明同步资源数为2，这样可以设置初始状态status为2，当一个线程进行获取，status减1，该线程释放，则status加1
- 最后，**组合自定义同步器**。前面的章节提到，自定义同步组件通过组合自定义同步器来完成同步功能，一般情况下自定义同步器会被定义为自定义同步组件的内部类。



````java
package ThreadPoolSample;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.AbstractQueuedSynchronizer;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;

public class TwinsLock implements Lock {

    private Sync sync = new Sync(2);

    static class Sync extends AbstractQueuedSynchronizer{

        public Sync(int count){
            if(count <= 0){
                throw new IllegalArgumentException("count must larger than 0");
            }
            setState(count);
        }

        @Override
        protected int tryAcquireShared(int reduceCount) {
            for(;;){
                int current = getState();
                int newCount = current - reduceCount;
                if(newCount < 0 || compareAndSetState(current, newCount)){
                    return newCount;
                }
            }
        }

        @Override
        protected boolean tryReleaseShared(int returnCount) {
            for(;;){
                int current = getState();
                int newCount = current + returnCount;
                if(compareAndSetState(current, newCount)){
                    return true;
                }
            }
        }

        public Condition newCondition() {
            return new ConditionObject();
        }
    }


    @Override
    public void lock() {
        sync.acquireShared(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return sync.tryAcquireShared(1) > 0;
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(time));
    }

    @Override
    public void unlock() {
        sync.release(1);
    }

    @Override
    public Condition newCondition() {
        return newCondition();
    }
}

````



### 5.3 重入锁(ReentrantLock)



ReentrantLock是**Lock接口**最常用的**实现**之一，ReentrantLock支持重进入：在调用`lock()`方法时，已经获取到锁的线程，能够**再次调用`lock()`方法获取锁而不被阻塞**。



然后解释一下**锁获取的公平性问题**，

- 如果在绝对时间上，**先对锁进行获取的请求一定先被满足，那么这个锁是公平的**，反之，是不公平的。
- **公平的获取锁**，也就是**等待时间最长的线程最优先获取锁**，也可以说锁获取是顺序的。
- ReentrantLock提供了一个构造函数，能够**控制锁是否是公平的**。



#### 5.4.1 实现重进入



**重进入是指任意线程在获取到锁之后能够再次获取该锁而不会被锁所阻塞**，该特性的实现需要**解决以下两个问题**：

1. **线程再次获取锁**：锁需要去识别获取锁的线程是否为当前占据锁的线程，如果是，则再次成功获取。

2. **锁的最终释放**：

   - 线程重复n次获取了锁，随后在第n次释放该锁后，其他线程能够获取到该锁。

   - 锁的最终释放要求锁对于获取进行计数自增，计数表示当前锁被重复获取的次数，而锁被释放时，计数自减，当计数等于0时表示锁已经成功释放。



下面查看一下非公平获取锁的源码

````java
//非公平地获取同步状态
final boolean nonfairTryAcquire(int acquires) {
    //首先获取当前线程
    final Thread current = Thread.currentThread();
    //获取同步状态值
    int c = getState();
    //如果这个值是0，说明当前没有人获取这个锁
    if (c == 0) {
        //用CAS更新同步状态
        if (compareAndSetState(0, acquires)) {
            //如果更新成功，则设置持有锁的线程是当前线程，然后返回true
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //如果这个值不是0并且当前线程是持有锁的线程，也就是发生锁重入了
    else if (current == getExclusiveOwnerThread()) {
        
        //先计算一下新的同步状态值
        int nextc = c + acquires;
        
        //溢出了，所以小于0了
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        
        //设置同步状态
        setState(nextc);
        return true;
    }
    return false;
}
````



然后查看一下释放锁的源码

````java
protected final boolean tryRelease(int releases) {
    //计算新的状态值c
    int c = getState() - releases;
    
    //如果当前线程不是持有锁的线程，讲道理它没资格调用这个方法，所以直接抛出异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    
    //释放标记
    boolean free = false;
    
    //如果新的状态值是0，说明重入次数已经消耗完了，就把锁释放掉
    if (c == 0) {
        //设置释放标记为true
        free = true;
        //设置持有锁的线程为null
        setExclusiveOwnerThread(null);
    }
    //设置同步状态值为新状态值c
    setState(c);
    //返回释放标记，说明释放锁成功或者失败
    return free;
}
````



#### 5.4.2 公平与非公平获取锁的区别



下面查看公平地获取锁的源代码

````java
 protected final boolean tryAcquire(int acquires) {
     //获取当前线程
     final Thread current = Thread.currentThread();
     //获取状态值
     int c = getState();
     //如果同步状态值为0，说明此时没有线程持有这个锁
     if (c == 0) {
         //注意这个判断！！！
         //如果没有前驱节点（也就是这个节点是等待最久的节点）就CAS更新同步状态
         if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
             //更新成功则设置持有锁的线程为当前线程
             setExclusiveOwnerThread(current);
             //返回true
             return true;
         }
     }
    //如果同步状态值不为0并且当前线程就是持有锁的线程
     else if (current == getExclusiveOwnerThread()) {
         //算一下新的同步状态
         int nextc = c + acquires;
         //处理一下溢出
         if (nextc < 0)
             throw new Error("Maximum lock count exceeded");
         //设置同步状态为新状态
         setState(nextc);
         //返回true，表示获取到了锁
         return true;
     }
     return false;
 }
````



该方法与nonfairTryAcquire(int acquires)比较，唯一不同的位置为判断条件多了hasQueuedPredecessors()方法，即加入了同步队列中当前节点是否有前驱节点的判断



**对比公平锁与非公平锁**

- 公平锁可以保证不引起饥饿，但是这意味着大量线程切换，极为消耗资源
- 非公平锁可以减少线程切换，但是会引起饥饿
- 如果不特殊声明，ReentrantLock默认为非公平锁



### 5.4 读写锁



**读写锁**在**同一时刻可以允许多个读线程访问**，但是在**写线程访问**时，所有的读线程和其他写线程均被**阻塞**。



在使用读写锁时，只需要在读操作时获取读锁，写操作时获取写锁即可。当写锁被获取到时，后续（非当前写
操作线程）的读写操作都会被阻塞，写锁释放之后，所有操作继续执行，



**一般情况下，读写锁的性能都会比排它锁好**，因为大多数场景读是多于写的。在读多于写的情况下，读写锁能够提供比排它锁更好的并发性和吞吐量。Java并发包提供读写锁的实现是**ReentrantReadWriteLock**，**它提供的特性如下**：

- **公平性选择**：用户可以选择使用公平锁还是非公平锁
- **重进入**：支持重进入，读线程在获取读锁之后，能再次获取读锁。写线程在获取写锁之后能再次获取写锁，同时也可以获取读锁。
- **锁降级**：在某些特定情况下，写锁可以降级为读锁



#### 5.4.1 读写锁的接口与示例



下面通过一个缓存的示例来介绍读写锁的使用方式

````java
package ThreadPoolSample;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class Cache {
    static Map<String, Object> map = new HashMap<>();
    static ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    static Lock r = rwl.readLock();
    static Lock w = rwl.writeLock();
    public static final Object get(String key){
        r.lock();
        try{
            return map.get(key);
        }finally {
            r.unlock();
        }
    }
    public static final Object put(String key, Object value){
        w.lock();
        try{
            return map.put(key, value);
        } finally {
            w.unlock();
        }
    }
    public static final void clear(){
        w.lock();
        try{
            map.clear();
        }finally {
            w.unlock();
        }
    }
}

````



上述示例中，Cache组合一个非线程安全的HashMap作为缓存的实现，同时使用读写锁的读锁和写锁来保证Cache是线程安全的。

- 在**读操作**get(String key)方法中，需要**获取读锁**，这使得并发访问该方法时不会被阻塞。
- **写操作**put(String key,Object value)方法和clear()方法，在更新HashMap时必须提前**获取写锁**，当获取写锁后，其他线程对于读锁和写锁的获取均被阻塞，而只有写锁被释放之后，其他读写操作才能继续。
- Cache使用读写锁**提升读操作的并发性**，**也保证每次写操作对所有的读写操作的可见性**，同时简化了编程方式。



#### 5.4.2 读写锁的实现分析



接下来分析ReentrantReadWriteLock的实现，主要包括：

1. **读写状态的设计**
2. **写锁的获取与释放**
3. **读锁的获取与释放**
4. **锁降级**（以下没有特别说明读写锁均可认为是ReentrantReadWriteLock）



##### 5.4.2.1 读写状态的设计



读写锁的自定义同步器需要**在同步状态（一个整型变量）上维护多个读线程和一个写线程的状态**，使得该状态的设计成为读写锁实现的关键。



如果在一个整型变量上维护多种状态，就一定需要“**按位切割使用**”这个变量，读写锁将变量切分成了两个部分，**高16位表示读**，**低16位表示写**

![](https://i.loli.net/2020/10/08/iSanmTZoyXFub7M.png)



**读写锁是如何迅速确定读和写各自的状态呢**？

- 答案是通过**位运算**。假设当前同步状态值为S，写状态等于S&0x0000FFFF（将高16位全部抹去），读状态等于S>>>16（无符号补0右移16位）。
- 当写状态增加1时，等于S+1，当读状态增加1时，等于S+(1<<16)，也就是S+0x00010000。



##### 5.4.2.2 写锁的获取与释放



**写锁**是一个**支持重进入**的**排它锁**。

- 如果当前线程已经获取了写锁，则增加写状态。
- 如果当前线程在获取写锁时，读锁已经被获取（读状态不为0）或者该线程不是已经获取写锁的线程，则当前线程进入等待状态



获取写锁的代码如代码清单5-17所示。

```java
//获取写锁的逻辑
protected final boolean tryAcquire(int acquires) {
    //获取当前线程
    Thread current = Thread.currentThread();
    //获取同步状态
    int c = getState();
    //从同步状态中提取低16位，也就是提取写状态
    int w = exclusiveCount(c);
    //如果同步状态不是0，也就是读锁或者写锁有线程持有
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        //如果同步状态不为0但是写状态为0，则说明当前有线程持有读锁
        //如果存在读锁或者存在写锁但是当前线程并不是写锁持有者时
        if (w == 0 || current != getExclusiveOwnerThread())
            //返回false，获取失败
            return false;
        
        //处理一下溢出情况
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        
        //执行到这里，就说明：存在写锁且写锁的拥有者是当前线程，此时更新一下同步状态即可
        // Reentrant acquire
        setState(c + acquires);
        //返回true，获取成功
        return true;
    }
    
    //同步状态是0时，直接获取写锁，获取失败就返回false
    if (writerShouldBlock() || !compareAndSetState(c, c + acquires))
        return false;
    //获取成功就设置当前线程为写锁持有者并返回true
    setExclusiveOwnerThread(current);
    return true;
}

//从同步状态中提取写状态的函数
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

该方法保证了如果存在读锁则写锁不能被获取，原因是：如果允许读锁在已被获取的情况下对写锁的获取，那么正在运行的其他读线程就无法感知到当前写线程的操作。



##### 5.4.2.3 读锁的获取与释放



**读锁**是一个**支持重进入**的**共享锁**，它能够**被多个线程同时获取**，在没有其他写线程访问（或者写状态为0）时，读锁总会被成功地获取，而所做的也只是（线程安全的）增加读状态。



下面查看获取读锁的删减版源码

````java
protected final int tryAcquireShared(int unused) {
	//死循环
    for (;;) {
        //获取同步状态
        int c = getState();
        //计算新的同步状态（读状态+1）
        int nextc = c + (1 << 16);
        //处理溢出
        if (nextc < c)
        	throw new Error("Maximum lock count exceeded");
        //如果写状态不为0并且写锁的持有者不是当前线程
        if (exclusiveCount(c) != 0 && owner != Thread.currentThread())
            //返回-1代表获取失败
        	return -1;
        //如果写锁持有者是当前线程或者写状态为0，则CAS更新同步状态，并返回1，代表获取成功
        if (compareAndSetState(c, nextc))
        	return 1;
    }
}
````



##### 5.4.2.4 锁降级



**锁降级**是指把持住（当前拥有的）写锁，再获取到读锁，随后释放（先前拥有的）写锁的过程。



下面是一个锁降级的例子

```java
public void processData() {
    readLock.lock();
    if (!update) {
        // 必须先释放读锁
        readLock.unlock();
        // 锁降级从写锁获取到开始
        writeLock.lock();
        try {
        	if (!update) {
                // 准备数据的流程（略）
                update = true;
            }
            readLock.lock();
        } finally {
        	writeLock.unlock();
        }
    	// 锁降级完成，写锁降级为读锁
    }
    try {
    // 使用数据的流程（略）
    } finally {
    	readLock.unlock();
    }
}
```



### 5.5 LockSupport工具



LockSupport定义了一组的**公共静态方法**，这些方法提供了**最基本的线程阻塞和唤醒功能**，而LockSupport也成为构建同步组件的基础工具。

- LockSupport定义了一组以**park开头的方法**用来**阻塞当前线程**
- 以及**unpark(Thread thread)方法**来**唤醒一个被阻塞的线程**。



| 方法名称                      | 描述                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| void park()                   | **阻塞当前线程**，如果调用unpark(Thread thread)方法或者当前线程被中断，才能从park()方法返回 |
| void parkNanos(long nanos)    | **阻塞当前线程**，最长不超过nanos纳秒，返回条件在`park()`的基础上增加了超时返回 |
| void parkUntil(long deadline) | **阻塞当前线程**，直到`deadline`时间                         |
| void unpark(Thread thread)    | **唤醒处于阻塞状态的线程**`thread`                           |



### 5.6 Condition接口



**任意一个Java对象，都拥有一组监视器方法**（定义在java.lang.Object上），主要包括wait()、wait(long timeout)、notify()以及notifyAll()方法，这些方法与synchronized同步关键字配合，可以实现等待/通知模式。



**Condition接口**也**提供了类似Object的监视器方法**，**与Lock配合可以实现等待/通知模式**，但是这两者在使用方式以及功能特性上还是有差别的。



#### 5.6.1 Condition接口与示例



Condition定义了等待/通知两种类型的方法，**当前线程调用这些方法时，需要提前获取到Condition对象关联的锁。**Condition对象是由Lock对象（调用Lock对象的newCondition()方法）创建出来的，换句话说，**Condition是依赖Lock对象的。**



`Condition`的使用方式如下

````java
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();
public void conditionWait() throws InterruptedException {
    lock.lock();
    try {
    	condition.await();
    } finally {
    	lock.unlock();
    }
} 
public void conditionSignal() throws InterruptedException {
    lock.lock();
    try {
    	condition.signal();
    } finally {
    	lock.unlock();
    }
}
````



如示例所示，一般都会将Condition对象作为成员变量。

- 当**调用await()方法后**，**当前线程会释放锁并在此等待**，
- **而其他线程调用Condition对象的signal()方法**，通知当前线程后，**当前线程才从await()方法返回，并且在返回前已经获取了锁。**



Condition定义的部分方法如下表

![](https://i.loli.net/2020/10/08/rxNGbfUFA9IgnMe.png)



获取一个Condition必须通过Lock的newCondition()方法



下面通过一个有界队列的示例来深入了解Condition的使用方式。有界队列是一种特殊的队列，当队列为空时，队列的获取操作将会阻塞获取线程，直到队列中有新增元素，当队列已满时，队列的插入操作将会阻塞插入线程，直到队列出现“空位”



````java
package ThreadPoolSample;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class BoundedQueue<T> {
    private Object[] items;
    private int addIndex, removeIndex, count;
    private Lock lock = new ReentrantLock();
    private Condition notEmpty = lock.newCondition();
    private Condition notFull = lock.newCondition();
    public BoundedQueue(int size){
        items = new Object[size];
    }
    public void add(T t) throws InterruptedException{
        lock.lock();
        try{
            //满了就在notFull条件上释放锁然后等待
            while(count == items.length)
                notFull.await();
            //等到了之后继续执行下面的入队操作
            items[addIndex] = t;
            if(++addIndex == items.length)
                addIndex = 0;
            ++count;
            //发出一个当前非空的通知，因为现在明显非空
            notEmpty.signal();
        }finally {
            //在finally中解锁，这很严谨
            lock.unlock();
        }
    }

    public T remove() throws InterruptedException{
        lock.lock();
        try{
            //空了就在notEmpty条件上释放锁然后等待
            while(count == 0)
                notEmpty.await();
            //然后执行出队操作
            Object x = items[removeIndex];
            if(++removeIndex == items.length)
                removeIndex = 0;
            --count;
            //发出一个当前非满的通知，因为现在明显非满
            notFull.signal();
            return (T)x;
        }finally {
            lock.unlock();
        }
    }
}

````





#### 5.6.2 Condition的实现分析



**每个Condition对象都包含着一个队列**（以下称为等待队列），该队列是Condition对象实现等待/通知功能的关键。



下面将**分析Condition的实现**，主要包括：等待队列、等待和通知，



##### 5.6.2.1 等待队列



等待队列是一个**FIFO的队列**，**在队列中的每个节点都包含了一个线程引用**，该线程就是在Condition对象上等待的线程，**如果一个线程调用了Condition.await()方法，那么该线程将会释放锁、构造成节点加入等待队列并进入等待状态。**



一个Condition包含一个等待队列，Condition拥有**首节点**（firstWaiter）和**尾节点**（lastWaiter）。

当前线程调用Condition.await()方法，将会以当前线程构造节点，并将节点从尾部加入等待队列

![](https://i.loli.net/2020/10/08/OZtrQqAFkKyTfbJ.png)

如图所示，Condition拥有首尾节点的引用，而新增节点只需要将原有的尾节点nextWaiter指向它，并且更新尾节点即可。上述节点引用更新的过程并没有使用CAS保证，原因在于**调用await()方法的线程必定是获取了锁的线程，也就是说该过程是由锁来保证线程安全的。**



一个对象拥有一个同步队列和等待队列，而并发包中的Lock（更确切地说是同步器）拥有一个同步队列和多个等待队列

![](https://i.loli.net/2020/10/08/g9fQXaBIhFKUAL2.png)

##### 5.6.2.2 等待



略



##### 5.6.2.3 通知



略