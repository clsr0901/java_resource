### 1. 类的定义

```java
public class ConditionObject implements Condition, java.io.Serializable
```

从类的定义中可以看出

* ConditionObject实现了Condition接口
* ConditionObject实现了java.io.Serializable接口

### 2. 字段属性

```java
//序列化版本号
private static final long serialVersionUID = 1173984872572414699L;
//条件队列第一个节点
//这个队列不同于AQS中的队列，AQS中的队列叫做等待队列（阻塞队列）
//最终条件队列中的节点都要添加到等待队列去等待获取锁执行
private transient Node firstWaiter;
//条件队列最后一个节点
private transient Node lastWaiter;
```

从字段属性中可以看出

* ConditionObject自己内部维护了一个队列
* ConditionObject内部的队列跟AQS中的队列不一样，Node中的nextWaiter属性就是为条件队列准备的，意味着条件队列是在等待队列中延出来的分支队列，等待队列中每个节点还可以有一个条件队列

### 3. 构造方法

```java
public ConditionObject() { }
```

### 4. 方法

####  await 方法

```java
//等待被唤醒，这个方法可以被中断
public final void await() throws InterruptedException {
    		//判断当前线程是否被中断
            if (Thread.interrupted())
                throw new InterruptedException();
    		//添加到条件队列中
            Node node = addConditionWaiter();
    		//释放锁，并获取以前的state值
    		//这个savedState表示当前持有几把锁
            int savedState = fullyRelease(node);
            int interruptMode = 0;
    		//判断是否已经转移到阻塞队列
            while (!isOnSyncQueue(node)) {
                //如果不在阻塞队列中，挂起线程，等待被唤醒
                LockSupport.park(this);
                //checkInterruptWhileWaiting检查线程是否中断
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
    		//节点已经添加到阻塞队列，线程被唤醒
    		// 获取独占锁
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
    		//断开节点和条件队列的关系
            if (node.nextWaiter != null) // clean up if cancelled
                //清除条件队列中取消的节点
                unlinkCancelledWaiters();
    		/*
    		先解释下 interruptMode。interruptMode 可以取值为 REINTERRUPT（1），THROW_IE（-1），0
			REINTERRUPT： 代表 await 返回的时候，需要重新设置中断状态
			THROW_IE： 代表 await 返回的时候，需要抛出 InterruptedException 异常
			0 ：说明在 await 期间，没有发生中断
			*/
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

//带超时的等待
public final boolean await(long time, TimeUnit unit)
                throws InterruptedException {
    		//计算超时时间
            long nanosTimeout = unit.toNanos(time);
    		//判断当前线程是否被中断
            if (Thread.interrupted())
                throw new InterruptedException();
    		//添加到条件队列中
            Node node = addConditionWaiter();
    		//释放锁，并获取以前的state值
    		//这个savedState表示当前持有几把锁
            int savedState = fullyRelease(node);
    		// 当前时间 + 等待时长 = 过期时间
            final long deadline = System.nanoTime() + nanosTimeout;
            boolean timedout = false;
            int interruptMode = 0;
    		//判断是否已经转移到阻塞队列
            while (!isOnSyncQueue(node)) {
                //超时了
                if (nanosTimeout <= 0L) {
                    //超时要取消等待
                    //取消等待的时候要把节点转移到等待队列
                    timedout = transferAfterCancelledWait(node);
                    break;
                }
                //static final long spinForTimeoutThreshold = 1000L;
                //如果不到 1 毫秒了，那就不要选择 parkNanos 了，自旋的性能反而更好
                if (nanosTimeout >= spinForTimeoutThreshold)
                    //挂起线程,超时在这体现
                    LockSupport.parkNanos(this, nanosTimeout);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                //计算新的剩余时间
                nanosTimeout = deadline - System.nanoTime();
            }
    		//节点已经添加到阻塞队列，线程被唤醒
    		// 获取独占锁
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
    		//断开节点和条件队列的关系
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
    		/*
    		先解释下 interruptMode。interruptMode 可以取值为 REINTERRUPT（1），THROW_IE（-1），0
			REINTERRUPT： 代表 await 返回的时候，需要重新设置中断状态
			THROW_IE： 代表 await 返回的时候，需要抛出 InterruptedException 异常
			0 ：说明在 await 期间，没有发生中断
			*/
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return !timedout;
        }
```

#### addConditionWaiter 方法

```java
//添加到条件队列
private Node addConditionWaiter() {
    		//获取条件队列尾节点
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
    		//t.waitStatus != Node.CONDITION 表示被取消
            if (t != null && t.waitStatus != Node.CONDITION) {
                //如果尾节点被取消，把尾节点清除
                unlinkCancelledWaiters();
                //重新获取尾节点
                t = lastWaiter;
            }
    		//创建新的节点
    		//注意这里的waitStatus设置的为CONDITION,可以解释上面为什么不等于CONDITION表示取消
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
    		//把新节点添加到尾部
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
```

#### unlinkCancelledWaiters 方法

```java
//移除被取消的节点
private void unlinkCancelledWaiters() {
    		//获取头结点
            Node t = firstWaiter;
    		//保存遍历的当前节点
            Node trail = null;
            while (t != null) {
                //获取下一个节点
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    //如果当前节点被取消
                    //下面的操作是把当前节点移除
                    //把下一个节点置引用为null
                    t.nextWaiter = null;
                    if (trail == null)
                        //如果trail为null，把头结点指向下一个节点
                        firstWaiter = next;
                    else
                        //如果trail不为null，把trail的下一个节点引用指向下一个节点
                        //这里表示移除当前节点, trail相当于当前节点的前一个节点
                        trail.nextWaiter = next;
                    if (next == null)
                        //如果到达尾节点，把尾节点指向trail
                        lastWaiter = trail;
                }
                else
                    //trail指向当前节点
                    trail = t;
                //当前节点指向下个节点，继续遍历
                t = next;
            }
        }
```

#### fullyRelease 方法

```java
//释放指定Node所有的锁，返回持有锁的数量
final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            //获取当前锁的数量
            int savedState = getState();
            //释放所有锁，相当于CAS设置state为0
            if (release(savedState)) {
                failed = false;
                //返回数量
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                //如果CAS失败，把当前node置为取消状态
                node.waitStatus = Node.CANCELLED;
        }
    }
```

#### isOnSyncQueue 方法

```java
//判断指定节点是否在等待队列中
final boolean isOnSyncQueue(Node node) {
    	//如果移动到等待队列中，node 的 waitStatus 会置为 0
    	//Node.CONDITION表示还在条件队列中
    	//node的前驱节点为null说明不会再等待队列中，等待队列中head必须存在
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null) // If has successor, it must be on queue
            //如果有后驱节点，肯定是在等待队列中
   
    	//从阻塞队列中，从后往前找指定节点
        return findNodeFromTail(node);
    }
```

#### findNodeFromTail 方法

```java
//从后往前找指定节点
private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (;;) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    }
```

#### signal 方法

```java
//唤醒条件队列中第一个节点
public final void signal() {
    		//必须持有独占锁才能操作
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                //找出第一个节点，并移动到等待队列中去
                doSignal(first);
        }
```

#### signalAll 方法

```java
//唤醒条件队列中所有节点 
public final void signalAll() {
     		//必须持有独占锁才能操作
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                //移动所有节点到等待队列
                doSignalAll(first);
        }
```

#### doSignal 方法

```java
//找出第一个节点，并移动到等待队列中去
private void doSignal(Node first) {
            do {
                // 将 firstWaiter 指向 first 节点后面的第一个，因为 first 节点马上要离开了
        		// 如果将 first 移除后，后面没有节点在等待了，那么需要将 lastWaiter 置为 null
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                // 因为 first 马上要被移到阻塞队列了，和条件队列的链接关系在这里断掉
                first.nextWaiter = null;
                //transferForSignal 移动节点到等待队列
                //如果 first 转移不成功，那么选择 first 后面的第一个节点进行转移，依此类推
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
```

#### doSignalAll 方法

```java
//移动所有节点到等待队列
private void doSignalAll(Node first) {
            lastWaiter = firstWaiter = null;
    		//while循环操作
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                //transferForSignal 移动节点到等待队列
                transferForSignal(first);
                first = next;
            } while (first != null);
        }
```

#### transferForSignal 方法

