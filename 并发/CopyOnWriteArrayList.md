### 1. 类的定义

```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

从类的定义中可以看出

* CopyOnWriteArrayList是一个泛型类
* CopyOnWriteArrayList实现了List接口，表示它是一个集合
* CopyOnWriteArrayList 实现了RandomAccess接口，表示支持随机访问
* CopyOnWriteArrayList实现了Cloneable接口，表示支持克隆
* CopyOnWriteArrayList实现了java.io.Serializable接口，表示支持序列化

### 2. 字段属性

```java
//序列化版本号
private static final long serialVersionUID = 8673264195747942595L;
//锁，访问数组的时候使用
final transient ReentrantLock lock = new ReentrantLock();
//Object数组，存储元素的容器
private transient volatile Object[] array;
```

从字段属性可以看出

* CopyOnWriteArrayList的底层是一个Object数组
* CopyOnWriteArrayList通过ReentrantLock来实现同步

### 3. 构造方法

```java
//默认空构造方法
public CopyOnWriteArrayList() {
    	//创建一个空集合
        setArray(new Object[0]);
    }

//传入一个集合对象的构造方法
public CopyOnWriteArrayList(Collection<? extends E> c) {
        Object[] elements;
        if (c.getClass() == CopyOnWriteArrayList.class)
            //如果传入的是一个CopyOnWriteArrayList集合对象
            //直接获取array数组
            elements = ((CopyOnWriteArrayList<?>)c).getArray();
        else {
            //如果是其他的集合对象
            //先获取集合对象的元素，并转换为数组
            elements = c.toArray();
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elements.getClass() != Object[].class)
                //有可能c的元素不是Object对象
                //使用Arrays.copyOf来做强转Object操作
                elements = Arrays.copyOf(elements, elements.length, Object[].class);
        }
    	//把上面获取到的Object数组赋值给当前对象
        setArray(elements);
    }
//传入一个数组对象
public CopyOnWriteArrayList(E[] toCopyIn) {
    	//使用Arrays.copyOf来做强转Object操作,再赋值给当前对象
        setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
    }
```

从构造方法可以看出

* CopyOnWriteArrayList默认是一个长度为0的数组
* CopyOnWriteArrayList可以接收一个集合对象来初始化
* CopyOnWriteArrayList可以接收一个数组对象来初始化

### 4. 方法

#### getArray 方法

```java
//获取array数组
final Object[] getArray() {
        return array;
    }
```

#### setArray 方法

```java
//设置array数组
final void setArray(Object[] a) {
        array = a;
    }
```

#### size 方法

```java
//获取列表的长度
public int size() {
    	//返回数组的长度
        return getArray().length;
    }
```

#### isEmpty 方法

```java
//列表是否为空
public boolean isEmpty() {
    	//数组长度是否为0
        return size() == 0;
    }
```

#### eq 方法

```java
//判断两个对象是否相等
private static boolean eq(Object o1, Object o2) {
    	//如果一个为null，判断另一个是否为null
    	//否则使用equals来比较
        return (o1 == null) ? o2 == null : o1.equals(o2);
    }
```

#### indexOf 方法

```java
//获取指定区间指定元素的下标，如果元素重复只返回第一个下标
private static int indexOf(Object o, Object[] elements,
                               int index, int fence) {
        if (o == null) {
            //如果对象为null，使用for循环遍历并使用==判断
            for (int i = index; i < fence; i++)
                if (elements[i] == null)
                    return i;
        } else {
            //如果对象不为null，使用for循环遍历并使用equals判断
            for (int i = index; i < fence; i++)
                if (o.equals(elements[i]))
                    return i;
        }
    	//没找到返回-1
        return -1;
    }

//获取指定元素的下标
public int indexOf(Object o) {
        Object[] elements = getArray();
    	//调用上面的方法查找
        return indexOf(o, elements, 0, elements.length);
    }
//从指定开始位置查找指定元素的下标
public int indexOf(E e, int index) {
        Object[] elements = getArray();
    	//调用第一个方法
        return indexOf(e, elements, index, elements.length);
    }
