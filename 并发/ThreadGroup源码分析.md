###  1. 类的定义

```java
public class ThreadGroup implements Thread.UncaughtExceptionHandler
```

从类的定义中可以看出

* ThreadGroup实现了Thread.UncaughtExceptionHandler接口

#### JDK介绍

>  A thread group represents a set of threads. In addition, a thread group can also include other thread groups. The thread groups form a tree in which every thread group except the initial thread group has a parent.
>
> 线程组代表一组线程。此外，线程组还可以包括其他线程组。线程组形成一棵树，其中除初始线程组之外的每个线程组都有一个父级
>
> A thread is allowed to access information about its own thread group, but not to access information about its thread group's parent thread group or any other thread groups.
>
> 允许线程访问有关其自己的线程*组的信息，但不允许访问有关其线程组的*父线程组或任何其他线程组的信息

### 2. 字段属性

```java
//父线程组
private final ThreadGroup parent;
//线程组名称
String name;
//线程组最大优先级
int maxPriority;
//销毁状态
boolean destroyed;
//是否为守护线程组 true： 守护程序线程组 false：普通线程组
boolean daemon;
boolean vmAllowSuspension;
//未开始的线程数量，即NEW状态的线程数
int nUnstartedThreads = 0;
//线程数组的大小
int nthreads;
//当前线程组的线程
Thread threads[];
//子线程组的大小
int ngroups;
//当前线程组的子线程组
ThreadGroup groups[];
```

从字段属性中可以看出

* ThreadGroup可以有自己的父线程组
* ThreadGroup可以有自己的子线程组
* ThreadGroup使用Thread数组作为存储容器

###  3. 构造函数

```java
//默认构造函数，private修饰，底层调用
private ThreadGroup() {     // called from C code
    	//线程组名称默认为system
        this.name = "system";
    	//最大优先级默认为Thread的最大优先级10
        this.maxPriority = Thread.MAX_PRIORITY;
    	//父线程组为null
        this.parent = null;
    }
//接收线程组名称
public ThreadGroup(String name) {
    	//调用下面的构造方法
        this(Thread.currentThread().getThreadGroup(), name);
    }
//接收一个父线程组和线程组名称
public ThreadGroup(ThreadGroup parent, String name) {
    	//调用下面的构造方法
    	//第一个参数需要注意，第一次见到，很神奇这种占位
        this(checkParentAccess(parent), parent, name);
    }

//接收一个Void参数和父线程组、线程组名称
//这个Void很新奇，占位
private ThreadGroup(Void unused, ThreadGroup parent, String name) {
    	//为当前线程组名称赋值
        this.name = name;
    	//最大线程组优先级为父线程组的最大优先级
        this.maxPriority = parent.maxPriority;
    	//守护线程状态为父线程组的守护线程状态
        this.daemon = parent.daemon;
    	//vmAllowSuspension设置为父线程组的vmAllowSuspension
        this.vmAllowSuspension = parent.vmAllowSuspension;
    	//传入的线程组设为父线程组
        this.parent = parent;
    	//父线程组添加当前线程为子线程
        parent.add(this);
    }
```

从构造方法可以看出

* 线程组默认名称为system，默认构造函数是private修饰的，给c代码调用
* 初始化时可以指定父线程组和线程组名称
* 上面的Void参数很不错，再说一遍

### 4. 方法

#### checkParentAccess 方法

```java
//检查线程组访问权限
private static Void checkParentAccess(ThreadGroup parent) {
        parent.checkAccess();
        return null;
    }
```

#### getName 方法

```java
//获取线程组名称
public final String getName() {
        return name;
    }
```

#### getParent 方法

```java
//获取父线程组
public final ThreadGroup getParent() {
        if (parent != null)
            //如果父线程组不为null
            //检查父线程组访问权限
            parent.checkAccess();
        return parent;
    }
```

#### getMaxPriority 方法

```java
//获取最大优先级
public final int getMaxPriority() {
        return maxPriority;
    }
```

#### isDaemon 方法

```java
//检查是否为守护程序线程组
//true 守护程序线程组 ：如果最后一个线程停止或者最后一个线程组被销毁，当前线程组会自动被销毁
//false 普通线程组 ：不会自动销毁
public final boolean isDaemon() {
        return daemon;
    }
```

