# ReentrantLock 源码解析

ReentrantLock锁的实现基于AQS同步器:

AQS维护一个volatile的int型state变量<br/>
AQS有条双向队列存放阻塞的等待线程，并提供一系列判断和处理方法

+ state是独占的，还是共享的；
+ state被获取后，其他线程需要等待；
+ state被释放后，唤醒等待线程；

只有一个实例变量sync提供锁的功能,在构造函数中初始化:
        
        private final Sync sync;

sync有两种实例对象：FairSync（公平锁）和NofairSync（非公平锁），二者及其抽象类Sync都是继承于AQS以及RentrantLock的内部类。

## 加锁

ReentrantLock 通过lock()加锁：

        public void lock() {
            sync.lock();
        }

case 1. 如果这个锁没有被任何线程持有，立即锁定并设置count（持锁计数）为1，方法立即返回

case 2. 如果当前线程已经持有这个锁，count+1，方法立即返回

case 3. 如果这个锁被其他线程持有，那当前线程立即休眠停止调度，直到这个锁可以被获得（count被设为0）

`lock()`方法内部是通过sync.lock()来实现加锁，即根据不同的加锁实现来执行不同的加锁策略（公平or非公平）

FairSync和NofairSync内的lock()方法都调用了AQS的acquire(1)<br/>

非公平因为不用考虑顺序情况，在state为0（无锁状态）时可直接cas尝试抢占state，失败的话，执行acquire()尝试加锁，否则阻塞线程

公平锁执行acquire()尝试加锁
        
        // NofairLock
        final void lock() {
            // case 1
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            // case 2/3
            else
                acquire(1);
        }


        // FairLock
        final void lock() {
            //case 2/3
            acquire(1);
        }

AQSDEacquire()方法中先调用实现类的tryAcquire()方法尝试获取锁,失败后将线程包装为Node并加入双端队列(addWriter),最后执行acquireQueued()挂起当前线程

#### Node中分别有:

+ pre: 指向上一个等待线程Node
+ tail: 指向下一个等待线程Node
+ thread: 存放线程实例
+ waitStatus: 线程状态 
    + SIGNAL=-1 等待被唤醒(前一个Node释放锁后就会通知当前结点)
    + CANCELLED=1 已经结束(在同步队列中等待超时或者被中断)
    + CONDITION=-2 条件状态(在等待队列中等待,注意:等待队列与同步队列不是同一个概念)
    + PROPAGATE=-3 共享模式下,同步状态会被传播(不用管,ReentrantLock用不到) 


            // AQS
            public final void acquire(int arg) {
                if (!tryAcquire(arg) &&
                    acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
                    selfInterrupt();
            }


+ 公平锁的尝试加锁逻辑:

如果state还不是被占有状态(锁未被占有),且当前线程就是第一个等待锁的线程,则尝试获得锁(设置state),并将当前线程设为占有锁的线程
如果state不为0,锁已经被占有,判断是否被当前线程占有,是则state计数+1

        /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
        

+ 非公平锁的尝试加锁逻辑:

跟公平锁加锁逻辑类似,只不过少了检查是否有等待当前锁的线程

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
        
        /**
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
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

+ acquireQueued() 用于管理同步队列中的等待线程,进行线程的挂起

        final boolean acquireQueued(final Node node, int arg) {
            boolean failed = true;
            try {
                boolean interrupted = false;
                // 注意:在自旋期间当前线程可能会被唤醒,
                for (;;) {
                    // 获取该结点的前一个结点,若前一个结点是头结点,那么这个结点就可以开始尝试去争抢锁
                    final Node p = node.predecessor();
                    if (p == head && tryAcquire(arg)) {
                        争抢成功(头结点线程被释放),把当前结点设为头结点,返回false
                        setHead(node);
                        p.next = null; // help GC
                        failed = false;
                        return interrupted;
                    }
                    // 判断当前线程是否需要挂起,若waitStatus(Node/线程状态)为
                    `SIGNAL`挂起,
                    `CANCELLED`在同步队列中移除当前结点
                    否则将其设为`SIGNAL`在下个自选周期中挂起线程
                    if (shouldParkAfterFailedAcquire(p, node) &&
                        // 该方法挂起线程
                        parkAndCheckInterrupt())
                        // 当被唤醒后获取当前线程interrput状态(判断是被唤醒还是中断操作),是则设interrupt为true,在外部方法中断当前线程
                        interrupted = true;
                }
            } finally {
                if (failed)
                    cancelAcquire(node);
            }
        }

+ 至此加锁结束

## 解锁

+ 公平锁与非公平锁使用的都是同一种加锁操作

调用AQS自带的release()方法:

    // RL
    public void unlock() {
        sync.release(1);
    }

使用ReentrantLock自定义的解锁逻辑解锁,解锁成功后获取当前等待的线程头,解除一个等待的Node线程

    // AQS
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

tryRelease通过state的大小判断当前解锁的线程是不是最后一个锁,是则释放当前锁的占有线程并返回true

    // RL
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }

这段代码作用是唤醒同步队列中第一个非CANCELLED状态的结点的阻塞线程.<br/>
故,即使使用非公平锁,一旦线程获取不到锁进入同步队列,之后的唤醒也是按顺序的,只是唤醒后不一定能拿到锁,需要跟新的线程竞争.<br/>
而公平锁按顺序唤醒,也按顺序获取锁

    // AQS
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }


## Condition原理

一个Condition有一个等待队列,一个AQS有一个同步队列,所以一个基于AQS的锁可以有多个等待队列

newCondition()方法实例化且返回AQS的ConditionObject

        // ReentrantLock
        final ConditionObject newCondition() {
            return new ConditionObject();
        }

ConditionObject的await()方法, 挂起当前线程:

        // ConditionObject
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            // 将当前线程包装加入这个Condition的等待队列
            Node node = addConditionWaiter();
            // 释放当前线程对锁的持有,唤醒后继线程
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            // 当前线程是否在同步队列中,否说明线程是活跃状态,需要挂起
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            // 被唤醒后尝试获取锁
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            // 清理等待队列中不为CONDITION状态(即CANCELLED)的结点
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

ConditionObject的signal()方法, 唤醒一个线程:

        // 取出等待队列中第一个线程唤醒
        public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }

        // 将要唤醒的线程结点的后继设为null,即将其移出等待队列
        // transferForSignal(node)进行cas唤醒,若失败,则说明被其他线程唤醒了,重新取下一个线程进行唤醒
        // 若取到最后一个线程结点,就把尾结点指向设为null
        private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }

        // 唤醒一个线程 
        final boolean transferForSignal(Node node) {
            /*
            * If cannot change waitStatus, the node has been cancelled.
            */
            if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
                return false;

            /*
            * Splice onto queue and try to set waitStatus of predecessor to
            * indicate that thread is (probably) waiting. If cancelled or
            * attempt to set waitStatus fails, wake up to resync (in which
            * case the waitStatus can be transiently and harmlessly wrong).
            */
            // 将唤醒的结点加入同步队列末端,并放回其前继结点
            Node p = enq(node);
            int ws = p.waitStatus;
            // 若前继结点无法设为SIGNAL状态时,唤醒线程
            if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
                LockSupport.unpark(node.thread);
            return true;
        }