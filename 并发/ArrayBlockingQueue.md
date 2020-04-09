### 1. 类的定义

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable
```

从类的定义中可以看出

* ArrayBlockingQueue是一个泛型类
* ArrayBlockingQueue继承了AbstractQueue类，AbstractQueue是一个抽象类（模板方法）
* ArrayBlockingQueue实现了BlockingQueue接口，表示一个阻塞队列
* ArrayBlockingQueue实现了java.io.Serializable，表示支持序列化

### 2. 字段属性

```java
//序列化版本号
private static final long serialVersionUID = -817911632652898426L;
//队列的元素
final Object[] items;
//下一个获取的元素下标，包括take, poll, peek or remove 方法
int takeIndex;
//下一个添加的元素下标，包括put, offer, or add方法
int putIndex;
//队列的元素个数
int count;
//锁
final ReentrantLock lock;
//等待获取元素的条件
private final Condition notEmpty;
//等待放入元素的条件
private final Condition notFull;
//迭代器，这是一个内部类
transient Itrs itrs = null;
```

从字段属性可以看出

* ArrayBlockingQueue 的本质是一个Object数组
* ArrayBlockingQueue 通过ReentrantLock来控制同步操作
* ArrayBlockingQueue 有唤醒取元素（队列空了）和存元素（队列满了）的条件
* ArrayBlockingQueue 保存了操作的下标，环形队列？

### 3. 构造方法

```java
//传入容量
public ArrayBlockingQueue(int capacity) {
    	//转调下面的构造方法，false表示非公平锁
        this(capacity, false);
    }
//传入容量和锁的类型
 public ArrayBlockingQueue(int capacity, boolean fair) {
     	//参数检查
        if (capacity <= 0)
            throw new IllegalArgumentException();
     	//初始化数组
        this.items = new Object[capacity];
     	//初始化ReentrantLock锁和条件
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
//传入容量和锁的类型和集合数据
public ArrayBlockingQueue(int capacity, boolean fair,
                              Collection<? extends E> c) {
    	//转调上面的构造函数
        this(capacity, fair);
		//加锁，遍历集合添加数据
        final ReentrantLock lock = this.lock;
        lock.lock(); // Lock only for visibility, not mutual exclusion
        try {
            int i = 0;
            try {
                for (E e : c) {
                    checkNotNull(e);
                    items[i++] = e;
                }
            } catch (ArrayIndexOutOfBoundsException ex) {
                throw new IllegalArgumentException();
            }
            //设置当前元素个数
            count = i;
            //设置下一个要存放的元素下标
            putIndex = (i == capacity) ? 0 : i;
        } finally {
            //解锁
            lock.unlock();
        }
    }
```

从构造方法可以看出

* 构造方法主要是做初始化操作
* 构造方法可以传入一个集合数据来初始化

### 4. 方法

#### add 方法

```java
//添加元素到队尾
public boolean add(E e) {
    	//调用父类的方法
        return super.add(e);
    }
//父类的添加方法
 public boolean add(E e) {
     	//实质是调用offer方法
        if (offer(e))
            return true;
        else
            throw new IllegalStateException("Queue full");
    }
```

#### offer 方法

```java
//添加制定元素到队尾，如果队列满了直接返回false
public boolean offer(E e) {
        checkNotNull(e);
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count == items.length)
                //如果队列满了，直接返回false
                return false;
            else {
                //队列还有空间，调用enqueue方法入队
                enqueue(e);
                return true;
            }
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
//添加制定元素到队尾，如果队列满了有超时时间的等待队列有元素出队
public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        checkNotNull(e);
    	//计算超时时间
        long nanos = unit.toNanos(timeout);
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length) {
                if (nanos <= 0)
                    //超时返回false
                    return false;
                //带超时时间的条件等待
                nanos = notFull.awaitNanos(nanos);
            }
            //未超时并且队列有空间，调用enqueue添加元素
            enqueue(e);
            return true;
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### enqueue 方法

```java
//元素入队
private void enqueue(E x) {
       	//获取数组的副本
        final Object[] items = this.items;
    	//把队尾元素设置为传入的元素
        items[putIndex] = x;
    	//把队尾位置加1
        if (++putIndex == items.length)
            //如果队尾位置等于数组长度，队尾位置置为0，环形队列
            putIndex = 0;
    	//元素数量加1
        count++;
    	//唤醒等待队列有元素入队的线程
        notEmpty.signal();
    }
```

#### put 方法

