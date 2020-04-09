### 1. 类的定义

```java
public class ThreadPoolExecutor extends AbstractExecutorService
```

从类的定义可以看出

* ThreadPoolExecutor继承了AbstractExecutorService类

* AbstractExecutorService 定义

  ```java
  public abstract class AbstractExecutorService implements ExecutorService
  ```

  从AbstractExecutorService定义中可以看出

  * AbstractExecutorService 实现了ExecutorService接口

* ThreadPoolExecutor 也是ExecutorService的实现类

### 2. 字段属性

```java
//主池控制状态，CTL，是一种原子整数包装的两种概念领域工人计数，表明有效数量的线程运行状态，指示是否在运行，关闭等
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
//容量2的29次方-1
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
   / *   RUNNING:  Accept new tasks and process queued tasks
       			  接收新的任务并且处理工作队列的任务
     *   SHUTDOWN: Don't accept new tasks, but process queued tasks
         		   不接受新任务，但是会处理工作队列中的任务
     *   STOP:     Don't accept new tasks, don't process queued tasks,
     *             and interrupt in-progress tasks
         		  不接受新任务，也不会处理工作队列中的任务，而且会中断正在执行的任务
     *   TIDYING:  All tasks have terminated, workerCount is zero,
     *             the thread transitioning to state TIDYING
     *             will run the terminated() hook method
         		   所有任务都执行完毕，工作线程数量为0，将会调用terminated方法
     *   TERMINATED: terminated() has completed
         			terminated 方法执行完
     *
     * The numerical order among these values matters, to allow
     * ordered comparisons. The runState monotonically increases over
     * time, but need not hit each state. The transitions are:
     *
     * RUNNING -> SHUTDOWN
     *    On invocation of shutdown(), perhaps implicitly in finalize()
     * (RUNNING or SHUTDOWN) -> STOP
     *    On invocation of shutdownNow()
     * SHUTDOWN -> TIDYING
     *    When both queue and pool are empty
     * STOP -> TIDYING
     *    When pool is empty
     * TIDYING -> TERMINATED
     *    When the terminated() hook method has completed
     */
//runState is stored in the high-order bits; 运行状态存储在高位
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

//工作队列，用来保存任务
private final BlockingQueue<Runnable> workQueue;
//锁
private final ReentrantLock mainLock = new ReentrantLock();
//Set containing all worker threads in pool. Accessed only when holding mainLock.
//工作线程集合
private final HashSet<Worker> workers = new HashSet<Worker>();
//mainLock的条件
private final Condition termination = mainLock.newCondition();
//池中存在的最大线程数
private int largestPoolSize;
//统计完成的任务数量
private long completedTaskCount;
//创建线程的工厂
private volatile ThreadFactory threadFactory;
//拒绝处理策略，saturated or shutdown的时候执行
private volatile RejectedExecutionHandler handler;
//超过核心线程容量的线程空闲保活时间
private volatile long keepAliveTime
//false（默认）：核心线程活着即使处于空闲状态。 true：核心线程使用存活时间超时等待工作    
private volatile boolean allowCoreThreadTimeOut;
//核心线程容量
private volatile int corePoolSize;
//最大线程容量
private volatile int maximumPoolSize;
//默认拒绝策略
private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();
//运行时权限，主要针对shutdown and shutdownNow方法
private static final RuntimePermission shutdownPerm =
        new RuntimePermission("modifyThread");
/* The context to be used when executing the finalizer, or null. */
//这个上下文被用来执行finalizer方法，可以为null
private final AccessControlContext acc;
```

从字段属性中可以看出

* 线程池包含了线程工厂、核心线程数量、最大线程数量、工作队列、拒绝策略（这几个最重要）
* 线程池持有ReentrantLock锁，用来控制访问线程池对象
* 线程池有五个运行状态（生命周期）

### 3. 构造方法

