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
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }