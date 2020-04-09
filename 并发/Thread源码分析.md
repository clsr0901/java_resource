### 1. 类的定义

```java
public class Thread implements Runnable
```

从类的定义中可以看出

* Thread 实现了Runnable接口

  再看看Runnable的定义

  ```java
  @FunctionalInterface
  public interface Runnable {
      public abstract void run();
  }
  ```

  * Runnable 是一个函数式接口（只有一个方法的接口）
  * Runnable只有一个run方法

### 2. 字段属性

```java 
    //线程的名字，使用volatile修饰
    private volatile String name;
	//线程的优先级
    private int            priority;
    private Thread         threadQ;
    private long           eetop;
	//当前线程是否是single_step
	private boolean     single_step;
	//当前线程是守护线程还是用户线程，默认为用户线程
	//当前JVM实例中，只要存在用户，守护线程就全部工作。如果不存在用户线程，守护线程随着JVM一同结束
	private boolean     daemon = false;
	//JVM 状态
	private boolean     stillborn = false;
	//当前线程将要运行的run，Thread实际上也是一个Runnable
	private Runnable target;
	//当前线程所在的线程组
	private ThreadGroup group;
	//当前线程中上下文对象的类加载器
	private ClassLoader contextClassLoader;
	/* The inherited AccessControlContext of this thread */
    private AccessControlContext inheritedAccessControlContext;
	//线程数量
	private static int threadInitNumber;
	//ThreadLocal 数据真实存储的容器
	//ThreadLocal 内容的副本是保存在每个线程中的ThreadLocal.ThreadLocalMap里面的
	ThreadLocal.ThreadLocalMap threadLocals = null;
	ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
	//当前线程栈的数量
	private long stackSize;
	private long nativeParkEventPointer;
	//当前线程的id
	private long tid;
	//用来生成线程id
	private static long threadSeqNumber;
	//线程当前的状态，使用volatile修饰
	private volatile int threadStatus = 0;

	volatile Object parkBlocker;
	//I/O耗时操作中断时设置interrupt status调用blocker的interrupt方法
	private volatile Interruptible blocker;
	//为同步blocker服务的对象
	private final Object blockerLock = new Object();

	//最小的线程优先级为1
	public final static int MIN_PRIORITY = 1;
	//默认的线程优先级为5
	public final static int NORM_PRIORITY = 5;
	//最大的线程优先级为10
	public final static int MAX_PRIORITY = 10;
```

### 3. 构造方法

```java 
//默认构造方法
public Thread() {
    	//调用init初始化
        init(null, null, "Thread-" + nextThreadNum(), 0);
    }

//传入一个Runnable对象
public Thread(Runnable target) {
    	//调用init初始化
        init(null, target, "Thread-" + nextThreadNum(), 0);
    }

//传入一个Runnable对象和一个AccessControlContext对象
Thread(Runnable target, AccessControlContext acc) {
    	//调用init初始化
        init(null, target, "Thread-" + nextThreadNum(), 0, acc, false);
    }

//传入一个Runnable对象和一个ThreadGroup对象
public Thread(ThreadGroup group, Runnable target) {
    	//调用init初始化
        init(group, target, "Thread-" + nextThreadNum(), 0);
    }
//传入线程的名字
public Thread(String name) {
    	//调用init初始化
        init(null, null, name, 0);
    }
//传入线程组和线程的名字
public Thread(ThreadGroup group, String name) {
    	//调用init初始化
        init(group, null, name, 0);
    }
//传入一个Runnable对象和线程的名字
public Thread(Runnable target, String name) {
    	//调用init初始化
        init(null, target, name, 0);
    }
//传入线程组、Runnable对象、线程的名字
public Thread(ThreadGroup group, Runnable target, String name) {
    	//调用init初始化
        init(group, target, name, 0);
    }
//传入线程组、Runnable对象、线程的名字，栈的大小
public Thread(ThreadGroup group, Runnable target, String name,
                  long stackSize) {
    	//调用init初始化
        init(group, target, name, stackSize);
    }
```

从构造方法可以看出

* 线程的核心是Runnable
* 构造方法都调用了init来初始化
* 线程有线程组和线程的名字
* 线程的默认名字是“Thread-”开头，后面接一串数字
* 线程初始化还可以传入栈的大小，访问控制上下文AccessControlContext对象

### 4. 方法

#### registerNatives 方法

```java
//静态native方法，确保这个方法是做的第一件事
//这个方法应该是注册到JVM之类的去，我猜的
private static native void registerNatives();
    static {
        registerNatives();
    }
```

#### nextThreadNum 方法

```java
//获取下一个线程号
//这个是静态同步方法
private static synchronized int nextThreadNum() {
    	//直接返回threadInitNumber，然后再把threadInitNumber加1
        return threadInitNumber++;
    }
```

####  nextThreadID  方法

```java
//获取下一个线程id
//这是个静态同步方法
private static synchronized long nextThreadID() {
    	//先把threadSeqNumber加1，再返回
        return ++threadSeqNumber;
    }
```

#### init 方法

```java
//初始化方法
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize) {
    	//转调下面的初始化方法
        init(g, target, name, stackSize, null, true);
    }
//这个才是Thread真正的初始化方法
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
    	//检查参数
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }
	    //设置当前线程的名称
        this.name = name;
		//获取当前调用的线程，把当前调用的线程作为父线程
        Thread parent = currentThread();
    	//获取安全控制器
        SecurityManager security = System.getSecurityManager();
        if (g == null) {
            //如果传入的线程组为null
            if (security != null) {
                //如果安全控制器不为null
                //线程组设置为当前线程的线程组
                /**
                public ThreadGroup getThreadGroup() {
        return Thread.currentThread().getThreadGroup();
    }
                */
                g = security.getThreadGroup();
            }

            if (g == null) {
                //如果安全控制器的线程组为null
                //使用父线程的线程组
                g = parent.getThreadGroup();
            }
        }
        /* checkAccess regardless of whether or not threadgroup is
           explicitly passed in. */
    	//检查线程组是否需要访问权限
        g.checkAccess();

        //检查是否有请求权限
        if (security != null) {
            if (isCCLOverridden(getClass())) {
                security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
            }
        }
		//把线程组的unstarted的线程数量加1
        g.addUnstarted();
		//设置线程组为上面获取的线程组
        this.group = g;
    	//把线程的daemon设置为父线程的damon
        this.daemon = parent.isDaemon();
    	//把线程的优先级设置为父线程的优先级
        this.priority = parent.getPriority();
    	//设置上线文类加载器
        if (security == null || isCCLOverridden(parent.getClass()))
            this.contextClassLoader = parent.getContextClassLoader();
        else
            this.contextClassLoader = parent.contextClassLoader;
        this.inheritedAccessControlContext =
                acc != null ? acc : AccessController.getContext();
    	//设置Runnable对象，为传入的Runnable对象
        this.target = target;
    	//设置线程的优先级为当前对象的优先级
        setPriority(priority);
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
    	//设置栈的大小
        this.stackSize = stackSize;
		//设置线程id		
        tid = nextThreadID();
    }
```

#### setPriority 方法

```java
//设置线程的优先级
public final void setPriority(int newPriority) {
    	//当前线程的线程组
        ThreadGroup g;
    	//检查访问权限
        checkAccess();
    	//参数校验
        if (newPriority > MAX_PRIORITY || newPriority < MIN_PRIORITY) {
            throw new IllegalArgumentException();
        }
        if((g = getThreadGroup()) != null) {
            //如果当前线程组不为null
            if (newPriority > g.getMaxPriority()) {
                //传入的新优先级大于线程组最大的优先级
                //把新的优先级设置为线程组最大的优先级
                newPriority = g.getMaxPriority();
            }
            //设置线程的优先级
            setPriority0(priority = newPriority);
        }
    }
//真正的设置线程优先级方法，这个是native方法
private native void setPriority0(int newPriority);
```

#### getPriority 方法

```java
//获取当前线程的优先级
public final int getPriority() {
        return priority;
    }
```

#### currentThread 方法

```java
//获取当前的线程，这个是native方法
public static native Thread currentThread();
```

#### yield 方法

```java
//暗示当前进程的调度器让出当前使用的进程，调度器可以自由去忽略这个暗示
//这个有可能是当前线程放弃执行，然后又抢到Cpu的资源继续执行
//yield让当前线程进入到就绪状态，但是自己还可以继续竞争资源
public static native void yield();
```

#### sleep 方法

```java
//让当前线程休眠多少毫米
//当前线程不会让出对象的锁让其他线程执行
public static native void sleep(long millis) throws InterruptedException;

public static void sleep(long millis, int nanos)
    throws InterruptedException {
        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }
        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }
        if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
            millis++;
        }
    	//最终调用的还是上面的方法，只不过把纳秒改成了毫米
        sleep(millis);
    }

```

#### clone 方法

```java
//不支持克隆，直接抛出异常
//这种方式要学习，JDK中有很多这种操作。子类不支持父类方法，直接抛出异常
@Override
    protected Object clone() throws CloneNotSupportedException {
        throw new CloneNotSupportedException();
    }
```

#### start 方法

```java 
//开始一个线程
//只是把线程的状态从NEW变成了Runnable状态
//并不一定立即执行，要看能不能抢到cpu的资源
public synchronized void start() {
        if (threadStatus != 0)
            //NEW状态为0
            //如果当前线程不是NEW状态，抛出异常
            throw new IllegalThreadStateException();

        //唤醒线程组，把当前线程加入到线程组的线程数组中去
    	//把线程组的线程数量加1，并把线程组的unstarted的线程数量减1
        group.add(this);
		//设置启动状态为false
        boolean started = false;
        try {
            //真正的启动线程方法，这个是一个native方法
            start0();
            //启动线程成功，把启动状态置为true
            started = true;
        } finally {
            try {
                if (!started) {
                    //如果启动失败，通知线程组做出相应的操作
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
               
            }
        }
    }
	//真正启动线程的方法，是一个native方法
	//可以看出启动线程操作是JVM去处理的
    private native void start0();
```

#### run 方法

```java
//执行具体业务的方法，这个应该是JVM底层回调
@Override
    public void run() {
        //直接交给Runnable对象去执行
        if (target != null) {
            target.run();
        }
    }
```

#### exit 方法

```java 
//在Thread退出之前，系统调用exit来回收资源
private void exit() {
        if (group != null) {
            //如果线程组不为null
            //唤醒线程组的其他线程
            group.threadTerminated(this);
            group = null;
        }
        /* Aggressively null out all reference fields: see bug 4006245 */
        target = null;
        /* Speed the release of some of these resources */
        threadLocals = null;
        inheritableThreadLocals = null;
        inheritedAccessControlContext = null;
        blocker = null;
        uncaughtExceptionHandler = null;
    }
```

#### stop 方法

```java 
//强制停止当前线程
public final void stop() {
    	//获取安全管理器
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            //如果安全管理器不为null
            //检查权限
            checkAccess();
            if (this != Thread.currentThread()) {
                //如果不是当前线程自己调用
                //检查权限
                security.checkPermission(SecurityConstants.STOP_THREAD_PERMISSION);
            }
        }
        if (threadStatus != 0) {
            //如果当前线程的状态不是NEW，可能处于暂停状态
            //调用resume方法恢复一个暂停的线程
            resume(); // Wake up thread if it was suspended; no-op otherwise
        }

        //VM可以操作所有的状态
    	//通过stop0来执行
        stop0(new ThreadDeath());
    }
//强制停止当前线程，这个是一个native方法
private native void stop0(Object o);
```

#### resume 方法

```java
//恢复一个暂停的线程，如果线程没有暂停，不做其他操作
@Deprecated
    public final void resume() {
        //检查权限
        checkAccess();
        //调用reume0来恢复暂停的线程
        resume0();
    }
//恢复暂停的线程，这个是一个nateive方法
private native void resume0();
```

#### interrupt 方法

```java
//中断线程
public void interrupt() {
        if (this != Thread.currentThread())
            //如果不是当前线程自己调用
            //检查权限
            checkAccess();
		//同步blockerLock
    	//同步代码块里包含blocker，因为blockerLock就是为了同步blocker
        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                //如果blocker不为null，即现在处于I/O阻塞状态
                //仅仅设置interrupt状态
                interrupt0();           
                //设置interrupt status后调用blocker的interrup方法
                b.interrupt(this);
                return;
            }
        }
    	//设置interrupt status
        interrupt0();
    }
//设置interrupt status 这是一个native方法
private native void interrupt0();
```

#### interrupted 方法

```java
//测试调用线程是否处于中断状态，这是一个静态方法
public static boolean interrupted() {
    	//调用当前线程的isInterrupted方法来测试
        return currentThread().isInterrupted(true);
    }
```

#### isInterrupted 方法

```java
//测试当前线程是否处于中断状态 
public boolean isInterrupted() {
    	//直接调用isInterrupted方法
        return isInterrupted(false);
    }
//测试线程是否中断，这是一个native方法
//ClearInterrupted true 会重置interrupted state, false 不会重置interrupted state
 private native boolean isInterrupted(boolean ClearInterrupted);
```

#### suspend 方法

```java
//暂停线程
@Deprecated
    public final void suspend() {
        //检查权限
        checkAccess();
        //调用susupend0来暂停线程
        suspend0();
    }
//暂停线程，这是一个native方法
private native void suspend0();
```

#### setName 方法

```java
//设置线程名称，这是一个同步方法
public final synchronized void setName(String name) {
    	//检查权限
        checkAccess();
    	//参数检查
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }
		//为name属性赋值
        this.name = name;
        if (threadStatus != 0) {
            //如果当前线程不是NEW状态，调用setNativeName方法
            setNativeName(name);
        }
    }
//设置线程名称
private native void setNativeName(String name);
```

#### getName 方法

```java
//获取线程名称
public final String getName() {
        return name;
    }
```

#### getThreadGroup 方法

```java
//获取线程的线程组
public final ThreadGroup getThreadGroup() {
        return group;
    }
```

#### activeCount 方法

```java
//返回当前线程组和子线程组，活动的线程数，返回的是一个估计值
public static int activeCount() {
        return currentThread().getThreadGroup().activeCount();
    }
```

