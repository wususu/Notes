# ThreadPoolExecutor解析

## 相关域

        private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
        private static final int COUNT_BITS = Integer.SIZE - 3;
        private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

        // runState is stored in the high-order bits
        private static final int RUNNING    = -1 << COUNT_BITS;
        private static final int SHUTDOWN   =  0 << COUNT_BITS;
        private static final int STOP       =  1 << COUNT_BITS;
        private static final int TIDYING    =  2 << COUNT_BITS;
        private static final int TERMINATED =  3 << COUNT_BITS;

        // Packing and unpacking ctl
        private static int runStateOf(int c)     { return c & ~CAPACITY; }
        private static int workerCountOf(int c)  { return c & CAPACITY; }
        private static int ctlOf(int rs, int wc) { return rs | wc; }


+ 线程池中实现了内部类Worker,实现了Runnable,扩展了AbstractQueuedSynchronizer(一个同步框架),其实例域封装了一个线程, 可以认为一个worker实例是一个工作者线程, 负责执行提交到线程池的任务.

+ ctl: 用于表示线程池状态和当前worker数量的包装属性(可以理解为记账本)，其二进制前三位用于表示`runState`（线程池运行状态），后29位表示当任务数.CAPACITY: 000111111~1; ~CAPACITY: 111000000~0(用于解包装)<br/>
`runStateOf(int c)`通过传入ctl获取当前运行状态<br/>
`workerCountOf(int c)`通过传入ctl获取当前worker数<br/>
`ctlOf(int rs, int wc)`通过传入运行状态和worker数算出ctl


+ RUNNING: 运行状态，此时接收并运行排队中的任务
+ SHUTDOWN: 不再接受新的任务，继续执行排队中的任务
+ STOP: 不再接收新的任务，也不执行排队中的任务，并且中断运行中的任务
+ TIDYING: 整理状态，所有任务都终止，当前worker数为0，即将运行`terminated()`方法（`terminated`交给子类实现，默认什么都不干）
+ TERMINATED: `terminated()`运行完毕


        private final BlockingQueue<Runnable> workQueue;

        private final ReentrantLock mainLock = new ReentrantLock();

        private final HashSet<Worker> workers = new HashSet<Worker>();


+ workQueue: 存储任务的阻塞队列,待执行任务在此排队
+ mainLock: 主锁, 用于控制线程池属性访问修改的同步
+ workers: 当前可用worker(线程)集合


        private int largestPoolSize;

        private long completedTaskCount;

        private volatile ThreadFactory threadFactory;

        private volatile RejectedExecutionHandler handler;

        private volatile long keepAliveTime;

        private volatile boolean allowCoreThreadTimeOut;

        private volatile int corePoolSize;

        private volatile int maximumPoolSize;


+ largestPoolSize: 记录线程池达到过的最高worker数量，　通过主锁访问修改
+ completedTaskCount:　完成的任务总数，只能在worker中更新,通过主锁读
+ threadFactory: 线程工厂
+ handler: 线程池饱和或关闭后的拒绝方案
+ keepAliveTime: 空闲线程的存活时间,超时则回收该线程(如果设置了`allowCoreThreadTimeOut`),也会回收核心线程
+ allowCoreThreadTimeOut: false,核心空闲线程超时不会被回收,反之被回收
+ corePoolSize: 核心线程数量
+ maximumPoolSize: 最高线程数量, 但是最高不超过CAPACITY的值

## 入口方法execute()

        /**
        * Executes the given task sometime in the future.  The task
        * may execute in a new thread or in an existing pooled thread.
        *
        * If the task cannot be submitted for execution, either because this
        * executor has been shutdown or because its capacity has been reached,
        * the task is handled by the current {@code RejectedExecutionHandler}.
        *
        * @param command the task to execute
        * @throws RejectedExecutionException at discretion of
        *         {@code RejectedExecutionHandler}, if the task
        *         cannot be accepted for execution
        * @throws NullPointerException if {@code command} is null
        */
        public void execute(Runnable command) {
            if (command == null)
                throw new NullPointerException();
            /*
            * Proceed in 3 steps:
            *
            * 1. If fewer than corePoolSize threads are running, try to
            * start a new thread with the given command as its first
            * task.  The call to addWorker atomically checks runState and
            * workerCount, and so prevents false alarms that would add
            * threads when it shouldn't, by returning false.
            *
            * 2. If a task can be successfully queued, then we still need
            * to double-check whether we should have added a thread
            * (because existing ones died since last checking) or that
            * the pool shut down since entry into this method. So we
            * recheck state and if necessary roll back the enqueuing if
            * stopped, or start a new thread if there are none.
            *
            * 3. If we cannot queue task, then we try to add a new
            * thread.  If it fails, we know we are shut down or saturated
            * and so reject the task.
            */
            int c = ctl.get();
            if (workerCountOf(c) < corePoolSize) {             // 第一步
                if (addWorker(command, true))
                    return;
                c = ctl.get();
            }
            if (isRunning(c) && workQueue.offer(command)) {   // 第二步
                int recheck = ctl.get();
                if (! isRunning(recheck) && remove(command))
                    reject(command);
                else if (workerCountOf(recheck) == 0)
                    addWorker(null, false);
            }
            else if (!addWorker(command, false))             // 第三步
                reject(command);                             // 第四步
        }


    1. 当前运行的worker小于`corePoolSize`时,直接实例化一个新的worker运行此任务,否则进入第二步
    2. 如果阻塞队列未满则进入阻塞队列排队等待worker执行,否则进入第三步
    3. 如果当前worker数小于`maximumPoolSize`时,尝试实例化一个新的worker来执行此任务,否则进入第四步
    4. 当worker数大于`maximumPoolSize`或线程池正处于关闭过程时,拒绝这个接收这个任务,由rejectHandler处理.

    + addWorker方法将会原子地检查`runState`和`workerCount`,防止多线程状态下中错误的添加worker
    + 第二部使用了双重检查,防止进入条件语句后线程池关闭


