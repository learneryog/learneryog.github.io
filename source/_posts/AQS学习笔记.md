---
title: AQS学习笔记
date: 2019-08-01 21:52:30
categories: 并发
cover: /img/Random-img/103.jpg
top: true
tags: 
- AQS
- 并发
---

# AQS简介


----------


> AbstractQueuedSynchronizer简称AQS，是一个用于构建锁和同步容器的框架。事实上concurrent包内许多类都是基于AQS构建，例如ReentrantLock，Semaphore，CountDownLatch，ReentrantReadWriteLock，FutureTask等。AQS解决了在实现同步容器时设计的大量细节问题。

 AbstractQueuedSynchronizer继承了AbstractOwnableSynchronizer，这个类只有一个变量：exclusiveOwnerThread，表示当前占用该锁的线程，并且提供了相应的get，set方法。
AQS内部通过一个int类型的成员变量state来控制同步状态，当state=0时，则说明没有任何线程占有共享资源的锁，当state=1时，则说明有线程目前正在使用共享变量，其他线程必须加入同步队列进行等待。
AQS内部通过内部类Node构成FIFO的同步队列来完成线程获取锁的排队工作，同时利用内部类ConditionObject构建等待队列，当Condition调用wait()方法后，线程将会加入等待队列中，而当Condition调用signal()方法后，线程将从等待队列转移动同步队列中进行锁竞争。注意这里涉及到两种队列，一种是同步队列，当线程请求锁而等待的后将加入同步队列等待，而另一种则是等待队列(可有多个)，通过Condition调用await()方法释放锁后，将加入等待队列。

# AQS常用 API


----------


AQS主要提供了如下一些方法：
>
> **getState()**：返回同步状态的当前值；
>**setState(int newState)**：设置当前同步状态；
>**compareAndSetState(int expect, int update)**：使用CAS设置当前状态，该方法能够保证状态设置的原子性；
>**tryAcquire(int arg)**：独占式获取同步状态，获取同步状态成功后，其他线程需要等待该线程释放同步状态才能获取同步状态；
>**tryRelease(int arg)**：独占式释放同步状态；
>**tryAcquireShared(int arg)**：共享式获取同步状态，返回值大于等于0则表示获取成功，否则获取失败；
>**tryReleaseShared(int arg)**：共享式释放同步状态；
>**isHeldExclusively()**：当前同步器是否在独占式模式下被线程占用，一般该方法表示是否被当前线程所独占；
>**acquire(int arg)**：独占式获取同步状态，如果当前线程获取同步状态成功，则由该方法返回，否则，将会进入同步队列等待，该方法将会调用可重写的tryAcquire(int arg)方法；
>**acquireInterruptibly(int arg)**：与acquire(int arg)相同，但是该方法响应中断，当前线程为获取到同步状态而进入到同步队列中，如果当前线程被中断，则该方法会抛出InterruptedException异常并返回；
>**tryAcquireNanos(int arg,long nanos)**：超时获取同步状态，如果当前线程在nanos时间内没有获取到同步状态，那么将会返回false，已经获取则返回true；
>**acquireShared(int arg)**：共享式获取同步状态，如果当前线程未获取到同步状态，将会进入同步队列等待，与独占式的主要区别是在同一时刻可以有多个线程获取到同步状态；
>**acquireSharedInterruptibly(int arg)**：共享式获取同步状态，响应中断；
>**tryAcquireSharedNanos(int arg, long nanosTimeout)**：共享式获取同步状态，增加超时限制；
>**release(int arg)**：独占式释放同步状态，该方法会在释放同步状态之后，将同步队列中第一个节点包含的线程唤醒；
>**releaseShared(int arg)**：共享式释放同步状态；

# CLH同步队列


----------