```java
//移动指定节点到等待队列
final boolean transferForSignal(Node node) {
       	//把节点的waitStatus设置为0
    	//如果设置失败表示节点已经取消获取锁
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;
		//把指定节点添加到等待队列
    	//自旋添加到队尾,p是node节点的前驱节点
        Node p = enq(node);
        int ws = p.waitStatus;
    	// ws > 0 说明 node 在阻塞队列中的前驱节点取消了等待锁，直接唤醒 node 对应的线程。
    	// 如果 ws <= 0, 那么 compareAndSetWaitStatus 将会被调用
    	//节点入队后，需要把前驱节点的状态设为 Node.SIGNAL(-1)
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            // 如果前驱节点取消或者 CAS 失败，会进到这里唤醒线程
            LockSupport.unpark(node.thread);
    	//成功添加到等待队列，返回true
        return true;
    }
```

#### checkInterruptWhileWaiting 方法

```java
// 1. 如果在 signal 之前已经中断，返回 THROW_IE
// 2. 如果是 signal 之后中断，返回 REINTERRUPT
// 3. 没有发生中断，返回 0
private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
        }
```

#### transferAfterCancelledWait 方法

```java
// 只有线程处于中断状态，才会调用此方法
final boolean transferAfterCancelledWait(Node node) {
    	//CAS设置waitStatus为0
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
            //加入等待队列
            enq(node);
            //成功返回true
            return true;
        }
        //CAS失败，因为 signal 方法已经将 waitStatus 设置为了 0
    	// signal 方法会将节点转移到阻塞队列，但是可能还没完成，这边自旋等待其完成
    	//signal 调用之后，没完成转移之前，发生了中断
    	//即使发生了中断，还是会转移到等待队列
        while (!isOnSyncQueue(node))
            Thread.yield();
        return false;
    }
```



#### reportInterruptAfterWait 方法

```java
private void reportInterruptAfterWait(int interruptMode)
            throws InterruptedException {
            if (interruptMode == THROW_IE)//抛出异常
                throw new InterruptedException();
            else if (interruptMode == REINTERRUPT)//需要重新被中断
                selfInterrupt();
        }
```

#### awaitUninterruptibly 方法

```java
//不能中断的等待
public final void awaitUninterruptibly() {
    		//添加到条件队列中
            Node node = addConditionWaiter();
    		//释放锁，并获取以前的state值
    		//这个savedState表示当前持有几把锁
            int savedState = fullyRelease(node);
            boolean interrupted = false;
    		//判断是否已经转移到阻塞队列
            while (!isOnSyncQueue(node)) {
                //如果不在阻塞队列中，挂起线程，等待被唤醒
                LockSupport.park(this);
                //判断线程是否被中断
                if (Thread.interrupted())
                    interrupted = true;
            }
    		//节点已经添加到阻塞队列，线程被唤醒
    		// 获取独占锁
            if (acquireQueued(node, savedState) || interrupted)
                selfInterrupt();
        }
```

#### acquireQueued 方法

```java
//尝试获取锁，失败的话就中断等着前驱节点唤醒自己
final boolean acquireQueued(final Node node, int arg) {
    	//默认失败
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //获取node的前驱结点
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    //如果p为头节点 并且 获取锁成功
                    //把node设置为头节点
                    setHead(node);
                    //把前驱结点的后驱节点置为null
                    p.next = null; // help GC
                    //设置为成功标志
                    failed = false;
                    //返回interrupted标志
                    return interrupted;
                }
                //找到能够唤醒自己的前驱节点，然后中断在这等着唤醒
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    //如果node需要被挂起
                    //interrupted标志置为true
                    interrupted = true;
            }
        } finally {
            if (failed)
                //如果失败，取消node获取锁的尝试
                cancelAcquire(node);
        }
    }
```

#### hasWaiters 方法

```java
//判断条件队列中是否有节点等待锁
protected final boolean hasWaiters() {
    		//当前线程必须获取到独占锁的
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
    		//for循环，只要有存在waitStatus为Node.CONDITION就表示有节点在等锁
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION)
                    return true;
            }
            return false;
        }
```

#### getWaitQueueLength 方法

```java
//获取条件队列中等待锁的节点数
protected final int getWaitQueueLength() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int n = 0;
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION)
                    ++n;
            }
            return n;
        }
```

#### getWaitingThreads 方法

```java
//获取条件队列中等待锁的线程数，并以集合的方式返回
protected final Collection<Thread> getWaitingThreads() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            ArrayList<Thread> list = new ArrayList<Thread>();
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION) {
                    Thread t = w.thread;
                    if (t != null)
                        list.add(t);
                }
            }
            return list;
        }
```

