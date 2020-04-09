### 1. 类的定义

```java
public class Semaphore implements java.io.Serializable 
```

### 2. 字段属性

```java
//序列化版本号
private static final long serialVersionUID = -3222578661600680210L;
//同步器，AbstractQueuedSynchronizer的子类
private final Sync sync;
```

从字段属性中可以看出

* Semaphore的所有操作都是使用内部类Sync对象进行操作的

### 3. 构造方法

```java
//传入信号数
public Semaphore(int permits) {
    	//默认使用非公平锁
        sync = new NonfairSync(permits);
    }
//传入信号数和锁的类型
public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }
```

从构造方法中可以看出

* 构造方法只做了一件事：初始化sync对象

### 4.方法

#### acquire 方法

```java
//可中断的获取信号量
public void acquire() throws InterruptedException {
    	//调用sync的acquireSharedInterruptibly方法
        sync.acquireSharedInterruptibly(1);
    }

//获取指定数量的信号量
public void acquire(int permits) throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
    	//调用sync的acquireSharedInterruptibly方法
        sync.acquireSharedInterruptibly(permits);
    }
```

### acquireUninterruptibly 方法

```java
//不可中断的获取信号量
public void acquireUninterruptibly() {
    	//调用sync的acquireShared方法
        sync.acquireShared(1);
    }
//不可中断的获取指定数量的信号量
public void acquireUninterruptibly(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireShared(permits);
    }
```

#### tryAcquire 方法

```java
//尝试获取信号量
public boolean tryAcquire() {
    	//调用sync的nonfairTryAcquireShared方法
        return sync.nonfairTryAcquireShared(1) >= 0;
    }

//尝试获取指定数量的信号量
public boolean tryAcquire(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
    	//尝试获取指定数量的信号量
        return sync.nonfairTryAcquireShared(permits) >= 0;
    }

//设置超时时间的尝试获取信号量
public boolean tryAcquire(long timeout, TimeUnit unit)
        throws InterruptedException {
    	//调用sync的tryAcquireSharedNanos方法
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

//设置超时时间的尝试获取指定数量的信号量
public boolean tryAcquire(int permits, long timeout, TimeUnit unit)
        throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
    	//调用sync的tryAcquireSharedNanos方法
        return sync.tryAcquireSharedNanos(permits, unit.toNanos(timeout));
    }
```

#### release 方法

```java
//释放一个信号量
public void release() {
    	//调用sync的releaseShared方法
        sync.releaseShared(1);
    }
//释放指定数量的信号量
public void release(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.releaseShared(permits);
    }
```

#### availablePermits 方法

```java
//获取当前可用的通道数
public int availablePermits() {
        return sync.getPermits();
    }
```

#### drainPermits 方法

```java
//获取立即可用的通道数
public int drainPermits() {
        return sync.drainPermits();
    }
```

#### reducePermits 方法

```java
//减少信号数
protected void reducePermits(int reduction) {
        if (reduction < 0) throw new IllegalArgumentException();
        sync.reducePermits(reduction);
    }
```

#### isFair 方法

```java
//获取锁的类型， true 公平锁， false 非公平锁
public boolean isFair() {
        return sync instanceof FairSync;
    }
```

#### hasQueuedThreads 方法

```java
//队列中是否有正在等信号的线程
public final boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }
```

#### getQueueLength 方法

```java
//获取队列中等待信号的线程数
public final int getQueueLength() {
        return sync.getQueueLength();
    }
```

#### getQueuedThreads 方法

```java
//获取队列中的线程，以集合的方式返回
protected Collection<Thread> getQueuedThreads() {
        return sync.getQueuedThreads();
    }
```

### 5. 内部类Sync

#### 1. 类的定义

```java
abstract static class Sync extends AbstractQueuedSynchronizer
```

从类的定义中可以看出

* Sync是一个抽象的静态内部类，使用模板方法的设计模式让子类继承
* Sync继承AbstractQueuedSynchronizer类

#### 2. 字段属性

```java
//序列化版本号
private static final long serialVersionUID = 1192457210091910933L;
```

#### 3. 构造方法

```java
//传入的信号数就是AQS中的state
Sync(int permits) {
            setState(permits);
        }
```

#### 4. 方法

##### getPermits 方法

```java
//获取信号数
final int getPermits() {
    		//实质上就是获取state的值
            return getState();
        }
```

##### nonfairTryAcquireShared 方法

```java
//非公平方式获取共享锁，返回剩余可用信号数
final int nonfairTryAcquireShared(int acquires) {
    		//for无限循环，自旋CAS
            for (;;) {
                //获取当前的state
                int available = getState();
                //可用信号-获取数量=剩余可用数量
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```

##### tryReleaseShared 方法

```java
//尝试释放共享锁
protected final boolean tryReleaseShared(int releases) {
    		//for无限循环，自旋CAS
            for (;;) {
                //获取当前的state
                int current = getState();
                //可用信号+释放数量=新的可用数量
                int next = current + releases;
                if (next < current) // overflow
                    //releases小于0 抛出异常
                    throw new Error("Maximum permit count exceeded");
                //CAS，设置新值
                if (compareAndSetState(current, next))
                    return true;
            }
        }
```

##### reducePermits 方法

```java
//减少信号数量
final void reducePermits(int reductions) {
    		//for无限循环，自旋CAS
            for (;;) {
                //获取当前的state
                int current = getState();
                //可用信号-减少的数量=新的可用数量
                int next = current - reductions;
                if (next > current) // underflow
                    //reductions小于0 抛出异常
                    throw new Error("Permit count underflow");
                //CAS，设置新值
                if (compareAndSetState(current, next))
                    return;
            }
        }
```

#####  drainPermits 方法

```java
//清空信号
final int drainPermits() {
    		//CAS 自旋把state置为0
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
```

### 6.  内部类 NonfairSync

#### 1. 类的定义

```java
static final class NonfairSync extends Sync
```

从类的定义可以看出

* NonfairSync是Sync的子类

#### 2. 字段属性

```java
//序列化版本号
private static final long serialVersionUID = -2694183684443567898L;
```

#### 3. 构造方法

```java
//设置state
NonfairSync(int permits) {
            super(permits);
        }
```

#### 4. 方法

##### tryAcquireShared 方法

```java
//尝试获取共享锁
protected int tryAcquireShared(int acquires) {
    		//调用父类的方法nonfairTryAcquireShared，直接抢锁
            return nonfairTryAcquireShared(acquires);
        }
```

### 7. 内部类FairSync

#### 1. 类的定义

```java
 static final class FairSync extends Sync
```

从类的定义中可以看出

* FairSync继承Sync类

#### 2. 字段属性

```java
//序列化版本号
private static final long serialVersionUID = 2014338818796000944L;
```

#### 3. 构造方法

```java
//设置state
FairSync(int permits) {
            super(permits);
        }
```

#### 方法

##### tryAcquireShared 方法

```java
//尝试获取共享锁
protected int tryAcquireShared(int acquires) {
            for (;;) {
                //先看队列中是否有线程排队
                if (hasQueuedPredecessors())
                    return -1;
                //获取state的值
                int available = getState();
                //可用信号-获取数量=剩余可用数量
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```



#### 



