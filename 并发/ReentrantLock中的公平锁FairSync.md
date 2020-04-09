### 1. 类的定义

```java
static final class FairSync extends Sync
```

从类的定义中可以看出

* FairSync是ReentrantLock中的一个静态内部类，并且使用final修饰，表示不可继承
* FairSync 继承了Sync，表明FairSync也是AbstractQueuedSynchronizer的子类

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
    		//调用AbstractQueuedSynchronizer的acquire方法
    		//先尝试获取锁，失败的话进入等待队列
            acquire(1);
        }
```

#### tryAcquire 方法

```java
//尝试获取锁的方法，重写父类的方法
protected final boolean tryAcquire(int acquires) {
    		//获取当前线程
            final Thread current = Thread.currentThread();
    		//获取状态值
            int c = getState();
            if (c == 0) {
                //c等于0，表示当前没有线程持有锁，但是可能是上一个线程刚好释放锁
                //既然是公平锁，应该先检查队列前面有没有等待的节点，有的话先让前面的节点获取锁
                //没有的话通过CAS修改state值来获取锁，并把持有锁的线程设置为当前线程
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
    		//如果当前线程已经获取了锁，表示重入锁
    		//这里只需要把state值加1即可
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
    		//没获取到锁返回false
            return false;
        }
```