#### enumerate 方法

```java
//把当前线程组和子线程组的活动的线程拷贝到指定的数组
public static int enumerate(Thread tarray[]) {
        return currentThread().getThreadGroup().enumerate(tarray);
    }
```

#### join 方法

```java
//等待这个线程结束后再执行
public final void join() throws InterruptedException {
    	//调用下面的方法
        join(0);
    }
//等待指定最大时间或者这个线程执行结束再执行
public final synchronized void join(long millis, int nanos)
    throws InterruptedException {
        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }
        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }
        if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
            millis++;
        }
    	//上面是把纳秒转换为毫秒
    	//调用下面的方法
        join(millis);
    }
//这个才是真正的join方法
//等待指定毫秒数，或者当前线程执行完毕后再截止执行调用线程
public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;
        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }
        if (millis == 0) {
            //不指定等待时间的情况
            //一直while循环，直到当前线程结束再执行调用线程
            while (isAlive()) {
                wait(0);
            }
        } else {
            //指定等待时间的情况
            //while循环判断当前线程是否结束
            while (isAlive()) {
                long delay = millis - now;
                //如果等待时间超过指定时间退出，继续执行调用线程
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
```

#### setDaemon 方法

```java
//设置线程是守护线程还是用户线程
//必须在start之前调用
public final void setDaemon(boolean on) {
    	//检查权限
        checkAccess();
    	//如果线程在运行抛出异常
        if (isAlive()) {
            throw new IllegalThreadStateException();
        }
    	//设置线程标记
        daemon = on;
    }
```

#### isDaemon 方法

```java
//测试当前线程是用户线程还是守护线程
public final boolean isDaemon() {
        return daemon;
    }
```

#### checkAccess 方法

```java
//权限检查
//判断当前运行线程（调用线程）是否有权限去修改这个线程
public final void checkAccess() {
    	//获取安全管理器
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            //如果安全管理器不为null
            //调用安全管理器的checkAccess方法
            security.checkAccess(this);
        }
    }
```

#### toString 方法

```java
//可以看出thread 的toString方法有线程名称、优先级、线程组名称
public String toString() {
        ThreadGroup group = getThreadGroup();
        if (group != null) {
            return "Thread[" + getName() + "," + getPriority() + "," +
                           group.getName() + "]";
        } else {
            return "Thread[" + getName() + "," + getPriority() + "," +
                            "" + "]";
        }
    }
```

#### getId 方法

```java
//获取线程id
public long getId() {
        return tid;
    }
```

### 5. 内部类

####  线程的状态

```java
public enum State {
        /**
         * Thread state for a thread which has not yet started.
         */
    	//初始状态，刚被new出来，还没调用start时的状态
        NEW,

        /**
         * Thread state for a runnable thread.  A thread in the runnable
         * state is executing in the Java virtual machine but it may
         * be waiting for other resources from the operating system
         * such as processor.
         */
    	//运行时状态，或者就绪时状态，调用start后的状态
        RUNNABLE,

        /**
         * Thread state for a thread blocked waiting for a monitor lock.
         * A thread in the blocked state is waiting for a monitor lock
         * to enter a synchronized block/method or
         * reenter a synchronized block/method after calling
         * {@link Object#wait() Object.wait}.
         */
    	//等待获取对象锁时的状态
        BLOCKED,

        /**
         * Thread state for a waiting thread.
         * A thread is in the waiting state due to calling one of the
         * following methods:
         * <ul>
         *   <li>{@link Object#wait() Object.wait} with no timeout</li>
         *   <li>{@link #join() Thread.join} with no timeout</li>
         *   <li>{@link LockSupport#park() LockSupport.park}</li>
         * </ul>
         *
         * <p>A thread in the waiting state is waiting for another thread to
         * perform a particular action.
         *
         * For example, a thread that has called <tt>Object.wait()</tt>
         * on an object is waiting for another thread to call
         * <tt>Object.notify()</tt> or <tt>Object.notifyAll()</tt> on
         * that object. A thread that has called <tt>Thread.join()</tt>
         * is waiting for a specified thread to terminate.
         */
    	//等待被人唤醒状态时状态
        WAITING,

        /**
         * Thread state for a waiting thread with a specified waiting time.
         * A thread is in the timed waiting state due to calling one of
         * the following methods with a specified positive waiting time:
         * <ul>
         *   <li>{@link #sleep Thread.sleep}</li>
         *   <li>{@link Object#wait(long) Object.wait} with timeout</li>
         *   <li>{@link #join(long) Thread.join} with timeout</li>
         *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
         *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
         * </ul>
         */
    	//等待别人唤醒，但设置了最长等待时间的状态
        TIMED_WAITING,

        /**
         * Thread state for a terminated thread.
         * The thread has completed execution.
         */
    	//run方法运行结束的状态
        TERMINATED;
    }
```