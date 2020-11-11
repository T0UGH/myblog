---
title: '[Java并发][4][Java并发编程基础]'
date: 2020-11-11 17:20:40
tags:
    - Java
    - 并发
categories:
    - Java并发
---
## 第4章 Java并发编程基础



**Java从诞生开始就明智地选择了内置对多线程的支持**，这使得Java语言相比同一时期的其他语言具有明显的优势。

- 线程作为操作系统调度的最小单元，多个线程能够同时执行，这将显著提升程序性能，在多核环境中表现得更加明显。
- 但是，过多地创建线程和对线程的不当管理也容易造成问题。



### 4.1 线程简介



#### 4.1.1 什么是线程



**现代操作系统调度的最小单元是线程**，也叫轻量级进程（Light Weight Process）

- 在一个进程里可以创建多个线程，这些线程都**拥有各自的计数器、堆栈和局部变量等属性**，并且**能够访问共享的内存变量**。
- 处理器在这些线程上**高速切换**，让使用者感觉到这些线程在同时执行。



一个Java程序从main()方法开始执行，然后按照既定的代码逻辑执行，看似没有其他线程参与，**但实际上Java程序天生就是多线程程序**，因为执行main()方法的是一个名称为main的线程。



````java
public class MultiThread{
    public static void main(String[] args) {
        // 获取Java线程管理MXBean
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        // 不需要获取同步的monitor和synchronizer信息，仅获取线程和线程堆栈信息
        ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(false, false);
        // 遍历线程信息，仅打印线程ID和线程名称信息
        for (ThreadInfo threadInfo : threadInfos) {
            System.out.println("[" + threadInfo.getThreadId() + "] " + threadInfo.
            getThreadName());
        }
    }
}
````



可以看到，一个Java程序的运行不仅仅是main()方法的运行，而是main线程和多个其他线程的同时运行。

```
[6] Monitor Ctrl-Break
[5] Attach Listener
[4] Signal Dispatcher
[3] Finalizer
[2] Reference Handler
[1] main
```



#### 4.1.2 为什么要使用多线程



正确使用多线程，总是能够给开发人员带来显著的好处，而使用多线程的原因主要有以下几点。

1. **更多的处理器核心**：现在大多数计算机都比以往更加擅长并行计算，而处理器性能的提升方式，也从更高的主频向更多的核心发展。
2. **更快的响应时间**：有时我们会编写一些较为复杂的代码（这里的复杂不是说复杂的算法，而是复杂的业务逻
   辑）。可以使用多线程技术，即将数据一致性不强的操作派发给其他线程处理（也可以使用消息队列），如生成订单快照、发送邮件等。这样做的好处是响应用户请求的线程能够尽可能快地处理完成，缩短了响应时间，提升了用户体验。
3. **更好的编程模型**：Java为多线程编程提供了良好、考究并且一致的编程模型，使开发人员能够更加专注于问
   题的解决



#### 4.1.3 线程优先级



现代操作系统基本采用**时分的形式**调度运行的线程，操作系统会分出一个个时间片，线程会分配到若干时间片，当线程的时间片用完了就会发生线程调度，并等待着下次分配。线程分配到的时间片多少也就决定了线程使用处理器资源的多少，而**线程优先级就是决定线程需要多或者少分配一些处理器资源的线程属性**。



在Java线程中，通过一个整型成员变量priority来控制优先级，优先级的范围从1~10，在线程构建的时候可以通过setPriority(int)方法来修改优先级，默认优先级是5，**优先级高的线程分配时间片的数量要多于优先级低的线程**。



在不同的JVM以及操作系统上，线程规划会存在差异，有些操作系统甚至会忽略对线程优先级的设定。因此，线程优先级不能作为程序正确性的依赖，因为**有的操作系统可以完全不用理会Java线程对于优先级的设定。**



#### 4.1.4 线程的状态



**Java线程**在运行的**生命周期**中可能处于表4-1所示的**6种不同的状态**，在给定的一个时刻，线程只能处于其中的一个状态。