AQS内部维护着一个FIFO队列，该队列就是CLH同步队列。
CLH同步队列是一个FIFO双向队列，AQS依赖它来完成同步状态的管理，当前线程如果获取同步状态失败时，AQS则会将当前线程已经等待状态等信息构造成一个节点（Node）并将其加入到CLH同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点唤醒（公平锁），使其再次尝试获取同步状态。
在CLH同步队列中，一个节点表示一个线程，它保存着线程的引用（thread）、状态（waitStatus）、前驱节点（prev）、后继节点（next），Node是AQS的内部类，其定义如下：
``` java
static final class Node {
    /** 共享 */
    static final Node SHARED = new Node();
    /** 独占 */
    static final Node EXCLUSIVE = null;
     //因为超时或者中断，节点会被设置为取消状态，
     //被取消的节点时不会参与到竞争中的，他会一直保持取消状态不会转变为其他状态；
    static final int CANCELLED =  1;
     //后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者被取消，
    //将会通知后继节点，使后继节点的线程得以运行
    static final int SIGNAL    = -1;
     //节点在等待队列中，节点线程等待在Condition上，当其他线程对Condition调用了
     //signal()后，改节点将会从等待队列中转移到同步队列中，加入到同步状态的获取中
    static final int CONDITION = -2;
    /** 表示下一次共享式同步状态获取将会无条件地传播下去*/
    static final int PROPAGATE = -3;
    /** 等待状态 */
    volatile int waitStatus;
    /** 前驱节点 */
    volatile Node prev;
    /** 后继节点 */
    volatile Node next;
    /** 获取同步状态的线程 */
    volatile Thread thread;
    Node nextWaiter;
    final boolean isShared() {
        return nextWaiter == SHARED;
    }
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }
    Node() {
    }
    Node(Thread thread, Node mode) {
        this.nextWaiter = mode;
        this.thread = thread;
    }
    Node(Thread thread, int waitStatus) {
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```
其中**SHARED**和**EXCLUSIVE**常量分别代表共享模式和独占模式，所谓共享模式是一个锁允许多条线程同时操作，如信号量Semaphore采用的就是基于AQS的共享模式实现的，而独占模式则是同一个时间段只能有一个线程对共享资源进行操作，多余的请求线程需要排队等待，如ReentranLock。变量waitStatus则表示当前被封装成Node结点的等待状态，共有4种取值CANCELLED、SIGNAL、CONDITION、PROPAGATE。
>CANCELLED：值为1，在同步队列中等待的线程等待超时或被中断，需要从同步队列中取消该Node的结点，其结点的waitStatus为CANCELLED，即结束状态，进入该状态后的结点将不会再变化。
SIGNAL：值为-1，被标识为该状态的结点，当其前继结点的线程释放了同步锁或被取消，将会通知该结点的线程执行。说白了，就是处于唤醒状态，只要前继结点释放锁，就会通知标识为SIGNAL状态的后继结点的线程执行。
CONDITION：值为-2，与Condition相关，该标识的结点处于等待队列中，结点的线程等待在Condition上，当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将从等待队列转移到同步队列中，等待获取同步锁。
PROPAGATE：值为-3，与共享模式相关，在共享模式中，该状态标识结点的线程处于可运行状态。
CLH同步队列模型及结构图如下：

![同步队列](/img/post-img/19-8-1-1.png)
![同步队列](/img/post-img/19-8-1-2.png)

head和tail分别是AQS中的变量，其中head指向同步队列的头部，**注意head为空结点，不存储信息**。而tail则是同步队列的队尾，同步队列采用的是双向链表的结构这样可方便队列进行结点增删操作。state变量则是代表同步状态，执行当线程调用lock方法进行加锁后，如果此时state的值为0，则说明当前线程可以获取到锁，同时将state设置为1，表示获取成功。如果state已为1，也就是当前锁已被其他线程持有，那么当前执行线程将被封装为Node结点加入同步队列等待。每个Node结点内部关联其前继结点prev和后继结点next，这样可以方便线程释放锁后快速唤醒下一个在等待的线程。

## 入队

无非就是tail指向新节点、新节点的prev指向当前最后的节点，当前最后一个节点的next指向当前节点。addWaiter(Node node)方法：
``` java
    private Node addWaiter(Node mode) {
        //新建Node
        Node node = new Node(Thread.currentThread(), mode);
        //快速尝试添加尾节点
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            //CAS设置尾节点
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        //多次尝试
        enq(node);
        return node;
    }
```
addWaiter(Node node)先通过快速尝试设置尾节点，如果失败，则调用enq(Node node)方法设置尾节点：
``` java
    private Node enq(final Node node) {
        //多次尝试，直到成功为止
        for (;;) {
            Node t = tail;
            //tail不存在，设置为首节点
            if (t == null) {
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                //设置为尾节点
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```
在上面代码中，两个方法都是通过一个CAS方法compareAndSetTail(Node expect, Node update)来设置尾节点，该方法可以确保节点是线程安全添加的。在enq(Node node)方法中，AQS通过“死循环”的方式来保证节点可以正确添加，只有成功添加后，当前线程才会从该方法返回，否则会一直执行下去。
过程图如下：
![入队](/img/post-img/19-8-1-3.png)

## 出队

