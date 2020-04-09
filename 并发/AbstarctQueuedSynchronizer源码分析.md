### 1. 类的定义

```java
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer
    implements java.io.Serializable
```

从类的定义中可以看出

* AbstractQueuedSynchronizer 是一个抽象类，使用了模板方法设计模式（实现了大部分功能，极少部分让子类实现）
* AbstractQueuedSynchronizer 继承了 AbstractOwnableSynchronizer
* AbstractOwnableSynchronizer 实现了java.io.Serializable接口，表示AbstractQueuedSynchronizer支持序列化

### 2. 字段属性

```java
//序列化版本号
private static final long serialVersionUID = 7373984972572414691L;
//等待队列的头，延迟初始化。 仅在初始化时，使用setHead修改。 注意：如果头存在，其等待状态保证不会被取消
private transient volatile Node head;
//等待队列的尾部，延迟初始化
private transient volatile Node tail;
//同步状态， 0(默认初始值)表示没有锁，1表示上锁，可以大于1表示重入锁，有的初始值为-1
//state使用volatile修饰
private volatile int state;
```

从字段属性可以看出

* AbstractQueuedSynchronizer内部维护了一个链表队列
* AbstractQueuedSynchronizer存储了锁的状态state

### 3. 构造方法

```java
//默认空的构造函数，没有任何操作
protected AbstractQueuedSynchronizer() { }
```

从构造方法可以看出

* AbstractQueuedSynchronizer的构造方法使用protected修饰，是一个空的构造函数，需要子类自己实现

### 4. 方法

#### getState 方法

```java
//获取当前的同步状态	
protected final int getState() {
        return state;
    }
```

#### setState 方法

```java
//设置当前的同步状态
protected final void setState(int newState) {
        state = newState;
    }
```

#### compareAndSetState 方法

```java
//使用CAS设置同步状态
protected final boolean compareAndSetState(int expect, int update) {
        //使用Unsafe的compareAndSwapInt方法更新同步状态，这是一个native方法
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```

#### enq 方法

```java
//添加节点到队尾
private Node enq(final Node node) {
    	//for无限循环，重试CAS
        for (;;) {
            //如果队列尾节点
            Node t = tail;
            if (t == null) { // Must initialize
                //如果尾节点为null，进行初始化
                //CAS设置一个新创建的节点为头节点，有可能其它线程先创建成功
                if (compareAndSetHead(new Node()))
                    //创建头节点成功，把尾节点指向头节点
                    tail = head;
            } else {
                //如果尾节点不为null
                //把加入的节点前驱结点的引用指向尾节点
                node.prev = t;
                //CAS设置node为尾节点，有可能其它节点也在设置尾节点，可能会失败，然后重试
                if (compareAndSetTail(t, node)) {
                    //如果添加成功
                    //把前驱节点的后驱的引用指向加入的节点
                    t.next = node;
                    //返回加入节点的前驱节点（可能是以前的尾节点，其它线程可能会修改它）
                    return t;
                }
            }
        }
    }
```

#### compareAndSetHead 方法

```java
//设置头节点，只在enq方法中调用
private final boolean compareAndSetHead(Node update) {
    	//如果头节点为null，才把传入的节点设置为头节点
        return unsafe.compareAndSwapObject(this, headOffset, null, update);
    }
```

#### compareAndSetTail 方法

```java
//设置尾节点，只在enq方法中调用
private final boolean compareAndSetTail(Node expect, Node update) {
    	//如果尾节点为expect，才把传入的节点设置为尾节点
        return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
    }
```

#### addWaiter 方法

```java
//添加节点到队尾
private Node addWaiter(Node mode) {
    	//使用当前线程和传入节点创建新的节点，后面添加的是这个新创建的节点
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
    	//获取尾节点的副本
        Node pred = tail;
        if (pred != null) {
            //如果尾节点不为null
            //把添加节点的前驱结点引用指向尾节点
            node.prev = pred;
            //CAS设置node为尾节点，有可能其它节点也在设置尾节点，可能会失败，然后重试
            if (compareAndSetTail(pred, node)) {
                //把前驱节点的后驱节点的引用指向加入的节点
                pred.next = node;
                //返回加入节点的前驱节点（可能是以前的尾节点，其它线程可能会修改它）
                return node;
            }
        }
    	//如果尾节点为null,或者被其它线程先添加成功
    	//使用enq方法添加
        enq(node);
        return node;
    }
```