```java
//传入核心线程数量、最大线程数量、保活时间、保活时间单位、工作队列
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
    	//转调最后一个构造方法
    	//这个构造函数使用默认的拒绝策略
    	//线程工厂使用的是Executors默认的线程工厂
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
//传入核心线程数量、最大线程数量、保活时间、保活时间单位、工作队列、线程工厂
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory) {
    	//转调最后一个构造方法
    	//这个构造函数使用默认的拒绝策略
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             threadFactory, defaultHandler);
    }
//传入核心线程数量、最大线程数量、保活时间、保活时间单位、工作队列、拒绝策略
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              RejectedExecutionHandler handler) {
    	//转调最后一个构造方法
    	//这个构造方法线程工厂使用的是Executors默认的线程工厂
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), handler);
    }
//传入核心线程数量、最大线程数量、保活时间、保活时间单位、工作队列、线程工厂、拒绝策略
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
    	//参数校验
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
    	//设置核心线程数量
        this.corePoolSize = corePoolSize;
    	//设置对打线程数量
        this.maximumPoolSize = maximumPoolSize;
    	//设置工作队列
        this.workQueue = workQueue;
    	//设置保活时间
        this.keepAliveTime = unit.toNanos(keepAliveTime);
    	//设置线程工厂
        this.threadFactory = threadFactory;
    	//设置拒绝策略
        this.handler = handler;
    }
```

从构造方法可以看出

* 线程池最核心的部分是 核心线程数，最大线程数，保活时间、工作队列
* 线程池的线程工厂和拒绝策略都有默认的实现
* Executors提供了几个默认线程池的实现，不过都存在弊端。阿里的java规范文档建议手动创建线程池

### 4. 方法

####  execute 方法

```java
//执行一个任务，没有返回值
public void execute(Runnable command) {
    	//参数检查
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
    	//获取控制状态值
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            //如果当前线程数量小于核心线程数量
            if (addWorker(command, true))
                //试图创建一个新的核心线程去执行这个任务
                //成功直接返回
                return;
            //失败，重新获取控制状态值
            c = ctl.get();
        }
    	//核心线程满了，先往工作队列中添加，后续线程会从队列中获取任务执行
        if (isRunning(c) && workQueue.offer(command)) {
            //如果线程池在运行，并且任务添加队列成功
            //进行double-check，以防上次检测后有线程结束
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                //如果线程池已经停止并且从工作队列中移除任务成功
                //使用拒绝策略拒绝该任务
                reject(command);
            else if (workerCountOf(recheck) == 0)
                //如果线程数量为0，添加新的工作线程，防止检查的时候所有线程都销毁了
                addWorker(null, false);
        }
    	//如果任务添加到工作队列失败（工作队列满了的情况）
    	//尝试创建一个新的线程处理当前任务
    	//如果失败采取拒绝策略处理当前任务
        else if (!addWorker(command, false))
            reject(command);
    }
```

#### addWorker 方法

```java
//添加工作线程
//firstTask null：表示只创建工作线程 非null：表示创建一个线程，并执行firstTask
//core 创建的线程是否为核心线程
private boolean addWorker(Runnable firstTask, boolean core) {
    	//goto坐标，重试
        retry:
    	//无限for循环
        for (;;) {
            //获取控制状态值
            int c = ctl.get();
            //获取运行状态
            int rs = runStateOf(c);
            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                //如果已经停止或者队列为空，直接返回false
                return false;
			//无限for循环
            for (;;) {
                //获取工作线程数量
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    //如果工作线程数量已经达到最大，直接返回false
                    //core 控制是否跟核心线程数比
                    return false;
                //CAS 增加工作线程的数量
                if (compareAndIncrementWorkerCount(c))
                    //设置成功，跳出循环，接着往下执行
                    break retry;
                //重新获取控制状态值
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    //如果状态改变，跳出循环，接着往下执行
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
                //如果CAS失败，内层循环重新进行CAS
            }
        }
		//线程运行标志
        boolean workerStarted = false;
    	//工作线程添加标志
        boolean workerAdded = false;
        Worker w = null;
        try {
            //传入任务，创建一个新的工作线程
            w = new Worker(firstTask);
            //获取新创建工作线程的线程
            final Thread t = w.thread;
            if (t != null) {
                //如果线程不为null，加锁
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    //获取当前状态
                    int rs = runStateOf(ctl.get());
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        //如果当前线程池没有被关掉，或者线程池已经被关掉但是任务为null
                        if (t.isAlive()) // precheck that t is startable
                            //如果线程已经运行，抛出异常
                            throw new IllegalThreadStateException();
                        //工作线程集合添加新创建的工作线程
                        workers.add(w);
                        //重新设置已存在线程的最大数量
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        //设置添加成功
                        workerAdded = true;
                    }
                } finally {
                    //释放锁
                    mainLock.unlock();
                }
                if (workerAdded) {
                    //如果工作线程添加成功
                    //启动线程
                    t.start();
                    //线程运行成功
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                //如果线程运行失败，执行addWorkerFailed方法
                addWorkerFailed(w);
        }
    	//返回线程运行标志
        return workerStarted;
    }
```