```

#### lastIndexOf 方法

```java
//获取指定结束位置指定元素的最后一个下标，如果元素重复只返回最后一个下标
private static int lastIndexOf(Object o, Object[] elements, int index) {
        if (o == null) {
            //如果对象为null，使用for循环遍历并使用==判断
            //从指定位置index往前遍历
            for (int i = index; i >= 0; i--)
                if (elements[i] == null)
                    return i;
        } else {
            //如果对象不为null，使用for循环遍历并使用equals判断
            //从指定位置index往前遍历
            for (int i = index; i >= 0; i--)
                if (o.equals(elements[i]))
                    return i;
        }
    	//没找到返回-1
        return -1;
    }
 //获取指定元素的最后一个下标，如果元素重复只返回最后一个下标
 public int lastIndexOf(Object o) {
        Object[] elements = getArray();
     	//调用上面的方法
        return lastIndexOf(o, elements, elements.length - 1);
    }
//从指定结束位置查找指定元素的最后一个下标，如果元素重复只返回最后一个下标
public int lastIndexOf(E e, int index) {
        Object[] elements = getArray();
        return lastIndexOf(e, elements, index);
    }
```

#### contains 方法

```java
//集合中是否包含指定元素
public boolean contains(Object o) {
    	//获取数组副本
        Object[] elements = getArray();
    	//调用indexOf方法查找指定元素
        return indexOf(o, elements, 0, elements.length) >= 0;
    }
```

#### clone 方法

```java
public Object clone() {
        try {
            @SuppressWarnings("unchecked")
            //调用父类的clone方法，浅拷贝
            CopyOnWriteArrayList<E> clone =
                (CopyOnWriteArrayList<E>) super.clone();
            //重置锁
            clone.resetLock();
            return clone;
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError();
        }
    }
```

#### toArray 方法

```java
//转化为数组
public Object[] toArray() {
        Object[] elements = getArray();
    	//通过Arrays.copyOf来拷贝数组
        return Arrays.copyOf(elements, elements.length);
    }
//把集合元素复制到指定数组中
public <T> T[] toArray(T a[]) {
    	//获取数组的副本
        Object[] elements = getArray();
    	//获取数组长度
        int len = elements.length;
        if (a.length < len)
            //如果目标数组的长度小于当前数组的长度
            //使用Arrays.copyOf进行拷贝
            return (T[]) Arrays.copyOf(elements, len, a.getClass());
        else {
            //如果目标数组的长度大于等于当前数组长度
            //直接使用System.arraycopy来进行拷贝，拷贝长度为当前数组长度
            System.arraycopy(elements, 0, a, 0, len);
            if (a.length > len)
                //把最后一个置为null
                a[len] = null;
            //返回目标数组
            return a;
        }
    }
```

#### get 方法

```java
//获取指定数组的指定下标的元素
private E get(Object[] a, int index) {
        return (E) a[index];
    }

//获取指定下标的元素
public E get(int index) {
    	//调用上面的方法
        return get(getArray(), index);
    }