#### isDestroyed 方法

```java
//检查当前线程组是否已被销毁
public synchronized boolean isDestroyed() {
        return destroyed;
    }
```

#### setDaemon 方法

```java
//设置线程组是否为守护程序线程组
public final void setDaemon(boolean daemon) {
        checkAccess();
        this.daemon = daemon;
    }
```

#### setMaxPriority 方法

```java
//设置最大优先级
public final void setMaxPriority(int pri) {
        int ngroupsSnapshot;
        ThreadGroup[] groupsSnapshot;
    	//==============同步代码块===========
        synchronized (this) {
            //检查访问权限
            checkAccess();
            //参数校验
            if (pri < Thread.MIN_PRIORITY || pri > Thread.MAX_PRIORITY) {
                return;
            }
            //如果父线程组不为null，则比较pri和父线程组的最大优先级取最小的
            //为null，直接使用pri
            maxPriority = (parent != null) ? Math.min(pri, parent.maxPriority) : pri;
            //拷贝子线程组
            ngroupsSnapshot = ngroups;
            if (groups != null) {
                groupsSnapshot = Arrays.copyOf(groups, ngroupsSnapshot);
            } else {
                groupsSnapshot = null;
            }
        }
    	//==============end===========
    	//遍历修改子线程组的最大优先级
    	//可以看出，子线程组的最大优先级不能大于父线程组的最大优先级
        for (int i = 0 ; i < ngroupsSnapshot ; i++) {
            groupsSnapshot[i].setMaxPriority(pri);
        }
    }
```

#### parentOf 方法

```java
//测试传入的线程组是否为当前线程组的父类
public final boolean parentOf(ThreadGroup g) {
    	//往上递归查找
        for (; g != null ; g = g.parent) {
            if (g == this) {
                //找到返回true
                return true;
            }
        }
    	//没找到返回false
        return false;
    }
```

#### checkAccess 方法

```java
//访问权限检查（是否有修改此线程的权限）
public final void checkAccess() {
    	//获取安全管理器
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            //如果安全管理器不为null
            //调用安全管理器的访问权限检查方法
            security.checkAccess(this);
        }
    }
```

#### activeCount 方法

```java
//返回当前线程组线程和子线程组线程的活动线程数
//返回的值是一个预估值，线程状态一直在变化
public int activeCount() {
        int result;
        // Snapshot sub-group data so we don't hold this lock
        // while our children are computing.
        int ngroupsSnapshot;
        ThreadGroup[] groupsSnapshot;
    	//=============同步代码块=================
        synchronized (this) {
            if (destroyed) {
                //如果线程组被销毁，直接返回0
                return 0;
            }
            //nthreads 当前线程组的活动线程数
            result = nthreads;
            //拷贝子线程组
            ngroupsSnapshot = ngroups;
            if (groups != null) {
                groupsSnapshot = Arrays.copyOf(groups, ngroupsSnapshot);
            } else {
                groupsSnapshot = null;
            }
        }
    	//=============end=================
    	//循环递归遍历子线程组的活动线程数量
        for (int i = 0 ; i < ngroupsSnapshot ; i++) {
            result += groupsSnapshot[i].activeCount();
        }
    	//返回统计的活动线程数量
        return result;
    }
```

#### enumerate 方法