#### setHead 方法

```java
//设置头节点，仅由acquire方法调用
private void setHead(Node node) {
    	//把头节点指向传入的节点
        head = node;
    	//把传入节点的thread置为null
        node.thread = null;
    	//把传入节点的prev置为null
        node.prev = null;
    }
```

#### unparkSuccessor 方法

```java
//如果有节点退出，唤醒队列下一个未被取消节点的线程执行
//注意，这个node参数是头节点
private void unparkSuccessor(Node node) {
    	//获取头节点的waitStatus，waitStatus大于0表示被取消
        int ws = node.waitStatus;
        if (ws < 0)
            //如果头节点的waitStatus小于0， 把头节点的waitStatus设置为0
            compareAndSetWaitStatus(node, ws, 0);

    	//获取头节的后驱节点
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            //如果后驱节点为nul 或者 后驱节点的waitStatus大于0（被取消）
            s = null;
            //使用for循环从尾节点开始往前遍历
            //找到遍历的第一个waitStatus小于等于0（未被取消）的节点
            //注意，找到的是顺序的第一个，不是倒序的第一个，这个会一直循环到头节点
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            //如果节点不为null， 唤醒节点的线程
            LockSupport.unpark(s.thread);
    }
```

#### doReleaseShared 方法

```java
//共享模式下的信号释放
private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
    	//for无限循环
        for (;;) {
            //获取头节点的副本
            Node h = head;
            if (h != null && h != tail) {
                //如果h节点不为null 并且 h节点不等于尾节点（队列中有元素）
                //获取h节点的waitStatus
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    //如果h节点的状态为-1
                    //把h节点的waitStatus设置为0
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        //如果失败进行下一次循环（继续操作）                        
                        continue;            // loop to recheck cases
                    //如果设置成功，唤醒h的后驱节点，头节点的状态可能会改变
                    unparkSuccessor(h);
                }
                //如果h节点的状态为0 并且 CAS设置节点的waitStatus为-3失败的话进行下一次循环
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                //如果h节点为头节点中断循环，如果head改变则继续循环
                break;
        }
    }
```

#### setHeadAndPropagate 方法

```java
//设置队列头和传播方式 
private void setHeadAndPropagate(Node node, int propagate) {
    	//获取头节点的副本
        Node h = head; // Record old head for check below
    	//把传入的节点设置为头节点
        setHead(node);
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            //如果传入的传播方式大于0
            //如果头节点为null
            //如果头节点的waitStatus小于0
            //如果node为null
            //如果node的waitStatus小于0
            
            //获取node的后驱节点
            Node s = node.next;
            if (s == null || s.isShared())
                //如果后驱节点为null 或者 后驱节点是共享模式
                //释放共享模式下的信号
                doReleaseShared();
        }
    }
```

#### cancelAcquire 方法

```java
//取消节点获取锁的尝试
private void cancelAcquire(Node node) {
        if (node == null)
            //如果node为null，直接返回
            return;
		//把node的线程置为null
        node.thread = null;

        //获取node的前一个节点
        Node pred = node.prev;
    	//while循环，如果前驱结点的waitStatus大于0（被取消），一直往上找
    	//找到前面第一个未被取消的节点
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;

    	//获取前驱结点的后驱节点
        Node predNext = pred.next;

    	//把node节点的waitStatus设置为取消状态
        node.waitStatus = Node.CANCELLED;

        // 如果当前节点是尾节点，移除自己，把前驱结点设置为尾节点
        if (node == tail && compareAndSetTail(node, pred)) {
            //把前驱节点的后驱节点设置为null
            compareAndSetNext(pred, predNext, null);
        } else {
            //node不是尾节点的情况
            //If successor needs signal, try to set pred's next-link
            //so it will get one. Otherwise wake it up to propagate.
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    compareAndSetNext(pred, predNext, next);
            } else {
                //唤醒前驱结点
                unparkSuccessor(node);
            }
            node.next = node; // help GC
        }
    }
```

