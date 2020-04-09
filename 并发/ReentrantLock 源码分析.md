### 1. 类的定义

```java
public class ReentrantLock implements Lock, java.io.Serializable
```

从类的定义中可以看出

* ReentrantLock实现了Lock接口
* ReentrantLock实现了java.io.Serializable接口，表示支持序列化

### 2. 字段属性

```java
//序列化版本号
private static final long serialVersionUID = 7373984872572414699L;
//同步器，实现了所有的同步机制。是AbstractQueuedSynchronizer的子类
private final Sync sync;
```

从字段属性中可以看出

* ReentranLock最核心的就是同步器Sync，所有的操作都是通过它来完成的

### 3. 构造方法

```java
//空的默认构造方法
public ReentrantLock() {
    	//默认是非公平锁
        sync = new NonfairSync();
    }

//传入一个bool对象，指定公平锁还是非公平锁 true 公平锁； false 非公平锁
 public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

从构造方法可以看出

* ReentrantLock的构造方法只是为了初始化Sync对象

### 4. 方法

#### lock 方法

```java
//获取锁
public void lock() {
    	//调用sync的lock方法获取锁
        sync.lock();
    }
```

#### lockInterruptibly 方法

```java
//获取锁，如果当前线程已经中断，抛出异常
public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
```

#### tryLock 方法

```java
//尝试获取锁
public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }

//尝试获取锁，指定超时时间
public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
```

####  unlock 方法

```java
//释放锁
public void unlock() {
        sync.release(1);
    }
```

#### newCondition 方法

```java
//获取一个Condition实例
public Condition newCondition() {
        return sync.newCondition();
    }
```

#### getHoldCount 方法

```java
//获取当前线程对锁的获取次数，注意 ReentrantLock是重入锁，可以多次获取锁
public int getHoldCount() {
        return sync.getHoldCount();
    }
```

#### isHeldByCurrentThread 方法

```java
//查询该锁是否被当前线程持有
public boolean isHeldByCurrentThread() {
        return sync.isHeldExclusively();
    }
```

#### isLocked 方法

```java
//查询是否有线程持有该锁
public boolean isLocked() {
        return sync.isLocked();
    }
```

#### isFair 方法

```java
//查询该锁是否为公平锁
public final boolean isFair() {
        return sync instanceof FairSync;
    }
```

#### getOwner 方法

```java
//获取拥有该锁的当前线程
protected Thread getOwner() {
        return sync.getOwner();
    }
```

#### hasQueuedThreads 方法

```java
//查询是否有线程正在等待获取该锁
public final boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }

//查询是否给定的线程正在等待获取该锁
public final boolean hasQueuedThread(Thread thread) {
        return sync.isQueued(thread);
    }
```

#### getQueueLength 方法

```java
//获取等待获取该锁线程的数量
public final int getQueueLength() {
        return sync.getQueueLength();
    }
//获取等待获取该锁线程，以集合方法返回
protected Collection<Thread> getQueuedThreads() {
        return sync.getQueuedThreads();
    }
```

#### hasWaiters 方法

```java
//查询是否有任何线程正在等待与此锁关联的给定条件
public boolean hasWaiters(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.hasWaiters((AbstractQueuedSynchronizer.ConditionObject)condition);
    }
```

#### getWaitQueueLength  方式

```java
//返回等待与此锁关联给定条件的线程数
public int getWaitQueueLength(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.getWaitQueueLength((AbstractQueuedSynchronizer.ConditionObject)condition);
    }
```

#### getWaitingThreads 方法

```java
//返回等待与此锁关联给定条件的线程集合
protected Collection<Thread> getWaitingThreads(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.getWaitingThreads((AbstractQueuedSynchronizer.ConditionObject)condition);
    }
```



可以看出，ReentrantLock中所有的方法都是通过`Sync`对象来实现的