```java
//把线程组及其子线程组的活动线程复制到指定Thread数组中 
public int enumerate(Thread list[]) {
    	//检查访问权限
        checkAccess();
    	//调用下面的方法
        return enumerate(list, 0, true);
    }
//把线程组及其子线程组的活动线程复制到指定Thread数组中 
private int enumerate(Thread list[], int n, boolean recurse) {
        int ngroupsSnapshot = 0;
        ThreadGroup[] groupsSnapshot = null;
    	//===========同步代码块============
        synchronized (this) {
            if (destroyed) {
                //如果当前线程组被销毁，直接返回0
                return 0;
            }
            int nt = nthreads;
            //n代表什么？保留n个长度？默认为0
            if (nt > list.length - n) {
                nt = list.length - n;
            }
            //循环添加当前线程组的活动线程
            for (int i = 0; i < nt; i++) {
                if (threads[i].isAlive()) {
                    list[n++] = threads[i];
                }
            }
            //如果recurse为true，把子线程组的活动线程加进去
            if (recurse) {
                ngroupsSnapshot = ngroups;
                if (groups != null) {
                    groupsSnapshot = Arrays.copyOf(groups, ngroupsSnapshot);
                } else {
                    groupsSnapshot = null;
                }
            }
        }
    	//===========end============
     //如果recurse为true，遍历递归添加子线程组的活动线程
        if (recurse) {
            for (int i = 0 ; i < ngroupsSnapshot ; i++) {
                n = groupsSnapshot[i].enumerate(list, n, true);
            }
        }
        return n;
    }
 //把线程组及其子线程组的活动线程复制到指定Thread数组中 
 //resurse true 添加子线程组的活动线程 false 不添加子线程组的线程
 public int enumerate(Thread list[], boolean recurse) {
     	//检查访问权限
        checkAccess();
     	//调用上面的方法
        return enumerate(list, 0, recurse);
    }
```

* 线程组的方法类似

####  activeGroupCount 方法

```java
//返回当前线程组线程和子线程组线程的活动线程组数
public int activeGroupCount() {
        int ngroupsSnapshot;
        ThreadGroup[] groupsSnapshot;
        synchronized (this) {
            if (destroyed) {
                return 0;
            }
            ngroupsSnapshot = ngroups;
            if (groups != null) {
                groupsSnapshot = Arrays.copyOf(groups, ngroupsSnapshot);
            } else {
                groupsSnapshot = null;
            }
        }
    	//赋值当前线程组数量
        int n = ngroupsSnapshot;
    	//循环递归遍历子线程组的线程组数量
        for (int i = 0 ; i < ngroupsSnapshot ; i++) {
            n += groupsSnapshot[i].activeGroupCount();
        }
        return n;
    }
```

#### stop 方法

```java
//停止线程组和所有子线程组的线程
@Deprecated
    public final void stop() {
        //停止当前线程组和所有子线程组的线程
        if (stopOrSuspend(false))
            //停止当前线程
            Thread.currentThread().stop();
    }
```

#### suspend 方法

```java
//挂起线程组和所有子线程组的线程
@Deprecated
    @SuppressWarnings("deprecation")
    public final void suspend() {
        //挂起线程和所有子线程组的线程
        if (stopOrSuspend(true))
            //挂起当前线程
            Thread.currentThread().suspend();
    }
```

#### stopOrSuspend 方法

```java
//停止或挂起当前线程组和子线程组的所有线程
@SuppressWarnings("deprecation")
    private boolean stopOrSuspend(boolean suspend) {
        boolean suicide = false;
        //获取当前线程
        Thread us = Thread.currentThread();
        int ngroupsSnapshot;
        ThreadGroup[] groupsSnapshot = null;
        //===========同步方法块=============
        synchronized (this) {
            //权限检查
            checkAccess();
            for (int i = 0 ; i < nthreads ; i++) {
                if (threads[i]==us)
                    //如果当前遍历线程为当前调用线程 suicide 置为true
                    suicide = true;
                else if (suspend)
                    //suspend true表示挂起线程
                    threads[i].suspend();
                else
                    //suspend false 表示停止线程
                    threads[i].stop();
            }

            ngroupsSnapshot = ngroups;
            if (groups != null) {
                groupsSnapshot = Arrays.copyOf(groups, ngroupsSnapshot);
            }
        }
        //===========end=============
        //循环递归遍历停止或挂起所有子线程组的线程
        for (int i = 0 ; i < ngroupsSnapshot ; i++)
            suicide = groupsSnapshot[i].stopOrSuspend(suspend) || suicide;

        return suicide;
    }
```

#### interrupt 方法

```java
//中断线程组和所有子线程组的线程
public final void interrupt() {
        int ngroupsSnapshot;
        ThreadGroup[] groupsSnapshot;
        synchronized (this) {
            checkAccess();
            //中断当前线程组的线程
            for (int i = 0 ; i < nthreads ; i++) {
                threads[i].interrupt();
            }
            //复制子线程组
            ngroupsSnapshot = ngroups;
            if (groups != null) {
                groupsSnapshot = Arrays.copyOf(groups, ngroupsSnapshot);
            } else {
                groupsSnapshot = null;
            }
        }
    	//循环递归中断子线程组的线程
        for (int i = 0 ; i < ngroupsSnapshot ; i++) {
            groupsSnapshot[i].interrupt();
        }
    }
```

