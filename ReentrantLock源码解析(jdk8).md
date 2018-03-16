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


    // AQS
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }




## 解锁