#### shouldParkAfterFailedAcquire 方法

```java
//当前线程没有获取到锁，是否需要挂起线程
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    	//获取前驱结点的waitStatus
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
           //前驱节点的waitStatus为-1，说明前驱结点状态正常，可以安全的挂起当前线程，直接返回true
            return true;
        if (ws > 0) {
            //如果前驱结点的waitStatus大于0，表示前驱结点取消了排队
            //注意：进入阻塞队列排队的线程会被挂起，而唤醒的操作是由前驱节点完成的
            //往前一直找，直到找到waitStatus大于等于0的节点作为node节点的前驱节点
            //node的线程挂起以后后面要靠找到的前驱节点来唤醒
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            //前驱节点的waitStatus不等于1和-1，只能等于0，-2， -3
            //把前驱节点的waitStatus设置为-1
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
    	//返回false表示不需要挂起
        return false;
    }
```

#### selfInterrupt 方法

```java
//中断当前线程
static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }
```

#### parkAndCheckInterrupt 方法

```java
//挂起当前线程，并检查是否中断
private final boolean parkAndCheckInterrupt() {
    	//挂起当前线程
        LockSupport.park(this);
    	//检查当前线程是否中断
        return Thread.interrupted();
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

#### doAcquireInterruptibly 方法

```java
//获取独占锁，如果获取失败，把线程挂起并抛出异常
private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
    	//添加独占模式节点到队尾
        final Node node = addWaiter(Node.EXCLUSIVE);
    	//设置失败标志为true
        boolean failed = true;
        try {
            for (;;) {
                //获取node的前驱节点
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    //如果node的前驱节点是头结点，并且获取锁成功
                    //把node设置为头结点
                    setHead(node);
                    //把前驱节点的后驱节点引用置为null
                    p.next = null; // help GC
                    //设置失败标志为false
                    failed = false;
                    //直接返回
                    return;
                }
                //如果没获取到锁，把线程挂起
                //如果挂起线程失败，抛出异常
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                //失败，取消node获取锁的尝试
                cancelAcquire(node);
        }
    }
```

#### doAcquireNanos 方法

```java
//获取独占锁， 指定超时时间
private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
    	//参数检查
        if (nanosTimeout <= 0L)
            return false;
    	//计算过期时间
        final long deadline = System.nanoTime() + nanosTimeout;
    	//添加独占模式节点到队尾
        final Node node = addWaiter(Node.EXCLUSIVE);
    	//设置失败标志为true
        boolean failed = true;
        try {
            //for无限循环
            for (;;) {
                //获取node的前驱节点
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    //如果node的前驱节点是头结点，并且获取锁成功
                    //把node设置为头结点
                    setHead(node);
                    //把前驱节点的后驱节点引用置为null
                    p.next = null; // help GC
                    //设置失败标志为false
                    failed = false;
                    //获取锁成功，返回true
                    return true;
                }
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L)
                    //超时，返回false
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    //如果没获取到锁，把线程挂起
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted())
                    //如果线程被中断，抛出异常
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                //失败，取消node获取锁的尝试
                cancelAcquire(node);
        }
    }