#### resume 方法

```java
//恢复线程组和所有子线程组的线程
@Deprecated
    @SuppressWarnings("deprecation")
    public final void resume() {
        int ngroupsSnapshot;
        ThreadGroup[] groupsSnapshot;
        synchronized (this) {
            checkAccess();
            //恢复当前线程组的线程
            for (int i = 0 ; i < nthreads ; i++) {
                threads[i].resume();
            }
            //复制子线程组
            ngroupsSnapshot = ngroups;
            if (groups != null) {
                groupsSnapshot = Arrays.copyOf(groups, ngroupsSnapshot);
            } else {
                groupsSnapshot = null;
            }
        }
        //循环递归恢复子线程组的线程
        for (int i = 0 ; i < ngroupsSnapshot ; i++) {
            groupsSnapshot[i].resume();
        }
    }
```

#### destroy 方法

```java
//销毁线程组和所有子线程组
public final void destroy() {
        int ngroupsSnapshot;
        ThreadGroup[] groupsSnapshot;
        synchronized (this) {
            checkAccess();
            if (destroyed || (nthreads > 0)) {
                //如果已经被销毁或者还有线程，抛出异常
                throw new IllegalThreadStateException();
            }
            //复制子线程组
            ngroupsSnapshot = ngroups;
            if (groups != null) {
                groupsSnapshot = Arrays.copyOf(groups, ngroupsSnapshot);
            } else {
                groupsSnapshot = null;
            }
            if (parent != null) {
                //如果父线程组不为null
                //清空所有数据
                destroyed = true;
                ngroups = 0;
                groups = null;
                nthreads = 0;
                threads = null;
            }
        }
    	//循环递归销毁子线程组
        for (int i = 0 ; i < ngroupsSnapshot ; i += 1) {
            groupsSnapshot[i].destroy();
        }
        if (parent != null) {
            //如果父线程组不为null
            //从父线程组里移除自己
            parent.remove(this);
        }
    }
```

#### add 方法

```java
//添加子线程组
private final void add(ThreadGroup g){
        synchronized (this) {
            if (destroyed) {
                throw new IllegalThreadStateException();
            }
            if (groups == null) {
                //如果子线程组数组为null
                //默认创建大小为4的线程组数组
                groups = new ThreadGroup[4];
            } else if (ngroups == groups.length) {
                //如果线程组数组满了，扩容到两倍
                groups = Arrays.copyOf(groups, ngroups * 2);
            }
            //赋值
            groups[ngroups] = g;
		    //把线程组数量加1
            ngroups++;
        }
    }
```

#### remove 方法

```java
//移除一个子线程组
private void remove(ThreadGroup g) {
        synchronized (this) {
            if (destroyed) {
                //如果当前线程组被销毁，直接返回
                return;
            }
            for (int i = 0 ; i < ngroups ; i++) {
                //for循环遍历查找
                if (groups[i] == g) {
                    //如果找到
                    //把当前子线程组数量减1
                    ngroups -= 1;
                    //移除当前子线程数组，相当于覆盖，后面的移动到前面
                    System.arraycopy(groups, i + 1, groups, i, ngroups - i);
                    //把子线程数组最后一个置为null
                    groups[ngroups] = null;
                    //结束for循环
                    break;
                }
            }
            if (nthreads == 0) {
                //如果当前线程数为0
                //唤醒所有线程
                notifyAll();
            }
            if (daemon && (nthreads == 0) &&
                (nUnstartedThreads == 0) && (ngroups == 0))
            {
                //如果是守护线程组，并且线程数量和子线程数组都为0
                //销毁当前线程组
                destroy();
            }
        }
    }

//移除线程
private void remove(Thread t) {
        synchronized (this) {
            if (destroyed) {
                //如果线程组被销毁，直接返回
                return;
            }
            //for循环遍历查找
            for (int i = 0 ; i < nthreads ; i++) {
                if (threads[i] == t) {
                    //如果找到
                    //覆盖的方式移除当前线程
                    System.arraycopy(threads, i + 1, threads, i, --nthreads - i);
                    //把子线程数组最后一个置为null
                    threads[nthreads] = null;
                    //结束for循环
                    break;
                }
            }
        }
    }
```