```

#### set 方法

```java
//把指定元素设置到指定位置
public E set(int index, E element) {
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取数组副本
            Object[] elements = getArray();
            //获取指定位置的旧值
            E oldValue = get(elements, index);
            if (oldValue != element) {
                //如果旧值不等于新值
                //获取数组长度
                int len = elements.length;
                //拷贝一份新的数组
                Object[] newElements = Arrays.copyOf(elements, len);
                //在指定位置设置新值
                newElements[index] = element;
                //设置新的数组
                setArray(newElements);
            } else {
                // Not quite a no-op; ensures volatile write semantics
                //新值等于旧值，没有任何操作，确保语义上的volatile写
                setArray(elements);
            }
            //返回旧值
            return oldValue;
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### add 方法

```java
//添加指定元素
public boolean add(E e) {
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取数组副本
            Object[] elements = getArray();
            //获取数组长度
            int len = elements.length;
            //复制数组并把长度扩容为数组len+1
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            //设置指定元素
            newElements[len] = e;
            //设置新的数组
            setArray(newElements);
            return true;
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }

//添加指定元素到指定位置
public void add(int index, E element) {
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取数组副本
            Object[] elements = getArray();
            //获取数组长度
            int len = elements.length;
            //参数验证
            if (index > len || index < 0)
                throw new IndexOutOfBoundsException("Index: "+index+
                                                    ", Size: "+len);
            Object[] newElements;
            //计算需要移动的数量，从插入位置，后面的元素统一向后移动一位
            int numMoved = len - index;
            if (numMoved == 0)
                //如果移动位置为0，表示末尾插入
               	//跟上面的方法一样了
                //复制数组并把长度扩容为数组len+1
                newElements = Arrays.copyOf(elements, len + 1);
            else {
                //插入的位置在中间的情况
                //创建新的容量为len+1的数组
                newElements = new Object[len + 1];
                //拷贝0到index的元素
                System.arraycopy(elements, 0, newElements, 0, index);
                //拷贝index+1到末尾的元素
                System.arraycopy(elements, index, newElements, index + 1,
                                 numMoved);
            }
            //设置index的元素为新元素
            newElements[index] = element;
            //设置新的数组
            setArray(newElements);
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### remove 方法

```java
//移除指定位置的元素并返回指定位置的值
public E remove(int index) {
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取数组的副本
            Object[] elements = getArray();
            //获取数组的长度
            int len = elements.length;
            //获取指定元素的旧值
            E oldValue = get(elements, index);
            //计算移动的数量，移除元素相当于把移除位置后所有的元素向前移动一位
            int numMoved = len - index - 1;
            if (numMoved == 0)
                //如果移动数量为0， 表示移除末尾元素
                //把数组元素的0到len-1的元素拷贝就行
                setArray(Arrays.copyOf(elements, len - 1));
            else {
                //移除的元素不在末尾
                //创建len-1的新数组
                Object[] newElements = new Object[len - 1];
                //拷贝0到inde的元素
                System.arraycopy(elements, 0, newElements, 0, index);
                //拷贝index+1到末尾的元素
                System.arraycopy(elements, index + 1, newElements, index,
                                 numMoved);
                //设置新数组
                setArray(newElements);
            }
            //返回旧值
            return oldValue;
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }

//移除指定对象
public boolean remove(Object o) {
    	//获取数组对象
        Object[] snapshot = getArray();
    	//获取指定对象的下标
        int index = indexOf(o, snapshot, 0, snapshot.length);
    	//如果不存在返回false
    	//存在使用下面的方法移除对应位置的元素
        return (index < 0) ? false : remove(o, snapshot, index);
    }

//从指定数组中移除指定位置的元素
private boolean remove(Object o, Object[] snapshot, int index) {
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取数组副本
            Object[] current = getArray();
            //获取数组的长度
            int len = current.length;
            //如果数组被修改，快照不等于当前数组
            //这里有一个goto
            if (snapshot != current) findIndex: {
                //获取指定位置和长度的最小值
                int prefix = Math.min(index, len);
                //for循环，从0到index遍历
                for (int i = 0; i < prefix; i++) {
                    if (current[i] != snapshot[i] && eq(o, current[i])) {
                        //i的位置被修改，并且i的位置元素为o
                        //既然i的元素也是o，就算被修改了，移除i的元素也一样，反正只要移除o就行
                        //大概就是不断调整index的位置，因为可能被修改，而且可能存在多个o对象
                        //把index置为i，继续goto
                        index = i;
                        break findIndex;
                    }
                }
                if (index >= len)
                   	//如果下标大于数组长度，直接返回false
                    return false;
                if (current[index] == o)
                    //如果当前元素等于o，goto findIndex
                    break findIndex;
                //获取index到len区间，o的下标
                index = indexOf(o, current, index, len);
                if (index < 0)
                    //如果o不存在，直接返回false
                    return false;
            }
            //创建新的数组，长度为len-1
            Object[] newElements = new Object[len - 1];
            //拷贝0到index的元素
            System.arraycopy(current, 0, newElements, 0, index);
            //拷贝index+1到末尾的元素
            System.arraycopy(current, index + 1,
                             newElements, index,
                             len - index - 1);
            //设置新的数组
            setArray(newElements);
            return true;
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### removeRange 方法

```java
//移除指定区间的元素
void removeRange(int fromIndex, int toIndex) {
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取数组副本
            Object[] elements = getArray();
            //获取数组长度
            int len = elements.length;
			//参数检查
            if (fromIndex < 0 || toIndex > len || toIndex < fromIndex)
                throw new IndexOutOfBoundsException();
            //获取新的长度
            int newlen = len - (toIndex - fromIndex);
            //获取需要移动的数量，结束位置后面的所有元素都要往前移动
            int numMoved = len - toIndex;
            if (numMoved == 0)
                //如果移动数量为0，表示移除的是末尾的元素
                //只要拷贝前面的元素就行
                setArray(Arrays.copyOf(elements, newlen));
            else {
                //移除的元素位置在前面
                //创建新长度大小的数组
                Object[] newElements = new Object[newlen];
                //拷贝0到fromIndex的元素
                System.arraycopy(elements, 0, newElements, 0, fromIndex);
                //拷贝toIndex到末尾的元素
                System.arraycopy(elements, toIndex, newElements,
                                 fromIndex, numMoved);
                //设置新的数组
                setArray(newElements);
            }
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### addIfAbsent 方法

```java
//添加一个不存在的元素
public boolean addIfAbsent(E e) {
    	//获取数组副本
        Object[] snapshot = getArray();
    	//如果元素存在返回false
    	//元素不存在，调用addIfAbsent方法添加
        return indexOf(e, snapshot, 0, snapshot.length) >= 0 ? false :
            addIfAbsent(e, snapshot);
    }

//在指定的数组中添加一个指定元素
private boolean addIfAbsent(E e, Object[] snapshot) {
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取数组副本
            Object[] current = getArray();
            //获取数组长度
            int len = current.length;
            if (snapshot != current) {
                //如果数组被修改
                // Optimize for lost race to another addXXX operation
                //获取最小的长度
                int common = Math.min(snapshot.length, len);
                for (int i = 0; i < common; i++)
                    if (current[i] != snapshot[i] && eq(e, current[i]))
                        //如果元素被其它线程添加进来，直接返回false
                        //current[i] != snapshot[i]表示其中有个被修改
                        //eq(e, current[i]) 表示当前数组中已经存在指定对象e
                        return false;
                if (indexOf(e, current, common, len) >= 0)
                    	//如果已经存在指定对象直接返回false
                        return false;
            }
            //创建一个新的数组，长度为len+1
            Object[] newElements = Arrays.copyOf(current, len + 1);
            //把指定元素添加到新数组的末尾
            newElements[len] = e;
            //设置新的数组
            setArray(newElements);
            return true;
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### containsAll 方法

```java
//查看是否包含指定集合中所有的元素
public boolean containsAll(Collection<?> c) {
    	//获取数组副本
        Object[] elements = getArray();
    	//获取数组的长度
        int len = elements.length;
    	//for循环遍历传入的集合
        for (Object e : c) {
            if (indexOf(e, elements, 0, len) < 0)
                //如果有元素不存在，返回false
                return false;
        }
    	//都存在，返回true
        return true;
    }
```

#### removeAll 方法

```java
//移除集合中所有的元素
public boolean removeAll(Collection<?> c) {
    	//参数检查
        if (c == null) throw new NullPointerException();
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取数组副本
            Object[] elements = getArray();
            //获取数组的长度
            int len = elements.length;
            if (len != 0) {
                //如果数组中有元素
                // temp array holds those elements we know we want to keep
                //新的长度
                int newlen = 0;
                //创建一个长度为len的数组，来存放未被删除的数组，数组真实数据长度为newlen
                Object[] temp = new Object[len];
                //for循环遍历
                for (int i = 0; i < len; ++i) {
                    //获取当前位置的元素
                    Object element = elements[i];
                    if (!c.contains(element))
                        //如果集合中不包含当前元素
                        //把添加元素添加到新数组中去，并把newlen加1
                        temp[newlen++] = element;
                }
                if (newlen != len) {
                    //如果新数组中有元素
                    //把新数组中的有效元素拷贝到数组中去
                    setArray(Arrays.copyOf(temp, newlen));
                    return true;
                }
            }
            return false;
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### retainAll 方法

```java
//设置指定集合和当前数组的交集
public boolean retainAll(Collection<?> c) {
    	//参数检查
        if (c == null) throw new NullPointerException();
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取数组副本
            Object[] elements = getArray();
            //获取数组的长度
            int len = elements.length;
            if (len != 0) {
                //如果数组中有元素
                // temp array holds those elements we know we want to keep
                //新的长度
                int newlen = 0;
                //创建一个长度为len的数组，来存放交集的数据，数组真实数据长度为newlen
                Object[] temp = new Object[len];
                //for循环遍历
                for (int i = 0; i < len; ++i) {
                     //获取当前位置的元素
                    Object element = elements[i];
                    if (c.contains(element))
                        //如果集合中包含当前元素
                        //把添加元素添加到新数组中去，并把newlen加1
                        temp[newlen++] = element;
                }
                if (newlen != len) {
                    //如果新数组中元素数量不等于当前数组元素数量
                    //把新数组中的有效元素拷贝到数组中去
                    setArray(Arrays.copyOf(temp, newlen));
                    return true;
                }
            }
            return false;
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### addAllAbsent 方法

```java
//添加指定集合中，当前数组不存在的元素
public int addAllAbsent(Collection<? extends E> c) {
    	//集合转换为数组
        Object[] cs = c.toArray();
        if (cs.length == 0)
            //如果集合为空，直接返回
            return 0;
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取数组副本
            Object[] elements = getArray();
             //获取数组的长度
            int len = elements.length;
            //需要添加的数量，c中元素在当前数组中不存在的数量（差集？）
            int added = 0;
            // uniquify and compact elements in cs
             //for循环遍历
            for (int i = 0; i < cs.length; ++i) {
                //获取cs中对应的元素
                Object e = cs[i];
                if (indexOf(e, elements, 0, len) < 0 &&
                    indexOf(e, cs, 0, added) < 0)
                    //当前元素不存在当前数组，并且也不存在在cs被覆盖的前面区域中
                    //把当前元素移动到cs的前面区域
                    cs[added++] = e;
            }
            if (added > 0) {
                //如果cs中存在当前数组不存在的元素
                //创建一个新数组，长度为len+added
                //把当前数组的元素拷贝到新数组中
                Object[] newElements = Arrays.copyOf(elements, len + added);
                //把cs覆盖的前面区域的数据拷贝到新数组中
                System.arraycopy(cs, 0, newElements, len, added);
                //设置新的数组
                setArray(newElements);
            }
            //返回添加的数量(当前数组不包含传入集合中的元素数量)
            return added;
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### clear 方法

```java
//清除所有元素
public void clear() {
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //直接设置一个长度为0的新数组
            setArray(new Object[0]);
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### addAll 方法

```java
//添加指定集合中的所有元素
public boolean addAll(Collection<? extends E> c) {
    	//把集合转换为数组
        Object[] cs = (c.getClass() == CopyOnWriteArrayList.class) ?
            ((CopyOnWriteArrayList<?>)c).getArray() : c.toArray();
        if (cs.length == 0)
            //如果传入的集合为空，直接返回false
            return false;
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取数组副本
            Object[] elements = getArray();
            //获取数组的长度
            int len = elements.length;
            if (len == 0 && cs.getClass() == Object[].class)
                //如果当前数组长度为0 并且传入的集合元素为Object类
                //直接设置传入集合转换的数组为新的数组
                setArray(cs);
            else {
                //把当前数组扩容到len+cs.length，并把原数组元素拷贝到新的数组
                Object[] newElements = Arrays.copyOf(elements, len + cs.length);
                //把转换数组的元素拷贝到新数组尾部
                System.arraycopy(cs, 0, newElements, len, cs.length);
                //设置新的数组
                setArray(newElements);
            }
            return true;
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
//在指定位置添加指定集合中的所有元素
public boolean addAll(int index, Collection<? extends E> c) {
    	//把集合转换为数组
        Object[] cs = c.toArray();
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取数组副本
            Object[] elements = getArray();
            //获取数组的长度
            int len = elements.length;
            //参数检查
            if (index > len || index < 0)
                throw new IndexOutOfBoundsException("Index: "+index+
                                                    ", Size: "+len);
            //如果传入的集合为空，直接返回false
            if (cs.length == 0)
                return false;
            //计算移动数量
            //相当于把指定位置后面的所有元素向后移动传入集合的长度，把中间空出来存放集合元素
            int numMoved = len - index;
            Object[] newElements;
            if (numMoved == 0)
                //如果移动元素为0，表示在尾部添加。跟上面的方法一样
                //把当前数组扩容到len+cs.length，并把原数组元素拷贝到新的数组
                newElements = Arrays.copyOf(elements, len + cs.length);
            else {
                //在前面添加集合的情况
                //创建新的数组，长度为len + cs.length
                newElements = new Object[len + cs.length];
                //拷贝0到index的元素到新数组
                System.arraycopy(elements, 0, newElements, 0, index);
                //拷贝index到length的元素到新数组末尾
                System.arraycopy(elements, index,
                                 newElements, index + cs.length,
                                 numMoved);
            }
            //把传入集合的元素拷贝到新数组的中间，拷贝起始位置在index，长度为集合的长度
            System.arraycopy(cs, 0, newElements, index, cs.length);
            //设置新的数组
            setArray(newElements);
            return true;
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### forEach 方法

```java
//遍历元素执行操作
public void forEach(Consumer<? super E> action) {
        if (action == null) throw new NullPointerException();
        Object[] elements = getArray();
        int len = elements.length;
    	//for循环遍历元素。并对遍历元素执行action操作
        for (int i = 0; i < len; ++i) {
            @SuppressWarnings("unchecked") E e = (E) elements[i];
            action.accept(e);
        }
    }
```

#### removeIf 方法

```java
//移除符合条件的元素
public boolean removeIf(Predicate<? super E> filter) {
    	//参数校验
        if (filter == null) throw new NullPointerException();
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取数组副本
            Object[] elements = getArray();
            //获取数组的长度
            int len = elements.length;
            if (len != 0) {
                //如果当前数组存在元素
                //记录新数组元素的个数
                int newlen = 0;
                //创建一个新的元素
                Object[] temp = new Object[len];
                //for循环遍历
                for (int i = 0; i < len; ++i) {
                    @SuppressWarnings("unchecked") E e = (E) elements[i];
                    if (!filter.test(e))
                        //如果当前遍历元素不符合条件
                        //把当前元素添加到新数组
                        //移除相当于把不符合条件的留下来
                        //新数组元素数量加1
                        temp[newlen++] = e;
                }
                if (newlen != len) {
                    //如果新数组元素数量不等于旧数组元素数量
                    //把新数组的元素数量拷贝出来，并设置新的数组
                    setArray(Arrays.copyOf(temp, newlen));
                    return true;
                }
            }
            return false;
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### replaceAll 方法

```java
//替换所有元素，对每个元素进行操作
public void replaceAll(UnaryOperator<E> operator) {
    	//参数检查
        if (operator == null) throw new NullPointerException();
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取数组副本
            Object[] elements = getArray();
            //获取数组的长度
            int len = elements.length;
            //创建新的数组并拷贝数组所有元素
            Object[] newElements = Arrays.copyOf(elements, len);
            //遍历数组
            for (int i = 0; i < len; ++i) {
                //获取新数组元素
                @SuppressWarnings("unchecked") E e = (E) elements[i];
                //使用传入的operator操作当前遍历元素
                newElements[i] = operator.apply(e);
            }
            //设置新数组
            setArray(newElements);
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

#### sort 方法

```java
//根据传入的Comparator对象排序
public void sort(Comparator<? super E> c) {
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取数组副本
            Object[] elements = getArray();
             //创建新的数组并拷贝数组所有元素
            Object[] newElements = Arrays.copyOf(elements, elements.length);
            //强转新的数组为泛型对象
            @SuppressWarnings("unchecked") E[] es = (E[])newElements;
            //调用Arrays.sort对象对新数组排序
            Arrays.sort(es, c);
            //设置新数组
            setArray(newElements);
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```

####  toString 方法

```java
public String toString() {
    	//调用Arrays.toString方法
        return Arrays.toString(getArray());
    }
```

#### equals 方法

```java
public boolean equals(Object o) {
        if (o == this)
            //如果引用相等，直接返回true
            return true;
        if (!(o instanceof List))
            //如果不是List类型，返回false
            return false;

        List<?> list = (List<?>)(o);
    	//获取传入对象的迭代器
        Iterator<?> it = list.iterator();
        Object[] elements = getArray();
        int len = elements.length;
    	//for循环遍历
        for (int i = 0; i < len; ++i)
            //使用eq方法比较两个遍历对象
            //这里也比较了两个集合的长度
            if (!it.hasNext() || !eq(elements[i], it.next()))
                return false;
        if (it.hasNext())
            //传入的集合还有元素，表示长度不一样
            return false;
        return true;
    }
```

#### hashCode 方法

```java
public int hashCode() {
        int hashCode = 1;
        Object[] elements = getArray();
        int len = elements.length;
    	//循环获取每个元素的hashcode
    	//s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
        for (int i = 0; i < len; ++i) {
            Object obj = elements[i];
            //注意这里出现了31，优质质数，String的hashcode也使用的它
            hashCode = 31*hashCode + (obj==null ? 0 : obj.hashCode());
        }
        return hashCode;
    }
```

#### subList 方法

```java
//获取起始位置到结束位置的子集
public List<E> subList(int fromIndex, int toIndex) {
    	//上锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取数组副本
            Object[] elements = getArray();
             //获取数组的长度
            int len = elements.length;
            //参数检查
            if (fromIndex < 0 || toIndex > len || fromIndex > toIndex)
                throw new IndexOutOfBoundsException();
            //通过COWSubList来创建子集
            return new COWSubList<E>(this, fromIndex, toIndex);
        } finally {
            //释放锁资源
            lock.unlock();
        }
    }
```