## 新建线程:addWorker()

        /*
        * Methods for creating, running and cleaning up after workers
        */

        /**
        * Checks if a new worker can be added with respect to current
        * pool state and the given bound (either core or maximum). If so,
        * the worker count is adjusted accordingly, and, if possible, a
        * new worker is created and started, running firstTask as its
        * first task. This method returns false if the pool is stopped or
        * eligible to shut down. It also returns false if the thread
        * factory fails to create a thread when asked.  If the thread
        * creation fails, either due to the thread factory returning
        * null, or due to an exception (typically OutOfMemoryError in
        * Thread.start()), we roll back cleanly.
        *
        * @param firstTask the task the new thread should run first (or
        * null if none). Workers are created with an initial first task
        * (in method execute()) to bypass queuing when there are fewer
        * than corePoolSize threads (in which case we always start one),
        * or when the queue is full (in which case we must bypass queue).
        * Initially idle threads are usually created via
        * prestartCoreThread or to replace other dying workers.
        *
        * @param core if true use corePoolSize as bound, else
        * maximumPoolSize. (A boolean indicator is used here rather than a
        * value to ensure reads of fresh values after checking other pool
        * state).
        * @return true if successful
        */
        private boolean addWorker(Runnable firstTask, boolean core) {
            retry:
            for (;;) {
                int c = ctl.get();
                int rs = runStateOf(c);

                // Check if queue empty only if necessary.
                if (rs >= SHUTDOWN &&
                    ! (rs == SHUTDOWN &&
                    firstTask == null &&
                    ! workQueue.isEmpty()))
                    return false;

                for (;;) {
                    int wc = workerCountOf(c);
                    if (wc >= CAPACITY ||
                        wc >= (core ? corePoolSize : maximumPoolSize))
                        return false;
                    if (compareAndIncrementWorkerCount(c))
                        break retry;
                    c = ctl.get();  // Re-read ctl
                    if (runStateOf(c) != rs)
                        continue retry;
                    // else CAS failed due to workerCount change; retry inner loop
                }
            }

            boolean workerStarted = false;
            boolean workerAdded = false;
            Worker w = null;
            try {
                w = new Worker(firstTask);
                final Thread t = w.thread;
                if (t != null) {
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
                    if (workerAdded) {
                        t.start();                      // 执行任务
                        workerStarted = true;
                    }
                }
            } finally {
                if (! workerStarted)
                    addWorkerFailed(w);
            }
            return workerStarted;
        }

### 方法执行操作:
检查线程池是否运行状态,添加是否超过核心工作者数与最大工作者数,都满足的话就添加(放入`workers`Set),启动新的worker并调整worker数目<br>
firstTask将作为这个worker的第一个运行任务

### 新建worker失败的情况:

1. 线程池是STOP状态或者有资格关闭(不知道什么意思,反正是非RUNNING状态就对了)
2. 线程工厂创建新线程失败(原因可能是工厂返回空值或抛出异常(可能是OutOfMemoryError))
失败后方法将会回滚并返回false, `addWorkerFailed()`方法执行回滚动作