```

####  doAcquireShared 方法

```java
//获取共享锁
private void doAcquireShared(int arg) {
    	//添加共享模式节点到队尾
        final Node node = addWaiter(Node.SHARED);
    	//设置失败标志为true
        boolean failed = true;
        try {
            //设置中断标志为true
            boolean interrupted = false;
            //for无限循环
            for (;;) {
                //获取node的前驱节点
                final Node p = node.predecessor();
                if (p == head) {
                    //如果前驱节点是头结点
                    //尝试获取共享锁
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        //获取成功，设置头结点为node节点， 并继续传播
                        setHeadAndPropagate(node, r);
                        //设置前驱节点的后驱节点引用置为null
                        p.next = null; // help GC
                        if (interrupted)
                            //如果中断标志为true，中断当前线程
                            selfInterrupt();
                        //设置失败标志为false
                        failed = false;
                        //直接返回
                        return;
                    }
                }
                //如果没获取到锁，把线程挂起
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    //挂起线程成功，设置中断标志为true
                    interrupted = true;
            }
        } finally {
            if (failed)
                //失败，取消node获取锁的尝试
                cancelAcquire(node);
        }
    }
```

#### doAcquireSharedInterruptibly 方法

```java
//获取共享锁
private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
    	//添加共享模式节点到队尾
        final Node node = addWaiter(Node.SHARED);
    	//设置失败标志为true
        boolean failed = true;
        try {
            //for无限循环
            for (;;) {
                //获取node的前驱节点
                final Node p = node.predecessor();
                if (p == head) {
                    //如果前驱节点是头结点
                    //尝试获取共享锁
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        //获取成功，设置头结点为node节点， 并继续传播
                        setHeadAndPropagate(node, r);
                        //设置前驱节点的后驱节点引用置为null
                        p.next = null; // help GC
                        //设置失败标志为false
                        failed = false;
                        //直接返回
                        return;
                    }
                }
                //如果没获取到锁，把线程挂起
                //如果挂起线程失败，抛出异常
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                //失败，取消node获取锁的尝试
                cancelAcquire(node);
        }
    }
```

#### doAcquireSharedNanos 方法

```java
//获取共享锁，指定超时时间
private boolean doAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
    	//参数检查
        if (nanosTimeout <= 0L)
            return false;
    	//计算过期时间
        final long deadline = System.nanoTime() + nanosTimeout;
    	//添加共享模式节点到队尾
        final Node node = addWaiter(Node.SHARED);
    	//设置失败标志为true
        boolean failed = true;
        try {
            //for无限循环
            for (;;) {
                //获取node的前驱节点
                final Node p = node.predecessor();
                if (p == head) {
                     //如果前驱节点是头结点
                    //尝试获取共享锁
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                         //获取成功，设置头结点为node节点， 并继续传播
                        setHeadAndPropagate(node, r);
                        //设置前驱节点的后驱节点引用置为null
                        p.next = null; // help GC
                        //设置失败标志为false
                        failed = false;
                        //直接返回
                        return true;
                    }
                }
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L)
                    //超时，返回false
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    //如果没获取到锁，把线程挂起
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted())
                    //如果线程被中断，抛出异常
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                //失败，取消node获取锁的尝试
                cancelAcquire(node);
        }
    }
```

#### tryAcquire 方法

```java
//尝试获取独占锁，强制子类实现
protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
```

#### tryRelease 方法

```java
//尝试释放锁，强制子类实现
protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
```

#### tryAcquireShared 方法

```java
//尝试获取共享锁，强制子类实现
protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
    }
```

#### tryReleaseShared 方法

```java
//尝试释放共享锁，强制子类实现
protected boolean tryReleaseShared(int arg) {
        throw new UnsupportedOperationException();
    }
```

#### isHeldExclusively 方法

```java
//返回锁的模式， true 独占锁
protected boolean isHeldExclusively() {
        throw new UnsupportedOperationException();
    }