```java
//添加制定元素到队尾，如果队列满了一直等待队列有元素出队
public void put(E e) throws InterruptedException {
        checkNotNull(e);
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                //while循环，如果队列满了一直等待
                notFull.await();
            //队列还有空间，调用enqueue方法入队
            enqueue(e);
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### poll 方法

```java
//获取元素，如果队列没有元素直接返回null
public E poll() {
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //如果队列有元素，调用dequeue获取元素，否则返回null
            return (count == 0) ? null : dequeue();
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
//获取元素，如果队列没有元素，有超时时间的等待队列有元素入队
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    	//计算超时时间
        long nanos = unit.toNanos(timeout);
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0) {
                if (nanos <= 0)
                    //超时返回null
                    return null;
                //带超时时间的条件等待
                nanos = notEmpty.awaitNanos(nanos);
            }
            //未超时并且队列有元素,调用dequeue方法获取元素
            return dequeue();
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### dequeue 方法

```java
//出队
private E dequeue() {
        //获取数组的副本
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
    	//获取队列起始元素
        E x = (E) items[takeIndex];
    	//把队列起始位置置为null
        items[takeIndex] = null;
    	//把队列起始位置加1
        if (++takeIndex == items.length)
            //如果队列起始位置等于数组长度，把队列起始位置设置为0
            takeIndex = 0;
    	//元素数量减1
        count--;
        if (itrs != null)
            //如果迭代器不为null，迭代器进行出队操作
            itrs.elementDequeued();
    	 //唤醒等待队列有元素出队的线程
        notFull.signal();
    	//返回出队的元素
        return x;
    }
```

#### take 方法