#### addWorkerFailed 方法

```java
//添加工作线程失败
private void addWorkerFailed(Worker w) {
    	//加锁
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                //如果添加失败的工作线程不为null
                //从工作线程集合中移除
                workers.remove(w);
            //使用CAS把工作线程数量减1
            decrementWorkerCount();
            //尝试终止线程池
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }
```

####  reject 方法

```java
//拒绝添加的任务
final void reject(Runnable command) {
    	//调用设置的拒绝策略
        handler.rejectedExecution(command, this);
    }
```

#### remove 方法

```java
//移除指定任务
public boolean remove(Runnable task) {
    	//从工作队列中移除任务
        boolean removed = workQueue.remove(task);
    	//尝试关闭线程池
        tryTerminate(); // In case SHUTDOWN and now empty
    	//返回是否移除成功
        return removed;
    }

```

#### tryTerminate 方法

```java
//尝试终止线程池
final void tryTerminate() {
    	//无限for循环
        for (;;) {
            //获取控制状态值
            int c = ctl.get();
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                //如果是运行状态 或者 状态大于TIDYING 或者 关闭状态并且工作队列不为null
                //直接返回
                return;
            if (workerCountOf(c) != 0) { // Eligible to terminate
                //如果工作线程数量不等于0
                //中断线程池中的线程，最多只中断一个线程
                interruptIdleWorkers(ONLY_ONE);
                return;
            }
			//上锁
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                //CAS操作，把状态置为TIDYING
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        //设置成功，调用terminated
                        terminated();
                    } finally {
                        //最后把状态设置为TERMINATED
                        ctl.set(ctlOf(TERMINATED, 0));
                        //唤醒所有等待的线程
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                //解锁
                mainLock.unlock();
            }
            // else retry on failed CAS
            //如果CAS失败，继续循环CAS操作
        }
    }

```

#### interruptIdleWorkers 方法

```java
//中断线程池中的线程
private void interruptIdleWorkers() {
    	//调用下面的方法
        interruptIdleWorkers(false);
    }
//中断线程池中的线程 onlyOne true 最多只中断一个 false 全部中断
private void interruptIdleWorkers(boolean onlyOne) {
    	//上锁
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //循环遍历工作线程集合
            for (Worker w : workers) {
                //获取工作线程的线程
                Thread t = w.thread;
                //如果线程没有中断 
                //尝试给工作线程上锁
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        //中断线程
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        //工作线程解锁
                        w.unlock();
                    }
                }
                //如果只操作一个工作线程，跳出循环
                if (onlyOne)
                    break;
            }
        } finally {
            //解锁
            mainLock.unlock();
        }
    }

```

#### shutdown 方法

```java
//关闭线程池
public void shutdown() {
    	//加锁
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //检查shutdown访问权限
            //循环遍历工作线程集合，针对每个工作线程检查访问权限
            checkShutdownAccess();
            //使用CAS把状态修改为SHUTDOWN
            advanceRunState(SHUTDOWN);
            //中断所有线程
            interruptIdleWorkers();
            //钩子函数，这里没有实现
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            //解锁
            mainLock.unlock();
        }
    	//尝试终止线程池
        tryTerminate();
    }

```

#### shutdownNow 方法

```java
//关闭线程池，并获取线程池的所有任务
public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
    	//加锁
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //检查shutdown访问权限
            //循环遍历工作线程集合，针对每个工作线程检查访问权限
            checkShutdownAccess();
            //使用CAS把状态修改为STOP
            advanceRunState(STOP);
            //中断所有线程
            interruptWorkers();
            //把工作队列的元素导入集合中
            tasks = drainQueue();
        } finally {
            //解锁
            mainLock.unlock();
        }
    	//尝试终止线程池
        tryTerminate();
        return tasks;
    }
```

#### advanceRunState 方法

```java
//预设置运行状态
private void advanceRunState(int targetState) {
    	//for 无限虚幻
        for (;;) {
            //获取控制状态值
            int c = ctl.get();
            //使用CAS把状态设置为目标状态
            if (runStateAtLeast(c, targetState) ||
                ctl.compareAndSet(c, ctlOf(targetState, workerCountOf(c))))
                //设置成功，中断循环
                break;
        }
    }
```

#### drainQueue 方法

