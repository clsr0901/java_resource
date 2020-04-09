### 1. 类的定义

```java
abstract static class Sync extends AbstractQueuedSynchronizer
```

从类的定义中可以看出

* Sync是一个抽象类
* Sync继承了AbstractQueuedSynchronizer类

### 2. 字段属性

```java
//序列化版本号
private static final long serialVersionUID = -5179523762034025860L;
```

### 3. 方法

#### lock 方法

```java
//上锁，抽象方法，让子类实现
abstract void lock();
```

#### nonfairTryAcquire 方法

```java
//尝试获取非公平锁
final boolean nonfairTryAcquire(int acquires) {
    		//获取当前线程
            final Thread current = Thread.currentThread();
    		//获取当前锁的状态
            int c = getState();
            if (c == 0) {
                //表示当前没有线程获取锁
                //CAS方式获取锁
                if (compareAndSetState(0, acquires)) {
                    //获取成功，把持有当前锁的线程设置为当前线程
                    setExclusiveOwnerThread(current);
                    //返回true
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                //如果当前线程持有锁，这里表示重入锁
                //修改状态为获取锁的次数
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
    		//获取锁失败返回false
            return false;
        }
```

#### tryRelease 方法

```java
//尝试释放锁
protected final boolean tryRelease(int releases) {
    		//计算新的状态值
            int c = getState() - releases;
    		//如果当前线程没有持有当前锁，抛出异常
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
    		//是否完全释放标志
            boolean free = false;
            if (c == 0) {
                //状态为0表示完全释放
                //完全释放标志设置为true
                free = true;
                //把持有当前锁的线程置为null
                setExclusiveOwnerThread(null);
            }
    		//重新设置锁的状态
            setState(c);
            return free;
        }
```

#### isHeldExclusively 方法

```java
//判断持有锁的是否为当前线程
protected final boolean isHeldExclusively() {
            return getExclusiveOwnerThread() == Thread.currentThread();
        }
```

#### newCondition 方法

```java
//获取一个新的Condition对象
final ConditionObject newCondition() {
            return new ConditionObject();
        }
```

#### getOwner 方法

```java
//获取持有当前锁的线程
final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }
```

#### getHoldCount 方法

```java
//获取当前锁被持有的次数
final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }
```

#### isLocked 方法

```java
//判断是否有线程持有当前锁
final boolean isLocked() {
            return getState() != 0;
        }
```