#### addUnstarted 方法

```java
//添加NEW状态的线程
void addUnstarted() {
        synchronized(this) {
            if (destroyed) {
                //如果当前线程组被销毁，抛出异常
                throw new IllegalThreadStateException();
            }
            //nUnstartedThreads加1
            nUnstartedThreads++;
        }
    }
```

#### add 方法

```java
//添加线程
void add(Thread t) {
    	//同步代码块
        synchronized (this) {
            if (destroyed) {
                //如果被销毁，抛出异常
                throw new IllegalThreadStateException();
            }
            if (threads == null) {
                //如果当前线程数组为null
                //创建一个大小为4的线程数组
                threads = new Thread[4];
            } else if (nthreads == threads.length) {
                //如果当前线程数组已经满了
                //扩容到2倍长度
                threads = Arrays.copyOf(threads, nthreads * 2);
            }
            //添加线程
            threads[nthreads] = t;
			//线程数量加1
            nthreads++;
            //未开始的线程统计减1
            nUnstartedThreads--;
        }
    }
```

####  threadStartFailed 方法

```java
//线程启动失败
void threadStartFailed(Thread t) {
    	//回滚状态，后面还可以重试
        synchronized(this) {
            //从线程数组里移除
            remove(t);
            //未开始的线程统计加1
            nUnstartedThreads++;
        }
    }
```

#### threadTerminated 方法

```java
//线程终止
void threadTerminated(Thread t) {
        synchronized (this) {
            //从线程数组里移除
            remove(t);
            if (nthreads == 0) {
                //如果当前活动线程为0
                //唤醒所有线程
                notifyAll();
            }
            if (daemon && (nthreads == 0) &&
                (nUnstartedThreads == 0) && (ngroups == 0))
            {
                //如果是守护线程组，并且线程数量和子线程数组都为0
                //销毁当前线程组
                destroy();
            }
        }
    }
```

#### list 方法

```java
//打印线程组信息到标准输出
public void list() {
    	//调用下面的方法
        list(System.out, 0);
    }
//打印线程组信息到标准输出
void list(PrintStream out, int indent) {
        int ngroupsSnapshot;
        ThreadGroup[] groupsSnapshot;
        synchronized (this) {
            for (int j = 0 ; j < indent ; j++) {
                out.print(" ");
            }
            out.println(this);
            indent += 4;
            //打印当前线程信息
            for (int i = 0 ; i < nthreads ; i++) {
                for (int j = 0 ; j < indent ; j++) {
                    out.print(" ");
                }
                out.println(threads[i]);
            }
            //复制子线程组
            ngroupsSnapshot = ngroups;
            if (groups != null) {
                groupsSnapshot = Arrays.copyOf(groups, ngroupsSnapshot);
            } else {
                groupsSnapshot = null;
            }
        }
    	//循环递归获取子线程的线程
        for (int i = 0 ; i < ngroupsSnapshot ; i++) {
            groupsSnapshot[i].list(out, indent);
        }
    }
```

#### uncaughtException 方法

```java
//当前线程组中线程未捕获异常回调，由虚拟机调用
public void uncaughtException(Thread t, Throwable e) {
        if (parent != null) {
            //父线程组不为null
            //往上递归
            parent.uncaughtException(t, e);
        } else {
            //打印异常信息
            Thread.UncaughtExceptionHandler ueh =
                Thread.getDefaultUncaughtExceptionHandler();
            if (ueh != null) {
                ueh.uncaughtException(t, e);
            } else if (!(e instanceof ThreadDeath)) {
                System.err.print("Exception in thread \""
                                 + t.getName() + "\" ");
                e.printStackTrace(System.err);
            }
        }
    }
```

####  toString 方法

```java
//ThreadGroup的toString方法打印了类名和线程组名称和最大优先级
public String toString() {
        return getClass().getName() + "[name=" + getName() + ",maxpri=" + maxPriority + "]";
    }
```