```java
//把工作队列的元素导入集合中
private List<Runnable> drainQueue() {
    	//拷贝工作队列副本
        BlockingQueue<Runnable> q = workQueue;
        ArrayList<Runnable> taskList = new ArrayList<Runnable>();
    	//把q中所有的元素添加到taskList中
        q.drainTo(taskList);
    	//上面操作可能出错，所以还需要再判断一次
        if (!q.isEmpty()) {
            //如果工作队列不是空的
            for (Runnable r : q.toArray(new Runnable[0])) {
                //循环遍历q，并将q的元素添加到taskList中
                if (q.remove(r))
                    taskList.add(r);
            }
        }
    	//返回taskList
        return taskList;
    }
```

####  isShutdown 方法

```java
//判断线程池是否关闭
public boolean isShutdown() {
    	//使用状态控制值来判断
        return ! isRunning(ctl.get());
    }
```

####  isTerminating 方法

```java
//判断线程池是否在终止中
public boolean isTerminating() {
    	//获取状态控制值
        int c = ctl.get();
    	//使用状态控制值来判断
        return ! isRunning(c) && runStateLessThan(c, TERMINATED);
    }
```

#### isTerminated 方法

```java
//判断线程池是否已终止
public boolean isTerminated() {
    	//使用状态控制值来判断
        return runStateAtLeast(ctl.get(), TERMINATED);
    }
```

#### awaitTermination 方法

```java
//设置等待时间终止线程池
public boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException {
        long nanos = unit.toNanos(timeout);
    	//上锁
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //for 无限循环
            for (;;) {
                //使用CAS设置为终止状态
                if (runStateAtLeast(ctl.get(), TERMINATED))
                    //成功返回true
                    return true;
                if (nanos <= 0)
                    //超时，返回false
                    return false;
                nanos = termination.awaitNanos(nanos);
            }
        } finally {
            //解锁
            mainLock.unlock();
        }
    }
```

#### finalize 方法

```java
//对象被销毁时调用方法
protected void finalize() {
    	//获取安全控制管理器
        SecurityManager sm = System.getSecurityManager();
        if (sm == null || acc == null) {
            //安全控制管理器为null 或者 访问控制上下文为null
            //直接关闭线程池
            shutdown();
        } else {
            //使用AccessController来操作
            PrivilegedAction<Void> pa = () -> { shutdown(); return null; };
            AccessController.doPrivileged(pa, acc);
        }
    }
```

####  setThreadFactory 方法

```java
//设置线程工厂
public void setThreadFactory(ThreadFactory threadFactory) {
    	//参数检查
        if (threadFactory == null)
            throw new NullPointerException();
    	//设置线程工厂
        this.threadFactory = threadFactory;
    }
```

#### getThreadFactory 方法

```java
//获取线程工厂
public ThreadFactory getThreadFactory() {
        return threadFactory;
    }
```

#### setRejectedExecutionHandler 方法

```java
//设置拒绝策略
public void setRejectedExecutionHandler(RejectedExecutionHandler handler) {
        if (handler == null)
            throw new NullPointerException();
        this.handler = handler;
    }
```

####  getRejectedExecutionHandler 方法

```java
//获取拒绝策略
public RejectedExecutionHandler getRejectedExecutionHandler() {
        return handler;
    }
```

#### setCorePoolSize 方法

```java
//设置核心线程数量
public void setCorePoolSize(int corePoolSize) {
    	//参数检查
        if (corePoolSize < 0)
            throw new IllegalArgumentException();
    	//获取差值
        int delta = corePoolSize - this.corePoolSize;
    	//赋值
        this.corePoolSize = corePoolSize;
        if (workerCountOf(ctl.get()) > corePoolSize)
            //如果设置的核心线程数小于当前线程数
            //中断线程池中所有的线程
            interruptIdleWorkers();
        else if (delta > 0) {
            //如果设置的核心线程数比原来的大
            //取差值和工作队列长度的最小值
            int k = Math.min(delta, workQueue.size());
            //使用while循环添加工作线程
            while (k-- > 0 && addWorker(null, true)) {
                if (workQueue.isEmpty())
                    //如果工作队列为空，退出循环
                    break;
            }
        }
    }
```

#### getCorePoolSize 方法

```java
//获取核心线程数
public int getCorePoolSize() {
        return corePoolSize;
    }
```

#### prestartCoreThread 方法