## 内部类Worker

        /**
        * Class Worker mainly maintains interrupt control state for
        * threads running tasks, along with other minor bookkeeping.
        * This class opportunistically extends AbstractQueuedSynchronizer
        * to simplify acquiring and releasing a lock surrounding each
        * task execution.  This protects against interrupts that are
        * intended to wake up a worker thread waiting for a task from
        * instead interrupting a task being run.  We implement a simple
        * non-reentrant mutual exclusion lock rather than use
        * ReentrantLock because we do not want worker tasks to be able to
        * reacquire the lock when they invoke pool control methods like
        * setCorePoolSize.  Additionally, to suppress interrupts until
        * the thread actually starts running tasks, we initialize lock
        * state to a negative value, and clear it upon start (in
        * runWorker).
        */
        private final class Worker
            extends AbstractQueuedSynchronizer
            implements Runnable
        {
            /**
            * This class will never be serialized, but we provide a
            * serialVersionUID to suppress a javac warning.
            */
            private static final long serialVersionUID = 6138294804551838833L;

            /** Thread this worker is running in.  Null if factory fails. */
            final Thread thread;
            /** Initial task to run.  Possibly null. */
            Runnable firstTask;
            /** Per-thread task counter */
            volatile long completedTasks;

            /**
            * Creates with given first task and thread from ThreadFactory.
            * @param firstTask the first task (null if none)
            */
            Worker(Runnable firstTask) {
                setState(-1); // inhibit interrupts until runWorker
                this.firstTask = firstTask;
                this.thread = getThreadFactory().newThread(this);
            }

            /** Delegates main run loop to outer runWorker  */
            public void run() {                                      // 线程调用start()方法时实际上是调用了runWorker方法
                runWorker(this);
            }

            // Lock methods
            //
            // The value 0 represents the unlocked state.
            // The value 1 represents the locked state.

            protected boolean isHeldExclusively() {
                return getState() != 0;
            }

            protected boolean tryAcquire(int unused) {
                if (compareAndSetState(0, 1)) {
                    setExclusiveOwnerThread(Thread.currentThread());
                    return true;
                }
                return false;
            }

            protected boolean tryRelease(int unused) {
                setExclusiveOwnerThread(null);
                setState(0);
                return true;
            }

            public void lock()        { acquire(1); }
            public boolean tryLock()  { return tryAcquire(1); }
            public void unlock()      { release(1); }
            public boolean isLocked() { return isHeldExclusively(); }

            void interruptIfStarted() {
                Thread t;
                if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    }
                }
            }
        }

明显从字面就能看出worker就是执行我们任务的工人,实现`Runnable`,其本身代表一个任务
覆盖了run方法, 所以线程`addWorker()`中启动线程,实际上执行了`runWorker()`(接下来会继续展开)

### 主要的实例域:

+ thread: 用于执行任务的线程
+ firstTask: 运行的第一个任务(伴随worker实例创建的)
+ completedTasks: 这个worker完成的任务数量

Worker还扩展了`AbstractQueuedSynchronizer`(一个用于构建锁和同步器的框架)且实现了父类的模板方法,实现任务执行前后的加锁与解锁


## 线程执行的方法runWorker()

        /**
        * Main worker run loop.  Repeatedly gets tasks from queue and
        * executes them, while coping with a number of issues:
        *
        * 1. We may start out with an initial task, in which case we
        * don't need to get the first one. Otherwise, as long as pool is
        * running, we get tasks from getTask. If it returns null then the
        * worker exits due to changed pool state or configuration
        * parameters.  Other exits result from exception throws in
        * external code, in which case completedAbruptly holds, which
        * usually leads processWorkerExit to replace this thread.
        *
        * 2. Before running any task, the lock is acquired to prevent
        * other pool interrupts while the task is executing, and then we
        * ensure that unless pool is stopping, this thread does not have
        * its interrupt set.
        *
        * 3. Each task run is preceded by a call to beforeExecute, which
        * might throw an exception, in which case we cause thread to die
        * (breaking loop with completedAbruptly true) without processing
        * the task.
        *
        * 4. Assuming beforeExecute completes normally, we run the task,
        * gathering any of its thrown exceptions to send to afterExecute.
        * We separately handle RuntimeException, Error (both of which the
        * specs guarantee that we trap) and arbitrary Throwables.
        * Because we cannot rethrow Throwables within Runnable.run, we
        * wrap them within Errors on the way out (to the thread's
        * UncaughtExceptionHandler).  Any thrown exception also
        * conservatively causes thread to die.
        *
        * 5. After task.run completes, we call afterExecute, which may
        * also throw an exception, which will also cause thread to
        * die. According to JLS Sec 14.20, this exception is the one that
        * will be in effect even if task.run throws.
        *
        * The net effect of the exception mechanics is that afterExecute
        * and the thread's UncaughtExceptionHandler have as accurate
        * information as we can provide about any problems encountered by
        * user code.
        *
        * @param w the worker
        */
        final void runWorker(Worker w) {
            Thread wt = Thread.currentThread();
            Runnable task = w.firstTask;
            w.firstTask = null;
            w.unlock(); // allow interrupts
            boolean completedAbruptly = true;
            try {
                while (task != null || (task = getTask()) != null) {
                    w.lock();
                    // If pool is stopping, ensure thread is interrupted;
                    // if not, ensure thread is not interrupted.  This
                    // requires a recheck in second case to deal with
                    // shutdownNow race while clearing interrupt
                    if ((runStateAtLeast(ctl.get(), STOP) ||
                        (Thread.interrupted() &&
                        runStateAtLeast(ctl.get(), STOP))) &&
                        !wt.isInterrupted())
                        wt.interrupt();
                    try {
                        beforeExecute(wt, task);
                        Throwable thrown = null;
                        try {
                            task.run();
                        } catch (RuntimeException x) {
                            thrown = x; throw x;
                        } catch (Error x) {
                            thrown = x; throw x;
                        } catch (Throwable x) {
                            thrown = x; throw new Error(x);
                        } finally {
                            afterExecute(task, thrown);
                        }
                    } finally {
                        task = null;
                        w.completedTasks++;
                        w.unlock();
                    }
                }
                completedAbruptly = false;
            } finally {
                processWorkerExit(w, completedAbruptly);
            }
        }

!!! 未完待续