```

#### acquire 方法

```java
//独占模式获取锁
public final void acquire(int arg) {
    	//先尝试获取锁，如果失败，以独占模式添加到队尾，中断当前线程等着被唤醒
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

#### acquireInterruptibly 方法

```java
//独占模式获取锁，获取失败抛出异常
public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }
```

#### tryAcquireNanos 方法

```java
//独占模式获取锁，设置超时时间
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }
```

#### release 方法

```java
//释放独占锁
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            //释放独占锁成功
            //拷贝头节点的副本
            Node h = head;
            if (h != null && h.waitStatus != 0)
                //如果头节点不为null 并且 头节点的状态正常
                //唤醒下一个节点
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

#### acquireShared 方法

```java
//共享模式获取锁
public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

#### acquireSharedInterruptibly 方法

```java
//共享模式获取锁，获取失败抛出异常
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```

#### tryAcquireSharedNanos 方法

```java
//共享模式获取锁,设置超时时间
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquireShared(arg) >= 0 ||
            doAcquireSharedNanos(arg, nanosTimeout);
    }
```

#### releaseShared 方法

```java
//释放共享锁
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

#### hasQueuedThreads 方法

```java
//队列中是否含有线程
public final boolean hasQueuedThreads() {
        return head != tail;
    }
```

#### hasContended 方法

```java
//当前是否有线程在使用锁
public final boolean hasContended() {
        return head != null;
    }
```

#### getFirstQueuedThread 方法

```java
//获取队列中第一个线程（等待时间最长的线程）
public final Thread getFirstQueuedThread() {
        // handle only fast path, else relay
    	//为空的话返回null
    	//调用fullGetFirstQueuedThread方法返回队列中第一个线程
        return (head == tail) ? null : fullGetFirstQueuedThread();
    }
```

#### fullGetFirstQueuedThread 方法

```java
//获取队列中第一个线程
private Thread fullGetFirstQueuedThread() {
        //一般情况下，head的后驱节点就是队列中的第一个元素
    	//如果head不为null，并且head的后驱节点不为null，并且head的后驱节点的前驱节点是head节点
    	//直接返回head的后驱节点
        Node h, s;
        Thread st;
        if (((h = head) != null && (s = h.next) != null &&
             s.prev == head && (st = s.thread) != null) ||
            ((h = head) != null && (s = h.next) != null &&
             s.prev == head && (st = s.thread) != null))
            return st;
		//有可能还没来的及设置head的后驱节点（多个线程操作）
    	//这种情况下，从tail往前找
        Node t = tail;
        Thread firstThread = null;
        while (t != null && t != head) {
            Thread tt = t.thread;
            if (tt != null)
                firstThread = tt;
            t = t.prev;
        }
        return firstThread;
    }
```

#### isQueued 方法

```java
//查看给定线程是否正在排队
public final boolean isQueued(Thread thread) {
        if (thread == null)
            throw new NullPointerException();
    	//for循环遍历，从tail往前找
        for (Node p = tail; p != null; p = p.prev)
            if (p.thread == thread)
                return true;
        return false;
    }
```

#### getQueueLength 方法

```java
//获取队列的长度
public final int getQueueLength() {
        int n = 0;
        for (Node p = tail; p != null; p = p.prev) {
            if (p.thread != null)
                ++n;
        }
        return n;
    }
```

#### getQueuedThreads 方法

```java
//获取队列中的所有线程，以集合的方式返回
public final Collection<Thread> getQueuedThreads() {
        ArrayList<Thread> list = new ArrayList<Thread>();
        for (Node p = tail; p != null; p = p.prev) {
            Thread t = p.thread;
            if (t != null)
                list.add(t);
        }
        return list;
    }
```

#### getExclusiveQueuedThreads 方法

```java
//获取队列中的所有独占模式的线程，以集合的方式返回
public final Collection<Thread> getExclusiveQueuedThreads() {
        ArrayList<Thread> list = new ArrayList<Thread>();
        for (Node p = tail; p != null; p = p.prev) {
            if (!p.isShared()) {
                Thread t = p.thread;
                if (t != null)
                    list.add(t);
            }
        }
        return list;
    }
```

#### getSharedQueuedThreads 方法

```java
//获取队列中的所有共享模式的线程，以集合的方式返回
public final Collection<Thread> getSharedQueuedThreads() {
        ArrayList<Thread> list = new ArrayList<Thread>();
        for (Node p = tail; p != null; p = p.prev) {
            if (p.isShared()) {
                Thread t = p.thread;
                if (t != null)
                    list.add(t);
            }
        }
        return list;
    }
```