```java
//预启动核心线程
public boolean prestartCoreThread() {
    	//如果当前线程数量小于核心线程数量 才能创建核心线程
        return workerCountOf(ctl.get()) < corePoolSize &&
            addWorker(null, true);
    }
```

#### ensurePrestart 方法

```java
//确保预启动核心线程，至少启动一个
void ensurePrestart() {
    	//获取线程数
        int wc = workerCountOf(ctl.get());
        if (wc < corePoolSize)
            //如果线程数比核心线程数少，启动一个核心线程
            addWorker(null, true);
        else if (wc == 0)
            //如果线程数为0，启动一个线程
            addWorker(null, false);
    }
```

#### prestartAllCoreThreads 方法

```java
//预启动所有的核心线程
public int prestartAllCoreThreads() {
        int n = 0;
    	//while循环启动核心线程，直到达到核心线程数量
        while (addWorker(null, true))
            ++n;
    	//返回方法启动线程的数量
        return n;
    }
```

#### allowsCoreThreadTimeOut 方法

```java
//核心线程是否有超时时间，返回true 到达超时时间核心线程被销毁 false 核心线程不会被销毁
public boolean allowsCoreThreadTimeOut() {
        return allowCoreThreadTimeOut;
    }
```

#### allowCoreThreadTimeOut 方法

```java
//设置核心线程是否超时
public void allowCoreThreadTimeOut(boolean value) {
        if (value && keepAliveTime <= 0)
            //如果允许超时，超时时间必须大于0
            throw new IllegalArgumentException("Core threads must have nonzero keep alive times");
        if (value != allowCoreThreadTimeOut) {
            //如果设置的状态跟以前的相反
            //设置新值
            allowCoreThreadTimeOut = value;
            if (value)
                //如果允许超时，中断所有线程
                interruptIdleWorkers();
        }
    }
```

#### setMaximumPoolSize 方法

```java
//设置线程池最大线程数
public void setMaximumPoolSize(int maximumPoolSize) {
    	//参数检查
        if (maximumPoolSize <= 0 || maximumPoolSize < corePoolSize)
            throw new IllegalArgumentException();
    	//设置新值
        this.maximumPoolSize = maximumPoolSize;
        if (workerCountOf(ctl.get()) > maximumPoolSize)
            //如果当前线程数大于新设置的最大线程数
            //中断所有线程
            interruptIdleWorkers();
    }
```

#### getMaximumPoolSize 方法

```java
//获取线程池最大线程数 
public int getMaximumPoolSize() {
        return maximumPoolSize;
    }
```

####  setKeepAliveTime 方法

```java
//设置保活时间
public void setKeepAliveTime(long time, TimeUnit unit) {
    	//参数检查
        if (time < 0)
            throw new IllegalArgumentException();
        if (time == 0 && allowsCoreThreadTimeOut())
            throw new IllegalArgumentException("Core threads must have nonzero keep alive times");
    	//转换为纳秒
        long keepAliveTime = unit.toNanos(time);
    	//获取差值
        long delta = keepAliveTime - this.keepAliveTime;
    	//设置新值
        this.keepAliveTime = keepAliveTime;
        if (delta < 0)
            //如果新设置的时间小于原来的时间
            //中断所有线程
            interruptIdleWorkers();
    }
```

#### getKeepAliveTime 方法

```java
//获取保活时间
public long getKeepAliveTime(TimeUnit unit) {
        return unit.convert(keepAliveTime, TimeUnit.NANOSECONDS);
    }
```

#### getQueue 方法

```java
//获取工作队列
public BlockingQueue<Runnable> getQueue() {
        return workQueue;
    }
```

####  purge 方法

```java
//移除工作队列中已取消的任务
public void purge() {
    	//拷贝工作队列
        final BlockingQueue<Runnable> q = workQueue;
        try {
            //获取工作队列的迭代器
            Iterator<Runnable> it = q.iterator();
            //使用while进行迭代
            while (it.hasNext()) {
                Runnable r = it.next();
                if (r instanceof Future<?> && ((Future<?>)r).isCancelled())
                    //如果当前迭代任务属于Future 并且已经取消
                    //把当前迭代任务从工作队列中移除
                    it.remove();
            }
        } catch (ConcurrentModificationException fallThrough) {
            //如果上面的操作遇到异常，使用下面的方式来操作，性能低于上面
            //把工作队列转换为数组并进行遍历
            for (Object r : q.toArray())
                if (r instanceof Future<?> && ((Future<?>)r).isCancelled())
                    //如果当前迭代任务属于Future 并且已经取消
                    //从工作队列中移除当前迭代任务
                    q.remove(r);
        }
		//尝试终止线程池
        tryTerminate(); // In case SHUTDOWN and now empty
    }
```