CLH同步队列遵循FIFO，首节点的线程释放同步状态后，将会唤醒它的后继节点（next），而后继节点将会在获取同步状态成功时将自己设置为首节点，这个过程非常简单，head执行该节点并断开原首节点的next和当前节点的prev即可，注意在这个过程是不需要使用CAS来保证的，因为只有一个线程能够成功获取到同步状态。过程图如下：
![出队](/img/post-img/19-8-1-4.png)

# 同步状态的获取与释放


----------


AQS的设计模式采用的模板模式，子类通过继承的方式，实现它的抽象方法来管理同步状态，对于子类而言它并没有太多的活要做，AQS提供了大量的模板方法来实现同步，主要是分为三类：独占式获取和释放同步状态、共享式获取和释放同步状态、查询同步队列中的等待线程情况。自定义子类使用AQS提供的模板方法就可以实现自己的同步语义。

## 独占式

**独占式，同一时刻仅有一个线程持有同步状态。**

### 独占式同步状态获取（acquire方法）

acquire(int arg)方法为AQS提供的模板方法，该方法为独占式获取同步状态，但是该方法对中断不敏感，也就是说由于线程获取同步状态失败加入到CLH同步队列中，后续对线程进行中断操作时，线程不会从同步队列中移除。代码如下：
``` java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
各个方法定义如下：
>
tryAcquire：去尝试获取锁，获取成功则设置锁状态并返回true，否则返回false。该方法自定义同步组件自己实现，该方法必须要保证线程安全的获取同步状态。
addWaiter：如果tryAcquire返回FALSE（获取同步状态失败），则调用该方法将当前线程加入到CLH同步队列尾部。
acquireQueued：当前线程会根据公平性原则来进行阻塞等待（自旋）,直到获取锁为止；并且返回当前线程在等待过程中有没有中断过。
selfInterrupt：产生一个中断。

acquireQueued方法为一个自旋的过程，也就是说当前线程（Node）进入同步队列后，就会进入一个自旋的过程，每个节点都会自行地观察，当条件满足，获取到同步状态后，就可以从这个自旋过程中退出，否则会一直执行下去。如下：
``` java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            //中断标志
            boolean interrupted = false;
            /*
             * 自旋过程，其实就是一个死循环而已
             */
            for (;;) {
                //当前线程的前驱节点
                final Node p = node.predecessor();
                //当前线程的前驱节点是头结点，且同步状态成功
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //获取失败，线程等待--具体后面介绍
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
从上面代码中可以看到，当前线程会一直尝试获取同步状态，当然前提是只有其前驱节点为头结点才能够尝试获取同步状态，理由：
- 保持FIFO同步队列原则。
- 头节点释放同步状态后，将会唤醒其后继节点，后继节点被唤醒后需要检查自己是否为头节点。
acquire(int arg)方法流程图如下：

![acquire流程图](/img/post-img/19-8-1-5.png)

### 独占式获取响应中断（acquireInterruptibly方法）

AQS提供了acquireInterruptibly(int arg)方法，该方法在等待获取同步状态时，如果当前线程被中断了，会立刻响应中断抛出异常InterruptedException。
``` java
    public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }
```
首先校验该线程是否已经中断了，如果是则抛出InterruptedException，否则执行tryAcquire(int arg)方法获取同步状态，如果获取成功，则直接返回，否则执行doAcquireInterruptibly(int arg)。doAcquireInterruptibly(int arg)定义如下：
``` java
private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
doAcquireInterruptibly(int arg)方法与acquire(int arg)方法仅有两个差别。
1. 方法声明抛出InterruptedException异常，
2. 在中断方法处不再是使用interrupted标志，而是直接抛出InterruptedException异常。

### 独占式超时获取（tryAcquireNanos方法）

AQS除了提供上面两个方法外，还提供了一个增强版的方法：tryAcquireNanos(int arg,long nanos)。该方法为acquireInterruptibly方法的进一步增强，它除了响应中断外，还有超时控制。即如果当前线程没有在指定时间内获取同步状态，则会返回false，否则返回true。如下：
``` java
   public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }
```
tryAcquireNanos(int arg, long nanosTimeout)方法超时获取最终是在doAcquireNanos(int arg, long nanosTimeout)中实现的，如下：
``` java
    private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        //nanosTimeout <= 0
        if (nanosTimeout <= 0L)
            return false;
        //超时时间
        final long deadline = System.nanoTime() + nanosTimeout;
        //新增Node节点
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            //自旋
            for (;;) {
                final Node p = node.predecessor();
                //获取同步状态成功
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                /*
                 * 获取失败，做超时、中断判断
                 */
                //重新计算需要休眠的时间
                nanosTimeout = deadline - System.nanoTime();
                //已经超时，返回false
                if (nanosTimeout <= 0L)
                    return false;
                //如果没有超时，则等待nanosTimeout纳秒
                //注：该线程会直接从LockSupport.parkNanos中返回，
                //LockSupport为JUC提供的一个阻塞和唤醒的工具类，后面做详细介绍
                if (shouldParkAfterFailedAcquire(p, node) &&
                        nanosTimeout > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                //线程是否已经中断了
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
针对超时控制，程序首先记录唤醒时间deadline ，deadline = System.nanoTime() + nanosTimeout（时间间隔）。如果获取同步状态失败，则需要计算出需要休眠的时间间隔nanosTimeout = deadline – System.nanoTime()，如果nanosTimeout <= 0 表示已经超时了，返回false，如果大于spinForTimeoutThreshold（1000L）则需要休眠nanosTimeout ，如果nanosTimeout <= spinForTimeoutThreshold ，就不需要休眠了，直接进入快速自旋的过程。原因在于 spinForTimeoutThreshold 已经非常小了，非常短的时间等待无法做到十分精确，如果这时再次进行超时等待，相反会让nanosTimeout 的超时从整体上面表现得不是那么精确，所以在超时非常短的场景中，AQS会进行无条件的快速自旋。
整个流程如下：
![acquire流程图](/img/post-img/19-8-1-6.png)

### 独占式同步状态释放（release方法）

当线程获取同步状态后，执行完相应逻辑就需要释放同步状态。AQS提供了release(int arg)方法释放同步状态：
``` java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
该方法同样是先调用自定义同步器自定义的tryRelease(int arg)方法来释放同步状态，释放成功后，会调用unparkSuccessor(Node node)方法唤醒后继节点。
**稍微总结下：**
>在AQS中维护着一个FIFO的同步队列，当线程获取同步状态失败后，则会加入到这个CLH同步队列的对尾并一直保持着自旋。在CLH同步队列中的线程在自旋时会判断其前驱节点是否为首节点，如果为首节点则不断尝试获取同步状态，获取成功则退出CLH同步队列。当线程执行完逻辑后，会释放同步状态，释放后会唤醒其后继节点。

## 共享式

共享式与独占式的最主要区别在于同一时刻独占式只能有一个线程获取同步状态，而共享式在同一时刻可以有多个线程获取同步状态。例如读操作可以有多个线程同时进行，而写操作同一时刻只能有一个线程进行写操作，其他操作都会被阻塞。

### 共享式同步状态获取（acquireShared方法）

AQS提供acquireShared(int arg)方法共享式获取同步状态：
``` java
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            //获取失败，自旋获取同步状态
            doAcquireShared(arg);
    }
```
从上面程序可以看出，方法首先是调用tryAcquireShared(int arg)方法尝试获取同步状态，如果获取失败则调用doAcquireShared(int arg)自旋方式获取同步状态，共享式获取同步状态的标志是返回 >= 0 的值表示获取成功。自旋式获取同步状态如下：
``` java
    private void doAcquireShared(int arg) {
        //共享式节点
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //前驱节点
                final Node p = node.predecessor();
                //如果其前驱节点是头结点，获取同步状态
                if (p == head) {
                    //尝试获取同步
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
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
tryAcquireShared(int arg)方法尝试获取同步状态，返回值为int，当其 >= 0 时，表示能够获取到同步状态，这个时候就可以从自旋过程中退出。
acquireShared(int arg)方法不响应中断，与独占式相似，AQS也提供了响应中断、超时的方法，分别是：acquireSharedInterruptibly(int arg)、tryAcquireSharedNanos(int arg,long nanos)。

### 共享式同步状态释放（release方法）

获取同步状态后，需要调用release(int arg)方法释放同步状态，方法如下：
``` java
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```
因为可能会存在多个线程同时进行释放同步状态资源，所以需要确保同步状态安全地成功释放，一般都是通过CAS和循环来完成的。

# 阻塞和唤醒线程


----------


在线程获取同步状态时如果获取失败，则加入CLH同步队列，通过自旋的方式不断获取同步状态，但是在自旋的过程中则需要判断当前线程是否需要阻塞，其主要方法在acquireQueued()：
``` java
if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
```
通过这段代码可以看到，在获取同步状态失败后，线程并不是立马进行阻塞，需要检查该线程的状态，检查状态的方法为 shouldParkAfterFailedAcquire(Node pred, Node node) 方法，该方法主要靠前驱节点判断当前线程是否应该被阻塞，代码如下：
``` java
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        //前驱节点
        int ws = pred.waitStatus;
        //状态为signal，表示当前线程处于等待状态，直接放回true
        if (ws == Node.SIGNAL)
            return true;
        //前驱节点状态 > 0 ，则为Cancelled,表明该节点已经超时或者被中断了，
        //需要从同步队列中取消
        if (ws > 0) {
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        }
        //前驱节点状态为Condition、propagate
        else {
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
这段代码主要检查当前线程是否需要被阻塞，具体规则如下：
>
如果当前线程的前驱节点状态为SINNAL，则表明当前线程需要被阻塞，调用unpark()方法唤醒，直接返回true，当前线程阻塞
如果当前线程的前驱节点状态为CANCELLED（ws > 0），则表明该线程的前驱节点已经等待超时或者被中断了，则需要从CLH队列中将该前驱节点删除掉，直到回溯到前驱节点状态 <= 0 ，返回false
如果前驱节点非SINNAL，非CANCELLED，则通过CAS的方式将其前驱节点设置为SINNAL，返回false

如果 shouldParkAfterFailedAcquire(Node pred, Node node) 方法返回true，则调用parkAndCheckInterrupt()方法阻塞当前线程：
``` java
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
parkAndCheckInterrupt() 方法主要是把当前线程挂起，从而阻塞住线程的调用栈，同时返回当前线程的中断状态。其内部则是调用LockSupport工具类的park()方法来阻塞该方法。
当线程释放同步状态后，则需要唤醒该线程的后继节点：
``` java
//唤醒后继节点
unparkSuccessor(h);
```
调用unparkSuccessor(Node node)唤醒后继节点：
``` java
    private void unparkSuccessor(Node node) {
        //当前节点状态
        int ws = node.waitStatus;
        //当前状态 < 0 则设置为 0
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        //当前节点的后继节点
        Node s = node.next;
        //后继节点为null或者其状态 > 0 (超时或者被中断了)
        if (s == null || s.waitStatus > 0) {
            s = null;
            //从tail节点来找可用节点
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        //唤醒后继节点
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```
可能会存在当前线程的后继节点为null，超时、被中断的情况，如果遇到这种情况了，则需要跳过该节点，但是为何是从tail尾节点开始，而不是从node.next开始呢？原因在于node.next仍然可能会存在null或者取消了，所以采用tail回溯办法找第一个可用的线程。最后调用LockSupport的unpark(Thread thread)方法唤醒该线程。

### LockSupport

从上面我可以看到，当需要阻塞或者唤醒一个线程的时候，AQS都是使用LockSupport这个工具类来完成的。
LockSupport是用来创建锁和其他同步类的基本线程阻塞原语
每个使用LockSupport的线程都会与一个许可关联，如果该许可可用，并且可在进程中使用，则调用park()将会立即返回，否则可能阻塞。如果许可尚不可用，则可以调用 unpark 使其可用。但是注意许可不可重入，也就是说只能调用一次park()方法，否则会一直阻塞。
LockSupport定义了一系列以park开头的方法来阻塞当前线程，unpark(Thread thread)方法来唤醒一个被阻塞的线程。如下：
![LockSupport](/img/post-img/19-8-1-7.png)

park(Object blocker)方法的blocker参数，主要是用来标识当前线程在等待的对象，该对象主要用于问题排查和系统监控。
park方法和unpark(Thread thread)都是成对出现的，同时unpark必须要在park执行之后执行，当然并不是说没有调用unpark线程就会一直阻塞，park有一个方法，它带了时间戳（parkNanos(long nanos)：为了线程调度禁用当前线程，最多等待指定的等待时间，除非许可可用）。
park()和unpark(Thread thread)方法的源码如下：
``` java
    public static void park() {
        UNSAFE.park(false, 0L);
    }
    public static void unpark(Thread thread) {
        if (thread != null)
            UNSAFE.unpark(thread);
    }
```
从上面可以看出，其内部的实现都是通过UNSAFE（sun.misc.Unsafe UNSAFE）来实现的，其定义如下：
`public native void park(boolean var1, long var2);
public native void unpark(Object var1);`

两个都是native本地方法。Unsafe 是一个比较危险的类，主要是用于执行低级别、不安全的方法集合。尽管这个类和所有的方法都是公开的（public），但是这个类的使用仍然受限，你无法在自己的java程序中直接使用该类，因为只有授信的代码才能获得该类的实例。