![](https://i.loli.net/2020/10/05/mMNicZew9VXEu5J.png)



线程在自身的生命周期中，并不是固定地处于某个状态，而是**随着代码的执行在不同的状态之间进行切换**，Java线程状态变迁如图4-1所示。

![](https://i.loli.net/2020/10/05/GJiUwEraqOSznNp.png)

1. 线程**创建之后**，调用**start()**方法开始**运行**。
2. 当线程执行**wait()**方法之后，线程进入等待状态。
3. 进入等待状态的线程需要依靠**其他线程的通知**才能够返回到**运行状态**，
4. 而**超时等待状态**相当于在等待状态的基础上**增加了超时限制**，也就是超时时间到达时将会返回到运行状态。
5. 当线程调用**同步方法**时，在**没有获取到锁的情况**下，线程将会进入到**阻塞状态**。
6. 线程在**执行完**Runnable的**run()方法之后**将会进入到**终止状态**。



#### 4.1.5 Daemon线程



Daemon线程是一种**支持型线程**，因为它主要**被用作程序中后台调度以及支持性工作**。这意味着，当一个Java虚拟机中不存在非Daemon线程的时候，Java虚拟机将会退出。可以通过**调用Thread.setDaemon(true)将线程设置为Daemon线程**。（Daemon属性需要在启动线程之前设置，不能在启动线程之后设置。）



Daemon线程被用作完成支持性工作，但是**在Java虚拟机退出时Daemon线程中的finally块并不一定会执行，**



### 4.2 启动和终止线程



#### 4.2.1 构造线程



在运行线程之前首先要**构造一个线程对象**，线程对象在构造的时候**需要提供线程所需要的属性**，如线程所属的线程组、线程优先级、是否是Daemon线程等信息



````java
private void init(ThreadGroup g, Runnable target, String name,long stackSize,
AccessControlContext acc) {
    
    //必须得指定一个名字
    if (name == null) {
    	throw new NullPointerException("name cannot be null");
    }
    
    // 当前线程就是该线程的父线程
    Thread parent = currentThread();
    this.group = g;
    
    // 将daemon、priority属性设置为父线程的对应属性
    this.daemon = parent.isDaemon();
    this.priority = parent.getPriority();
    this.name = name.toCharArray();
    this.target = target;
    setPriority(priority);
    
    // 将父线程的InheritableThreadLocal复制过来
    if (parent.inheritableThreadLocals != null){
    	this.inheritableThreadLocals=ThreadLocal
        .createInheritedMap(parent.inheritableThreadLocals);
    }
    
    // 分配一个线程ID
    tid = nextThreadID();
}
````



一个新构造的线程对象是由其parent线程来进行空间分配的，而child线程继承了parent是否为Daemon、优先级和加载资源的contextClassLoader以及可继承的ThreadLocal，同时还会分配一个唯一的ID来标识这个child线程。



#### 4.2.2 启动线程



线程对象在初始化完成之后，调用start()方法就可以启动这个线程。



线程start()方法的含义是：**当前线程**（即parent线程）同步**告知Java虚拟机**，**只要线程规划器空闲，应立即启动调用start()方法的线程。**



#### 4.2.3 过期的suspend()、resume()、stop()



如果把播放音乐比作一个线程的运作，那么对音乐播放做出的**暂停**、**恢复**和**停止**操作对应在线程Thread的API就是**suspend()**、**resume()**和**stop()**。



但是这些API是过期的，也就是不建议使用的。不建议使用的原因主要有：

- 以**suspend()**方法为例，在调用后，线程不会释放已经占有的资源（比如锁），而是**占有着资源进入睡眠状态**，这样容易引发死锁问题。
- 同样，**stop()**方法在终结一个线程时**不会保证线程的资源正常释放**，通常是没有给予线程完成资源释放工作的机会，因此会导致程序可能工作在不确定状态下。



#### 4.2.4 理解中断



##### 4.2.4.1 什么是中断？

在Java中没有办法立即停止一条线程，然而停止线程却显得尤为重要，如取消一个耗时操作。因此，Java提供了一种用于**停止线程的机制**——中断。

- **中断只是一种协作机制**，**Java没有给中断增加任何语法，中断的过程完全需要程序员自己实现**。若要中断一个线程，你需要手动调用该线程的interrupted方法，该方法也仅仅是将线程对象的中断标识设成true；接着你需要自己写代码不断地检测当前线程的标识位；如果为true，表示别的线程要求这条线程中断，此时**究竟该做什么需要你自己写代码实现。**
- **每个线程对象中都有一个标识，用于表示线程是否被中断**；该标识位为true表示中断，为false表示未中断；
- **通过调用线程对象的interrupt方法将该线程的标识位设为true；可以在别的线程中调用，也可以在自己的线程中调用。**

##### 4.2.4.2 中断的相关方法



- public void interrupt() ：**将调用者线程的中断状态设为true**。
- public boolean isInterrupted() ：判断调用者线程的**中断状态**。
- public static boolean interrupted ：只能通过Thread.interrupted()调用。 它会做两步操作：
  - 返回**当前线程**的**中断状态**；
  - 将当前线程的**中断状态设为false**；



##### 4.2.4.3 如何使用中断？



要使用中断，首先需要在**可能会发生中断的线程中不断监听中断状态，一旦发生中断，就执行相应的中断处理代码。** 



当需要中断线程时，调用该线程对象的interrupt函数即可。



**1.设置中断监听**

```java
Thread t1 = new Thread( new Runnable(){
    public void run(){
        // 若未发生中断，就正常执行任务
        while(!Thread.currentThread.isInterrupted()){
            // 正常任务代码……
        }

        // 中断的处理代码……
        doSomething();
    }
} ).start();
```

**正常的任务代码被封装在while循环中，每次执行完一遍任务代码就检查一下中断状态；一旦发生中断，则跳过while循环，直接执行后面的中断处理代码。**



**2.触发中断**

```text
t1.interrupt();
```

上述代码执行后会将t1对象的中断状态设为true，此时t1线程的正常任务代码执行完成后，进入下一次while循环前`Thread.currentThread.isInterrupted()`的结果为true，此时退出循环，执行循环后面的中断处理代码。



#### 4.2.5 如何安全地停止线程？



stop函数停止线程过于暴力，它会立即停止线程，不给任何资源释放的余地，下面介绍两种安全停止线程的方法。



##### **4.2.5.1.循环标记变量**

自定义一个共享的boolean类型变量，表示当前线程是否需要中断。

- 中断标识

```java
volatile boolean interrupted = false;
```

- 任务执行函数

```java
Thread t1 = new Thread( new Runnable(){
    public void run(){
        while(!interrupted){
            // 正常任务代码……
        }
        // 中断处理代码……
        // 可以在这里进行资源的释放等操作……
    }
} );
```

- 中断函数

```java
Thread t2 = new Thread( new Runnable(){
    public void run(){
        interrupted = true;
    }
} );
```



##### **4.2.5.2.循环中断状态**

- 中断标识：由线程对象提供，无需自己定义。
- 任务执行函数

```java
Thread t1 = new Thread( new Runnable(){
    public void run(){
        while(!Thread.currentThread.isInterrupted()){
            // 正常任务代码……
        }
        // 中断处理代码……
        // 可以在这里进行资源的释放等操作……
    }
} );
```

- 中断函数

```java
t1.interrupt();
```



上述两种方法**本质一样**，都是通过循环**查看一个共享标记为来判断线程是否需要中断**，他们的区别在于：第一种方法的标识位是我们自己设定的，而第二种方法的标识位是Java提供的。除此之外，他们的实现方法是一样的。



上述两种方法之所以较为安全，是因为**一条线程发出终止信号后，接收线程并不会立即停止，而是将本次循环的任务执行完，再跳出循环停止线程。此外，程序员又可以在跳出循环后添加额外的代码进行收尾工作。**



### 4.3 线程间通信



#### 4.3.1 volatile和synchronized关键字



Java支持多个线程同时访问一个对象或者对象的成员变量，由于每个线程可以拥有这个变量的拷贝，所以程序在执行过程中，**一个线程看到的变量并不一定是最新的**。



关键字**volatile**可以用来**修饰字段**（成员变量），就是**告知程序任何对该变量的访问均需要从共享内存中获取，而对它的改变必须同步刷新回共享内存**，它能保证所有线程对变量访问的可见性。



关键字synchronized可以**修饰方法**或者以**同步块**的形式来进行使用，它主要确保多个线程**在同一个时刻，只能有一个线程处于方法或者同步块中**，它**保证**了**线程对变量访问的可见性和排他性**。



任意一个对象都拥有自己的**监视器**，当这个对象由同步块或者这个对象的同步方法调用时，执行方法的**线程**必须**先获取到该对象的监视器才能进入同步块或者同步方法**，而**没有获取**到监视器（执行该方法）的线程将会被**阻塞在同步块和同步方法的入口处**，进入**BLOCKED状态**。



![](https://i.loli.net/2020/10/06/UEti1VgN5wrXy6n.png)

从上图中可以看到，任意线程对Object的访问，首先要获得Object的监视器。

- 如果**获取失败**，线程**进入同步队列**，线程状态变为**BLOCKED**。
- 当其他线程**释放了锁**，则该释放操作**唤醒阻塞在同步队列中的线程**，**使其重新尝试对监视器的获取。**



#### 4.3.2 等待/通知机制



一个线程修改了一个对象的值，而另一个线程感知到了变化，然后进行相应的操作，整个过程开始于一个线程，而最终执行又是另一个线程。前者是生产者，后者就是消费者。



等待/通知的相关方法是任意Java对象都具备的，因为这些方法被定义在所有对象的超类java.lang.Object上，方法和描述如表4-2所示。

![](https://i.loli.net/2020/10/06/BAqGs8r52jOYNCc.png)



等待/通知机制，是指

- 一个**线程A**调用了**对象O**的**wait()方法**进入**等待状态**，
- 而另一个**线程B**调用了**对象O的notify()**或者**notifyAll()**方法，
- **线程A** **收到通知后**从对象O的wait()方法**返回**，进而**执行后续操作**。
- 上述两个线程**通过对象O来完成交互**



下面举一个等待通知机制的例子

````java
package ThreadPoolSample;

import java.util.concurrent.TimeUnit;

public class WaitNotify {

    static Object lock = new Object();
    static boolean flag = true;

    public static void main(String[] args) throws InterruptedException {
        Thread waitThread = new Thread(new Wait(), "WaitThread");
        waitThread.start();
        TimeUnit.SECONDS.sleep(1);
        Thread notifyThread = new Thread(new Notify(), "NotifyThread");
        notifyThread.start();
    }

    static class Wait implements Runnable{

        @Override
        public void run() {
            synchronized (lock){
                //条件不满足时就一直wait
                while(flag){
                    try{
                        lock.wait();
                    }catch (InterruptedException e){
                        //pass
                    }
                }
                //do something
            }
        }
    }

    static class Notify implements Runnable{

        @Override
        public void run() {
            synchronized (lock){
                lock.notifyAll();
                //条件满足
                flag = false;
            }
        }
    }
}
````



上述例子主要说明调用wait和notify的细节

1. 使用wait()、notify()和notifyAll()时需要**先对调用对象加锁**。
2. **调用wait()方法后**，线程状态**由RUNNING变为WAITING**，并将当前线程放置到对象的**等待队列**。
3. notify()或notifyAll()方法调用后，等待线程依旧不会从wait()返回，**需要**调用notify()或
   notifAll()的线程**释放锁之后**，**等待线程才有机会从wait()返回**。
4. notify()方法**将等待队列中的一个等待线程从等待队列中移到同步队列中**，而notifyAll()
   方法则是将等待队列中所有的线程全部移到同步队列，**被移动的线程状态由WAITING变为**
   **BLOCKED。**
5. **从wait()方法返回**的**前提**是**获得了调用对象的锁**。



![](https://i.loli.net/2020/10/06/EieCbgn94HTKoI1.png)

- 在图4-3中，WaitThread首先**获取了对象的锁**，然后调用对象的**wait()方法**，从而**放弃了锁并进入了对象的等待队列**WaitQueue中，进入等待状态。
- 由于WaitThread**释放了对象的锁**，NotifyThread随后**获取了对象的锁**，并调用对象的notify()方法，将WaitThread**从WaitQueue移到SynchronizedQueue中**，此时WaitThread的状态**变为阻塞状态**。
- NotifyThread释放了锁之后，WaitThread**再次获取到锁并从wait()方法返回继续执行**。



#### 4.3.3 等待/通知的经典范式



从4.3.2节中的WaitNotify示例中可以提炼出等待/通知的经典范式，该范式分为两部分，分别针对等待方（消费者）和通知方（生产者）。



**等待方**遵循如下原则。

1. **获取对象的锁**。
2. 如果条件**不满足**，那么调用对象的**wait()方法**，被通知后仍要检查条件。
3. **条件满足**则**执行对应的逻辑**。

```
synchronized(对象) {
    while(条件不满足) {
    	对象.wait();
    }
    对应的处理逻辑
}
```



**通知方**遵循如下原则

1. **获得对象的锁**。
2. **改变条件**。
3. **通知**所有**等待**在对象上的**线程**。

````
synchronized(对象) {
	改变条件
	对象.notifyAll();
}
````



#### 4.3.4 管道输入/输出流



**管道输入/输出流**主要**用于线程之间的数据传输**，而**传输的媒介为内存**。



下面举例说明

````java
package ThreadPoolSample;

import java.io.IOException;
import java.io.PipedReader;
import java.io.PipedWriter;

public class Piped {
    public static void main(String[] args) throws Exception {
        PipedWriter out = new PipedWriter();
        PipedReader in = new PipedReader();
        // 将输出流和输入流进行连接，否则在使用时会抛出IOException
        out.connect(in);
        Thread printThread = new Thread(new Print(in), "PrintThread");
        printThread.start();
        int receive = 0;
        try {
            while ((receive = System.in.read()) != -1) {
                out.write(receive);
            }
        } finally {
            out.close();
        }
    }
    static class Print implements Runnable {
        private PipedReader in;
        public Print(PipedReader in) {
            this.in = in;
        }
        public void run() {
            int receive = 0;
            try {
                while ((receive = in.read()) != -1) {
                    System.out.print((char) receive);
                }
            } catch (IOException ex) {
            }
        }
    }
}

````



上面的代码创建了printThread，它用来接受main线程的输入，任何main线程的输入均通过PipedWriter写入，而printThread在另一端通过PipedReader将内容读出并打印。



对于Piped类型的流，**必须先要进行绑定**，也就是**调用connect()方法**，如果没有将输入/输出流绑定起来，对于该流的访问将会抛出异常。



#### 4.3.5 Thread.join()的使用



如果一个线程A执行了thread.join()语句，其含义是：当前**线程A等待thread线程终止之后**才从thread.join()返回，然后**继续执行**后面的逻辑。



下面是JDK中Thread.join()方法的源码（进行了部分调整）。

````java
// 加锁当前线程对象
public final synchronized void join() throws InterruptedException {
    // 条件不满足，继续等待
    while (isAlive()) {
        //在这个线程的同步队列上等待
   		wait(0);
    }
    // 条件符合，方法返回
}
````



当A线程终止时，会调用线程自身的notifyAll()方法，会**通知所有等待在该线程对象上的线程**。然后，这些线程先拿到锁，然后检查到A线程终止了，就会从join()方法上返回。



可以看到join()方法的逻辑结构与4.3.3节中描述的等待/通知经典范式一致，即加锁、循环和处理逻辑3个步骤。



#### 4.3.6 ThreadLocal的使用



**ThreadLocal设计的目的就是为了能够在当前线程中有属于自己的变量，并不是为了解决并发或者共享变量的问题**



ThreadLocal提供了线程的局部变量，每个线程都可以通过`set()`和`get()`来对这个局部变量进行操作，但不会和其他线程的局部变量进行冲突，**实现了线程的数据隔离**。



简要言之：往ThreadLocal中填充的变量属于**当前**线程，该变量对其他线程而言是隔离的。



### 4.4 线程应用示例



#### 4.4.1 等待超时模式



开发人员经常会遇到这样的方法调用场景：

- **调用一个方法时等待一段时间**（一般来说是给定一个时间段）
- 如果该方法**能够在给定的时间段之内得到结果，那么将结果立刻返回**
- 反之，**超时返回默认结果**。



示例代码如下：

````java
// 对当前对象加锁
public synchronized Object get(long mills) throws InterruptedException {
    long future = System.currentTimeMillis() + mills;
    long remaining = mills;
    // 当超时大于0并且result返回值不满足要求
    while ((result == null) && remaining > 0) {
        wait(remaining);
        remaining = future - System.currentTimeMillis();
    }
    return result;
}
````



#### 4.4.2 连接池技术



线程池技术**预先创建了若干数量的线程**，并且**不能由用户直接对线程的创建进行控制**，在这个前提下**重复使用固定或较为固定数目的线程来完成任务的执行**。

这样做的好处是

- 一方面，**消除了频繁创建和消亡线程的系统资源开销**，
- 另一方面，**面对过量任务的提交能够平缓的劣化**。



下面看一个简单的线程池接口定义

````java
public interface ThreadPool<Job extends Runnable> {
    // 执行一个Job，这个Job需要实现Runnable
    void execute(Job job);
    // 关闭线程池
    void shutdown();
    // 增加工作者线程
    void addWorkers(int num);
    // 减少工作者线程
    void removeWorker(int num);
    // 得到正在等待执行的任务数量
    int getJobSize();
}
````



客户端可以通过`execute(Job)`方法将Job提交到线程池执行。而`addWorkers`和`removeWorker`用来调整工作者线程的数量，`shutdown`用来关闭线程池。



下面是该线程池接口的一种实现

```java
package ThreadPoolSample;

import javafx.concurrent.Worker;

import java.util.ArrayList;
import java.util.Collections;
import java.util.LinkedList;
import java.util.List;
import java.util.concurrent.atomic.AtomicLong;

public class DefaultThreadPool<Job extends Runnable> implements ThreadPool<Job> {

    private static final int MAX_WORKER_NUMBERS = 10;

    private static final int DEFAULT_WORKER_NUMBERS = 5;

    private static final int MIN_WORKER_NUMBERS = 1;

    //工作队列
    private final LinkedList<Job> jobs = new LinkedList<>();

    //工作者队列
    private final List<Worker> workers = Collections.synchronizedList(new ArrayList<>());

    //工作者线程的数量
    private int workerNum = DEFAULT_WORKER_NUMBERS;

    //线程编号
    private AtomicLong threadNum = new AtomicLong();

    public DefaultThreadPool(){
        initWorkers(this.workerNum);
    }

    public DefaultThreadPool(int workerNum){
        this.workerNum = workerNum > MAX_WORKER_NUMBERS ? MAX_WORKER_NUMBERS
                : (workerNum < MIN_WORKER_NUMBERS ? MIN_WORKER_NUMBERS : workerNum);
        initWorkers(this.workerNum);
    }

    //execute方法将job放到Jobs列表中，并且尝试通知一个挂起的工作者线程
    @Override
    public void execute(Job job) {
        if(job != null){
            //向工作列表中添加一个工作，然后尝试通知一个挂起的线程
            synchronized (jobs){
                jobs.addLast(job);
                jobs.notify();
            }
        }
    }

    //线程池的关闭方法就是尝试将每个工作者线程关闭
    @Override
    public void shutdown() {
        for(Worker worker: workers){
            worker.shutdown();
        }
    }

    //调用initWorkers来增加工作者线程的数量
    @Override
    public void addWorker(int num) {
        synchronized (jobs){
            if(num + this.workerNum > MAX_WORKER_NUMBERS){
                num = MAX_WORKER_NUMBERS - this.workerNum;
            }
            initWorkers(num);
            this.workerNum += num;
        }
    }

    //直接从工作者线程池中删除线程，并且尝试将删除的线程shutdown掉
    @Override
    public void removeWorker(int num) {
        synchronized (jobs){
            if(num >= this.workerNum){
                throw new IllegalArgumentException("beyond workNum");
            }
            int count = 0;
            while(count < num){
                Worker worker = workers.get(count);
                if(workers.remove(worker)){
                    worker.shutdown();
                    count ++;
                }
            }
            this.workerNum -= count;
        }
    }

    @Override
    public int getJobSize() {
        return jobs.size();
    }

    //初始化工作者线程
    private void initWorkers(int num){
        for(int i = 0; i < num; i ++){
            
            //构建worker
            Worker worker = new Worker();
            
            //加入工作者列表
            workers.add(worker);
            
            //初始化线程并start
            Thread thread = new Thread(worker, "ThreadPool-Worker-" 
                                       + threadNum.incrementAndGet());
            thread.start();
        }
    }

    //工作者类
    class Worker implements Runnable{

        //终止标志
        private volatile boolean running = true;

        //shutdown方法用来终止线程
        public void shutdown(){
            running = false;
        }

        //run方法
        @Override
        public void run() {
            //先检查终止标志，如果没有调整为false则继续循环
            while(running){
                
                Job job = null;
                
                //获取jobs列表的锁
                synchronized (jobs){
                    //如果jobs列表中没有工作了，就在jobs对象上等待
                    while(jobs.isEmpty()){
                        try{
                            jobs.wait();
                        }catch (InterruptedException ie){
                            Thread.currentThread().interrupt();
                            return;
                        }
                    }
                    //如果jobs列表有工作，就弹出一个
                    job = jobs.removeFirst();
                }
                
                //处理弹出的这个job
                if(job != null){
                    try{
                        job.run();
                    }catch (Exception e){
                        //pass
                    }
                }
            }
        }
    }
}
```

