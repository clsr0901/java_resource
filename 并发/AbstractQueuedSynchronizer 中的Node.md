### 1. 类的定义

```java
static final class Node
```

从类的定义可以看出

* Node是AbstractQueuedSynchronizer中定义的一个静态内部类
* Node是final修饰的，不可继承

### 2. 字段属性

```java
//标记节点正在共享模式下等待锁
static final Node SHARED = new Node();
//标记节点正在独占模式下等待锁
static final Node EXCLUSIVE = null;

//waitStatus的值，表示线程已经取消
static final int CANCELLED =  1;
//waitStatus的值，表示后面的线程需要被唤醒
static final int SIGNAL    = -1;
//waitStatus的值，表示线程正在等待条件
static final int CONDITION = -2;
//waitStatus的值，表示下一个等待共享锁的线程应该无条件的传播
static final int PROPAGATE = -3;

/**
         * Status field, taking on only the values:
         *   SIGNAL:     The successor of this node is (or will soon be)
         *               blocked (via park), so the current node must
         *               unpark its successor when it releases or
         *               cancels. To avoid races, acquire methods must
         *               first indicate they need a signal,
         *               then retry the atomic acquire, and then,
         *               on failure, block.
         *   CANCELLED:  This node is cancelled due to timeout or interrupt.
         *               Nodes never leave this state. In particular,
         *               a thread with cancelled node never again blocks.
         *   CONDITION:  This node is currently on a condition queue.
         *               It will not be used as a sync queue node
         *               until transferred, at which time the status
         *               will be set to 0. (Use of this value here has
         *               nothing to do with the other uses of the
         *               field, but simplifies mechanics.)
         *   PROPAGATE:  A releaseShared should be propagated to other
         *               nodes. This is set (for head node only) in
         *               doReleaseShared to ensure propagation
         *               continues, even if other operations have
         *               since intervened.
         *   0:          None of the above
         *
         * The values are arranged numerically to simplify use.
         * Non-negative values mean that a node doesn't need to
         * signal. So, most code doesn't need to check for particular
         * values, just for sign.
         *
         * The field is initialized to 0 for normal sync nodes, and
         * CONDITION for condition nodes.  It is modified using CAS
         * (or when possible, unconditional volatile writes).
         */

//等待状态，值为上面五种中的一种
volatile int waitStatus;
//前驱节点
volatile Node prev;
//后驱节点
volatile Node next;
//该节点排队的线程
volatile Thread thread;
//链接到等待条件的下一个节点
Node nextWaiter;
```

从字段属性可以看出

* Node是一个双向链表
* Node分为共享或者独占两种模式
* Node中保存了要执行的线程Thread对象
* Node中保存了当前等待的状态，共有五种状态

### 3. 构造方法

```java
//空构造函数， 用来初始化AbstractQueuedSynchronizer中的head或者共享模式的Node
Node() {    // Used to establish initial head or SHARED marker
        }

//传入一个线程和Node对象，在AbstractQueuedSynchronizer中的addWaiter方法中使用
Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

//传入一个线程和等待状态，用于Condition
Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
```

从构造方法中可以看出

* AbstractQueuedSynchronizer中的head 不带Threa对象，所以head只是作为标记使用
* Node默认是共享模式的
* Node中的Thread是核心

### 4. 方法

#### isShared 方法

```java
//获取当前节点的模式
final boolean isShared() {
            return nextWaiter == SHARED;
        }
```

#### predecessor 方法

```java
//获取前驱节点，如果前驱节点不存在抛出异常
final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }
```

