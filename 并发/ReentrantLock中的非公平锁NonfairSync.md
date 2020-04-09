### 1. 类的定义

```java
static final class NonfairSync extends Sync
```

从类的定义中可以看出

* NonfairSync是ReentrantLock中的一个静态内部类，并且使用final修饰，表示不可继承
* NonfairSync 继承了Sync，表明NonfairSync也是AbstractQueuedSynchronizer的子类

### 2. 字段属性

```java
//序列化版本号
private static final long serialVersionUID = -3000897897090466540L;
```

### 3. 方法

#### lock 方法

```java
//上锁
final void lock() {
    		//非公平锁，直接获取锁即可
            if (compareAndSetState(0, 1))
                //获取成功把锁的占有线程设置为当前线程
                setExclusiveOwnerThread(Thread.currentThread());
            else
                //直接获取锁失败，调用AbstractQueuedSynchronizer的acquire方法
    			//先尝试获取锁，失败的话进入等待队列
                acquire(1);
        }
```

#### tryAcquire 方法

```java
//尝试获取锁的方法，重写父类的方法
protected final boolean tryAcquire(int acquires) {
    		//直接调用ReentrantLock的nonfairTryAcquire方法
            return nonfairTryAcquire(acquires);
        }
```