```java
//获取元素，如果队列没有元素，一直等待队列有元素入队
public E take() throws InterruptedException {
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                //while循环，如果队列没有元素，一直带条件等待
                notEmpty.await();
            //调用dequeue方法获取元素
            return dequeue();
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### peek 方法

```java
//获取元素，元素不出队
public E peek() {
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //调用itemAt方法获取元素
            return itemAt(takeIndex); // null when queue is empty
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### itemAt 方法

```java
//返回指定位置的元素
final E itemAt(int i) {
        return (E) items[i];
    }
```

#### size 方法

```java
//获取队列元素数量
public int size() {
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //返回元素数量
            return count;
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### remainingCapacity 方法

```java
//获取剩余容量
public int remainingCapacity() {
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //队列容量-元素容量=剩余容量
            return items.length - count;
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### remove 方法

```java
//移除指定元素
public boolean remove(Object o) {
    	//参数检查
        if (o == null) return false;
    	//获取队列的副本
        final Object[] items = this.items;
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count > 0) {
                //如果有元素
                //获取队尾下标
                final int putIndex = this.putIndex;
                //获取队列起始的下标
                int i = takeIndex;
                //while遍历查找
                do {
                    if (o.equals(items[i])) {
                        //如果找到，调用removeAt方法移除元素
                        removeAt(i);
                        return true;
                    }
                    if (++i == items.length)
                        //如果达到数组的尾部，把i置为0.这里表示数组是循环队列，数组收尾相连
                        i = 0;
                    //从队首一直找到队尾
                } while (i != putIndex);
            }
            //队列没有元素直接返回false
            return false;
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### removeAt 方法

```java
//移除指定位置的元素
void removeAt(final int removeIndex) {
    	//获取队列的副本
        final Object[] items = this.items;
        if (removeIndex == takeIndex) {
            //移除位置等于起始位置， 表示移除第一个元素
            //把对应位置置为null
            items[takeIndex] = null;
            //起始位置加1
            if (++takeIndex == items.length)
                //如果起始位置加1等于数组长度，把起始位置置为0，环形队列
                takeIndex = 0;
            //元素数量减1
            count--;
            if (itrs != null)
                //如果有迭代器，进行出队操作
                itrs.elementDequeued();
        } else {
            //移除的元素不是第一个元素
            //从删除位置开始，循环把后一个元素往前移动一位，并把最后一位置为null
            
           //获取队尾下标副本
            final int putIndex = this.putIndex;
            //从移除位置开始for循环
            for (int i = removeIndex;;) {
                //获取下一个元素的下标
                int next = i + 1;
                if (next == items.length)
                    //判断如果达到数组尾部，把下标置为0
                    next = 0;
                if (next != putIndex) {
                    //当前下标没有达到队尾
                    //使用后一个元素替换前一个元素
                    items[i] = items[next];
                    //把下标指向下一个元素继续遍历
                    i = next;
                } else {
                    //到达队尾
                    //把最后一个元素置为null
                    items[i] = null;
                    //把队尾位置往前移动1位
                    this.putIndex = i;
                    //中断循环
                    break;
                }
            }
            //元素数量减1
            count--;
            if (itrs != null)
                //如果有迭代器，移除对应位置的元素
                itrs.removedAt(removeIndex);
        }
    	//唤醒等待队列有元素出队的线程
        notFull.signal();
    }
```



#### contains 方法

```java
//查看队列是否包含指定元素
public boolean contains(Object o) {
    	//参数检查
        if (o == null) return false;
    	//获取队列的副本
        final Object[] items = this.items;
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count > 0) {
                //如果有元素
                //获取队尾下标
                final int putIndex = this.putIndex;
                //获取队列起始的下标
                int i = takeIndex;
                do {
                    if (o.equals(items[i]))
                        //如果找到，返回true
                        return true;
                    if (++i == items.length)
                        //如果达到数组的尾部，把i置为0.这里表示数组是循环队列，数组收尾相连
                        i = 0;
                     //从队首一直找到队尾
                } while (i != putIndex);
            }
            //队列没有元素直接返回false
            return false;
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### toArray 方法

```java
//转换为数组，使用System.arraycopy拷贝队首到队尾的元素，拷贝数量为count
public Object[] toArray() {
        Object[] a;
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final int count = this.count;
            a = new Object[count];
            int n = items.length - takeIndex;
            if (count <= n)
                System.arraycopy(items, takeIndex, a, 0, count);
            else {
                System.arraycopy(items, takeIndex, a, 0, n);
                System.arraycopy(items, 0, a, n, count - n);
            }
        } finally {
            lock.unlock();
        }
        return a;
    }

//队列元素添加到指定数组，跟上面的方法一样
public <T> T[] toArray(T[] a) {
        final Object[] items = this.items;
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final int count = this.count;
            final int len = a.length;
            if (len < count)
                a = (T[])java.lang.reflect.Array.newInstance(
                    a.getClass().getComponentType(), count);
            int n = items.length - takeIndex;
            if (count <= n)
                //前后有空位
                System.arraycopy(items, takeIndex, a, 0, count);
            else {
                //中间有空位
                System.arraycopy(items, takeIndex, a, 0, n);
                System.arraycopy(items, 0, a, n, count - n);
            }
            if (len > count)
                a[count] = null;
        } finally {
            lock.unlock();
        }
        return a;
    }
```

#### clear 方法

```java
//清空所有元素
public void clear() {
    	//获取队列副本
        final Object[] items = this.items;
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            int k = count;
            if (k > 0) {
                //如果有元素
                //循环置为null
                final int putIndex = this.putIndex;
                int i = takeIndex;
                do {
                    items[i] = null;
                    if (++i == items.length)
                        i = 0;
                } while (i != putIndex);
                takeIndex = putIndex;
                //元素数量置为0
                count = 0;
                if (itrs != null)
                    itrs.queueIsEmpty();
                for (; k > 0 && lock.hasWaiters(notFull); k--)
                    //唤醒等待队列有元素出队的线程
                    notFull.signal();
            }
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### drainTo 方法

```java
//移除并且添加到指定集合
public int drainTo(Collection<? super E> c) {
    	//调用下面的方法
        return drainTo(c, Integer.MAX_VALUE);
    }
//移除指定数量的元素添加到指定集合
public int drainTo(Collection<? super E> c, int maxElements) {
    	//参数检查
        checkNotNull(c);
        if (c == this)
            throw new IllegalArgumentException();
        if (maxElements <= 0)
            return 0;
    	//获取队列的副本
        final Object[] items = this.items;
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //计算需要移除的元素数量
            int n = Math.min(maxElements, count);
            //队列起始元素的下标
            int take = takeIndex;
            int i = 0;
            try {
                //while循环处理逻辑
                while (i < n) {
                    //获取元素
                    E x = (E) items[take];
                    //把获取的元素添加到指定集合
                    c.add(x);
                    //清除队列的引用
                    items[take] = null;
                    if (++take == items.length)
                        //到队尾了把下标置为0， 环形队列
                        take = 0;
                    i++;
                }
                return n;
            } finally {
                // Restore invariants even if c.add() threw
                if (i > 0) {
                    //重新计算队列元素数量
                    count -= i;
                    //移动队列起始的位置
                    takeIndex = take;
                    if (itrs != null) {
                        if (count == 0)
                            //迭代器置为null
                            itrs.queueIsEmpty();
                        else if (i > take)
                            //迭代器移除对应元素
                            itrs.takeIndexWrapped();
                    }
                    for (; i > 0 && lock.hasWaiters(notFull); i--)
                         //唤醒等待队列有元素出队的线程
                        notFull.signal();
                }
            }
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### iterator 方法

```java
//获取迭代器
public Iterator<E> iterator() {
    	//Itr是内部类，可以获取外部类所有的信息
        return new Itr();
    }
```