#### getPoolSize 方法

```java
//获取线程数量
public int getPoolSize() {
    	//上锁
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // Remove rare and surprising possibility of
            // isTerminated() && getPoolSize() > 0
            //如果当前的状态大于TIDYING 返回0
            //其它 返回工作线程集合的大小
            return runStateAtLeast(ctl.get(), TIDYING) ? 0
                : workers.size();
        } finally {
            //解锁
            mainLock.unlock();
        }
    }
```

#### getActiveCount 方法

```java
//获取正在执行任务的线程数量，这个是预估值，不准确
public int getActiveCount() {
    	//上锁
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            int n = 0;
            //遍历工作线程集合
            for (Worker w : workers)
                if (w.isLocked())
                    //如果工作线程被锁定，表示正在执行任务
                    ++n;
            //返回统计的数量
            return n;
        } finally {
            mainLock.unlock();
        }
    }
```

#### getLargestPoolSize 方法

```java
//获取池中存在的最大线程数
public int getLargestPoolSize() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            return largestPoolSize;
        } finally {
            mainLock.unlock();
        }
    }
```

#### getTaskCount 方法

```java
//返回任务总数（包含已执行完成的、正在执行的、未执行的）
public long getTaskCount() {
    	//上锁
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //已执行完成的数量作为初始值
            long n = completedTaskCount;
            //for循环遍历工作线程集合
            for (Worker w : workers) {
                //加上每个工作线程执行完成的任务数量
                n += w.completedTasks;
                if (w.isLocked())
                    //如果当前现在正在执行
                    //统计加1
                    ++n;
            }
            //返回上面统计加上任务队列的长度
            return n + workQueue.size();
        } finally {
            //解锁
            mainLock.unlock();
        }
    }
```

#### getCompletedTaskCount 方法

```java
//获取已完成任务数量
public long getCompletedTaskCount() {
    	//上锁
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //已执行完成的数量作为初始值
            long n = completedTaskCount;
            //for循环遍历工作线程集合
            for (Worker w : workers)
                //加上每个工作线程执行完成的任务数量
                n += w.completedTasks;
            //返回统计数量
            return n;
        } finally {
            //解锁
            mainLock.unlock();
        }
    }
```

#### toString 方法

```java
public String toString() {
        long ncompleted;
        int nworkers, nactive;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            ncompleted = completedTaskCount;
            nactive = 0;
            nworkers = workers.size();
            for (Worker w : workers) {
                ncompleted += w.completedTasks;
                if (w.isLocked())
                    ++nactive;
            }
        } finally {
            mainLock.unlock();
        }
        int c = ctl.get();
        String rs = (runStateLessThan(c, SHUTDOWN) ? "Running" :
                     (runStateAtLeast(c, TERMINATED) ? "Terminated" :
                      "Shutting down"));
    	//返回了线程池状态、线程数量、正在执行任务的线程数量、工作队列中任务的数量、已经完成的任务数量
        return super.toString() +
            "[" + rs +
            ", pool size = " + nworkers +
            ", active threads = " + nactive +
            ", queued tasks = " + workQueue.size() +
            ", completed tasks = " + ncompleted +
            "]";
    }
```

#### submit 方法

```java
//父类中的方法，提交任务并指定返回值类型
public <T> Future<T> submit(Runnable task, T result) {
    	//参数检查
        if (task == null) throw new NullPointerException();
    	//把task包装成一个RunnableFuture对象
        RunnableFuture<T> ftask = newTaskFor(task, result);
    	//调用excute方法
        execute(ftask);
    	//返回包装后的对象
        return ftask;
    }
//父类中的方法，提交任务
public <T> Future<T> submit(Callable<T> task) {
    	//参数检查
        if (task == null) throw new NullPointerException();
    	//把task包装成一个RunnableFuture对象
        RunnableFuture<T> ftask = newTaskFor(task);
    	//调用excute方法
        execute(ftask);
    	//返回包装后的对象
        return ftask;
    }
```

#### newTaskFor 方法

```java
//把Callable包装成一个RunnableFuture对象
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
//把Callable包装成一个RunnableFuture对象，并且指定类型
 protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }
```

