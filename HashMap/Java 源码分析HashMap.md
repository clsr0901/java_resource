### 1. 类的定义

```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable 
```

从上面可以看出

* HashMap是一个泛型类，键值对都是泛型
* HashMap继承了AbstractMap类
* HashMap实现了Map接口
* HashMap实现了Cloneable接口，表示HashMap支持克隆
* HashMap实现了Serializable接口，表示HashMap支持序列化

### 2.字段属性

```java
    //序列化版本号号
    private static final long serialVersionUID = 362498820763181265L;
    //默认初始化容量16，2的4次方    
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
    //最大容量2的30次方
    static final int MAXIMUM_CAPACITY = 1 << 30;
    //默认的负载因子为0.75，这个是用来影响扩容的
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    //元素数量临界值，大于8链表转换为红黑树
    static final int TREEIFY_THRESHOLD = 8;
    //元素数量临界值，小于6红黑树转换为链表
    static final int UNTREEIFY_THRESHOLD = 6;
    //转换为树的最小容量
    static final int MIN_TREEIFY_CAPACITY = 64;
    //table，table的大小必须是2的幂次方，为什么必须是2的幂次方，下面会详细介绍
    transient Node<K,V>[] table;
    //存储所有键值对的Set集合
    transient Set<Map.Entry<K,V>> entrySet;
    //元素数量，实际存储的键值对个数
    transient int size;
    //修改次数统计，防止遍历时被其他线程修改
    transient int modCount;
    //需要扩容的临界值 capacity * load factor
    int threshold;
    //负载因子，代表了table的填充度有多少，默认0.75
    final float loadFactor;
	//从父类继承，存储所有key的Set集合
	transient Set<K>        keySet;
	//从父类继承，存储所有value的Collection集合
    transient Collection<V> values;
```

从字段属性可以看出

* HashMap是数组加链表或者是数组加红黑树的数据结构
* 链表跟红黑树转换中间有两个元素的缓冲
* HashMap默认的数组容量是16，数组容量都是2的幂次方
* 保存了扩容的临界值，达到临界值就扩容，默认的负载因子是0.75

### 3. 构造函数

```java
    //接收初始化容量和负载因子
    public HashMap(int initialCapacity, float loadFactor) {
        //验证传入的初始化容量的合法性
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        //初始化容量大于默认最大容量，把初始化容量设置为默认最大容量
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        //验证传入的负载因子的合法性
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        //设置负载因子为传入的负载因子
        this.loadFactor = loadFactor;
        //设置扩容临界值为调整初始化容量后的容量
        //容量必须是2的幂次方，会调整到离传入的初始化容量最近的2的幂次方的数
        //调整初始化详见下面方法介绍
        this.threshold = tableSizeFor(initialCapacity);
    }
    //接收初始化容量
    public HashMap(int initialCapacity) {
        //调用接收初始化容量和负载因子的构造函数
        //使用默认的负载因子0.75
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
    //空构造函数
    public HashMap() {
        //设置负载因子为默认的负载因子，其他使用默认值
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
    //接收一个map对象
    public HashMap(Map<? extends K, ? extends V> m) {
        //设置负载因子为默认负载因子0.75
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        //添加map对象中的元素，详见下面方法介绍
        putMapEntries(m, false);
    }
```

从构造函数可以看出

* HashMap可以接收一个容量和负载因子初始化
* HashMap可以接收容量初始化
* HashMap可以有默认空的构造函数
* HashMap可以接收一个Map对象初始化
* HashMap初始化时，不传入负载因子，则使用默认的0.75作为负载因子
* HashMap会对传入的初始化容量进行调整，让其满足2的幂次方

### 4. 方法

#### hash 方法

```java
//获取key对应的hash值
static final int hash(Object key) {
        //h为key的hashCode
        int h;
        //如果传入的key为空，返回0
        //(h = key.hashCode()) ^ (h >>> 16)表示于十六位不变，低十六位和高十六位按位异或
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

  key 的哈希值通过它自身**hashCode**的高十六位与低十六位进行异或得到。这么做得原因是因为，由于哈希表的大小固定为 2 的幂次方，那么某个 key 的 hashCode 值大于 table.length，其高位就不会参与到 hash 的计算（对于某个 key 其所在的table的位置的计算为 `hash & (table.length - 1)`）。因此通过`hashCode()`的高16位异或低16位实现的：`(h = key.hashCode()) ^ (h >>> 16)`，主要是从速度、功效、质量来考虑的，保证了高位 Bits 也能参与到 Hash 的计算。 

#### tableSizeFor 方法

```java
//设置容量
static final int tableSizeFor(int cap) {
        //以65为例
        //容量减1，如果是2的幂次方-1后所有位都为1
        int n = cap - 1;//n=64
        //无符号右移1位后再使用位或运算
        n |= n >>> 1;//n | (n >>> 1) => 1000000 | 0100000 => n=1100000
        n |= n >>> 2;//n | (n >>> 2) => 1100000 | 0011000 => n=1111000
        n |= n >>> 4;//n | (n >>> 4) => 1111000 | 0000111 => n=1111111
        n |= n >>> 8;//n | (n >>> 8) => 1111111 | 0000000 => n=1111111
        n |= n >>> 16;//n | (n >>> 8) => 1111111 | 0000000 => n=1111111
        //从上面可以看出，通过上面计算后会把第一位为1开始的后面所有位都变为1，即n会变为cap最近的2的幂次方减1的数
        //如果n超过Integer最大值，把容量赋值为1，
        //如果n小于默认容量，把容量赋值为默认容量16
        //如果n大于默认容量16，把n加1变为2的幂次方作为当前量
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

从上面可以看出，HashMap的容量值永远都会是2的幂次方，为啥是2的幂次方，下面会讲

#### putMapEntries 方法

```java
//添加map集合元素，evict为false表示初始化添加，true表示其他情况
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        //获取传入map的大小
        int s = m.size();
        //如果map的大小大于0
        if (s > 0) {
            if (table == null) { // pre-size
                //如果table为空的话，重新计算threshold
                //计算能存方s个元素的容量
                float ft = ((float)s / loadFactor) + 1.0F;
                //如果计算的容量小于默认最大容量容量，使用计算的容量
                //如果计算的容量大于等于最大默认容量，使用最大默认容量
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                //如果计算的容量大于临界值则重新计算table的容量
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
            else if (s > threshold)
                //如果map的大小大于当前临界值，扩容
                resize();
            //for循环遍历map
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                //获取当前遍历的map元素的key
                K key = e.getKey();
                //获取当前遍历的map元素的value
                V value = e.getValue();
                //添加元素
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
```

#### size 方法

```java
//获取元素数量，即键值对个数
public int size() {
        //直接返回保存的元素数量
        return size;
    }
```

#### isEmpty 方法

```java
public boolean isEmpty() {
        //如果键值对数量等于0，表示HashMap为空
        return size == 0;
    }
```

#### get 方法

```java
//获取传入对象对应的值
public V get(Object key) {
        Node<K,V> e;
        //通过getNode来获取节点，如果节点为空，返回空。不为空，返回对应的值
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
```

#### getOrDefault 方法

```java
//获取对应的value，不存在返回默认值
public V getOrDefault(Object key, V defaultValue) {
        Node<K,V> e;
    	//通过调用getNode去获取value值
        return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
    }
```

#### getNode 方法

```java
//获取对应的节点
final Node<K,V> getNode(int hash, Object key) {
        //tab table的副本
        //first hash对应链表的第一个元素
        //e 表示后续遍历的元素
        //n table的长度
        //k 表示对应元素的key
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //如果table不为null，table长度大于0，hash对应的链表第一个元素不为空
        //(n-1) & hash表示hash在table中对应的插槽，会有对个hash对应同一个插槽，同一个插槽的数据根据数量可以是链表，也可以是红黑树，
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            //从第一个元素开始比较，先比较哈希值再获取对应元素的key比较
            //如果第一个元素匹配就返回第一个元素
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            //获取第一个元素的下一个元素，并判读是否为空，为空直接返回null
            if ((e = first.next) != null) {
                //第二个元素开始就会判断是红黑树还是链表
                if (first instanceof TreeNode)
                    //红黑树的情况，使用getTreeNode查找
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                //链表的情况，使用while遍历查找
                do {
                    //比对hash和key，比对成功就返回，不成功继续下一个
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        //没找到，返回空
        return null;
    }
```

从上面方法可以看出

* `(table.length - 1) & hash`可以查找到对应key在table的位置
* table可以获取第一个node节点，既可以是红黑树的根节点也可以是链表的头节点
* 判断是红黑树还是链表从第二个node节点开始
* 查找元素比对hash和key是否都相等
* <font color='red'>`(n-1) & hash`</font> 需要特别注意，这个公式可以快速查找对应的插槽，这也是table的长度为什么必须是2的幂次方的原因
  * n 是2的幂次方 n-1 的每一位都是1，`()(n-1) & hash) == (hash % n)`.取余运算的性能远远低于位运算

#### containsKey 方法

```java
//是否包含当前key
public boolean containsKey(Object key) {
        //使用getNode来查找
        return getNode(hash(key), key) != null;
    }
```

#### put 方法

```java
//存放元素 
public V put(K key, V value) {
        //使用putVal来存放元素
        return putVal(hash(key), key, value, false, true);
    }
```

#### putVal 方法

```java
//存放元素
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        //tab table的副本
        //p table对应hash插槽的节点
        //n table的长度
        //i 表示当前hash在table中的下标
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            //如果table为空或者table长度为0，调用resize初始化table的大小
            n = (tab = resize()).length;
        //通过(n - 1) & hash获取hash对应的下标，p为当前插槽第一个节点
        if ((p = tab[i = (n - 1) & hash]) == null)
            //如果table中hash对应的插槽没有数据，创建新的节点加入插槽
            tab[i] = newNode(hash, key, value, null);
        else {
            //table中hash对应插槽有数据
            //e 保存遍历中对应的节点的下一个节点
            //k 保存遍历中对应节点的key
            Node<K,V> e; K k;
            //比对第一个节点的hash和key
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                //第一个节点比对成功，把该节点用e保存
                e = p;
            else if (p instanceof TreeNode)
                //如果第一个几点是红黑树，使用putTreeVal添加
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //如果为链表，循环遍历
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        //如果遍历到尾节点，则创建一个新的节点添加到尾节点后面
                        p.next = newNode(hash, key, value, null)
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            //添加节点后，如果插槽元素数量大于等7，则把当前插槽从链表转换为转换为红黑树
                            treeifyBin(tab, hash);
                        //达到末尾，中断循环
                        break;
                    }
                    //如果找到对应的节点，中断循环
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    //把p指向当前节点的下一个节点，继续下次循环
                    p = e;
                }
            }
            //e为空表示在尾节点添加了新节点
            if (e != null) { // existing mapping for key
                   //如果e不等于空，保存旧值
                V oldValue = e.value;
                //onlyIfAbsent 如果为true不改变以前的旧值
                if (!onlyIfAbsent || oldValue == null)
                    //为对应节点设置新值
                    e.value = value;
                // Callbacks to allow LinkedHashMap post-actions
                //钩子，HashMap此处没有任何实现
                afterNodeAccess(e);
                //修改，返回旧值
                return oldValue;
            }
        }
        //修改统计加1
        ++modCount;
        //元素数量加1
        if (++size > threshold)
            //如果元素数量超过临界值进行扩容
            resize();
        // Callbacks to allow LinkedHashMap post-actions
        //钩子，HashMap此处没有任何实现
        afterNodeInsertion(evict);
        //添加的话返回空
        return null;
    }
```

#### putAll 方法

```java
//添加集合的所有元素
public void putAll(Map<? extends K, ? extends V> m) {
    	//通过调用putMapEntries方法实现
        putMapEntries(m, true);
    }
```

#### resize 方法

```java
//扩容
final Node<K,V>[] resize() {
        //拷贝一份table副本
        Node<K,V>[] oldTab = table;
        //拷贝一份table容量副本
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        //拷贝一份threshold副本
        int oldThr = threshold;
        //初始化新的容量和临界值
        int newCap, newThr = 0;
        //旧的容量大于0的情况
        if (oldCap > 0) {

            if (oldCap >= MAXIMUM_CAPACITY) {
                //如果以前的容量大于最大容量，设置临界值为Integer.MAX_VALUE，然后返回table副本
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //设置新的容量为旧容量的两倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                //如果新容量小于最大容量且旧容量大于默认初始化容量
                //新的临界值设置为旧临界值的两倍
                newThr = oldThr << 1; // double threshold
        }
        //旧的临界值大于0的情况
        else if (oldThr > 0) 
            //新的容量用旧临界值替代            
            newCap = oldThr;
        //未初始化的情况，容量和临界值都为0的情况
        else {           
            //新的容量设置为默认初始化容量16
            newCap = DEFAULT_INITIAL_CAPACITY;
            //新的负载因子等0.75*16=12
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            //新的临界值等于0
            //计算新的临界值为新容量*负载因子
            float ft = (float)newCap * loadFactor;
            //如果新的容量小于最大容量并且计算得到的新的临界值小于最大容量的话，把计算得到的新的临界值赋值给新的临界值。否则，新的临界值大小为Integer.MAX_VALUE
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        //把计算的新的临界值赋值给当前临界值
        threshold = newThr;
        //创建一个新容量大小的Node数组并赋值给table
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        //把table指向新的table，后续移动元素操作副本oldTab
        table = newTab;
        //旧的table不等于空的情况下
        if (oldTab != null) {
            //循环遍历旧的table
            for (int j = 0; j < oldCap; ++j) {
                //e 存储当前遍历得到的插槽的第一个节点
                Node<K,V> e;
                //当前节点不等于null
                if ((e = oldTab[j]) != null) {
                    //把旧的插槽置为null

                    oldTab[j] = null;
                    //如果当前节点没有下一个节点
                    if (e.next == null)
                        //直接把当前节点存储到新的table对应的插槽位置
                        newTab[e.hash & (newCap - 1)] = e;
                    //当前插槽存储的红黑树的情况
                    else if (e instanceof TreeNode)
                        //使用split方法来移动所有节点到新的插槽
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //当前节点存储链表的情况
                    else { // preserve order 保留顺序
                        //不改变位置的头节点和尾节点
                        Node<K,V> loHead = null, loTail = null;
                        //改变位置的头节点和尾节点
                        Node<K,V> hiHead = null, hiTail = null;
                        //存储循环的下一个节点
                        Node<K,V> next;
                        do {
                            //保存下一个节点
                            next = e.next;
                            //详细说明，假定扩容是把容量扩大两倍，因此hash值要么在原位置，要么在原位置oldCap
                            //举例1，hash值为10101010，原容量为16
                            // 16 - 1 => 1111
                            //原位置 10101010 & 1111 => 1010 => 10
                            //容量扩大一倍变为32
                            //32 - 1 =>  11111
                            //新位置 10101010 & 11111=> 1010 => 10
                            //可以看出没有变化，如果hash值11111111
                            //同样可以算出原位置为15 新位置为31 
                            //增加了16，刚好为oldCap
                            //上面可以得出扩容后位置可以不变，或者增加oldCap
                            //需要注意的是，同一个插槽中的不同元素可能存在需要改变位置，也可能存在不需要改变位置，同一个插槽中的元素，只能表示hash最低的几位相同，超过oldCap-1的位数不一样，扩容后影响位置的恰好是从右往左数的高位决定                           
                            //(e.hash & oldCap) == 0 表示e.hash & (oldCap - 1)的高一位是0，表示扩容后不需要改变位置
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    //如果尾节点为null，表示没有元素，把当前元素置为头节点
                                    loHead = e;
                                else
                                    //尾节点不为空，把当前元素置为尾节点的下一个节点
                                    loTail.next = e;
                                //尾节点指向当前节点
                                loTail = e;
                            }
                            //位置变为原位置+oldCap

                            else {
                                if (hiTail == null)
                                    //如果尾节点为null，表示没有元素，把当前元素置为头节点
                                    hiHead = e;
                                else
                                //尾节点不为空，把当前元素置为尾节点的下一个节点
                                    hiTail.next = e;
                                //尾节点指向当前节点
                                hiTail = e;
                            }
                         //把当前节点指向下一个节点，如果不为空则继续循环
                        } while ((e = next) != null);
                        //不需要改变位置
                        if (loTail != null) {
                            //如果尾节点不为空，把尾节点下一个节点引用置为null
                            loTail.next = null;
                            //当前插槽存入头节点
                            newTab[j] = loHead;
                        }
                        //需要改变位置
                        if (hiTail != null) {
                            //如果尾节点不为空，把尾节点下一个节点引用置为null
                            hiTail.next = null;
                            //把当前插槽位置移动oldCap的插槽存入头节点
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        //返回新的table
        return newTab;
    }
```

#### treeifyBin 方法

```java
//链表转换为红黑树
final void treeifyBin(Node<K,V>[] tab, int hash) {
        //n tab的长度
        //index tab中当前hash对应的插槽下标
        //e tab中对应当前hash的插槽的第一个节点
        int n, index; Node<K,V> e;
        //如果tab为空，或者tab的长度小于默认树最小容量， 进行扩容
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            //如果e不为空
            //hd 存储头节点
            //tl 存储尾节点
            TreeNode<K,V> hd = null, tl = null;
            //进行循环遍历链表
            do {
                //replacementTreeNode根据传入的Node创建一个TreeNode
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    //如果尾节点为null，表示这是第一个节点
                    hd = p;
                else {
                    //把当前节点的前一个节点的引用指向上一个尾节点
                    p.prev = tl;
                    //把上一个尾节点的后一个节点的引用指向当前节点
                    tl.next = p;
                }
                //把当前节点置为尾节点
                tl = p;
                //把e设置为下一个节点
            } while ((e = e.next) != null);
            //如果头节点部位空，把上面合成的TreeNode链表通过treeify方法转换为红黑树
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
```

#### remove 方法

```java
//根据key移除元素
public V remove(Object key) {
        Node<K,V> e;
    	//调用removeNode实现，如果元素不存在返回null，存在返回元素的value
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }

@Override
	//value 匹配才移除
    public boolean remove(Object key, Object value) {
        //通过removeNode方法实现
        return removeNode(hash(key), key, value, true, true) != null;
    }
```

#### removeNode 方法

```java
final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
    	//tab table的副本
    	//p 当前遍历的元素
    	//n table的长度
    	//index hash对应在table的插槽位置
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            //如果table不为null，table长度大于0，当前插槽有元素
             //e 当前遍历元素
            //k 当前遍历元素的key
            //node 存储查找到的元素，如果最后node为null表示该元素不存在
            Node<K,V> node = null, e; K k; V v;
            //处理第一个元素
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                //如果当前元素的hash等于传入元素的hash 并且 当前元素的key等于传入的可以
                //node 指向当前元素
                node = p;
            //处理第二个元素
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    //如果是红黑树结构使用getTreeNode方法查找
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    //链表结构，循环遍历查找
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            //查找到退出循环
                            node = e;
                            break;
                        }
                        //p指向当前遍历元素，如果查找到了，p表示查找到的元素的上一个元素
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            //matchValue 表示只在value相等的情况下删除
            //movable 表示红黑树树结构中删除时是否要移动其它元素，false表示不移动
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    //如果是红黑树结构，使用removeTreeNode方法删除元素
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    //链表结构
                    //node==p表示插槽第一个元素符合规则
                    //把node的下一个节点置为插槽的第一个元素就可以了
                    tab[index] = node.next;
                else
                    //链表结构
                    //p表示符合元素的上一个元素
                    //直接跳过node元素就可以了
                    p.next = node.next;
                //修改统计加1
                ++modCount;
                //元素数量减1
                --size;
                //钩子，此处没有实现
                afterNodeRemoval(node);
                //移除成功返回node
                return node;
            }
        }
    	//元素不存在，返回null
        return null;
    }
```

#### clear 方法

```java
//清空所有元素
public void clear() {
    	//table副本
        Node<K,V>[] tab;
    	//修改统计加1
        modCount++;
        if ((tab = table) != null && size > 0) {
            //如果table不为null并且元素数量大于0
            //元素数量置为0
            size = 0;
            for (int i = 0; i < tab.length; ++i)
                //遍历table，把所有插槽置为null
                tab[i] = null;
        }
    }
```

#### containsValue 方法

```java
//是否存在当前值
public boolean containsValue(Object value) {
    	//tab table副本
    	//v 遍历当前元素的值
        Node<K,V>[] tab; V v;
        if ((tab = table) != null && size > 0) {
            //如果table不为null 并且 元素数量大于0
            for (int i = 0; i < tab.length; ++i) {
                //循环遍历table
                for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                    //循环遍历插槽中的元素
                    if ((v = e.value) == value ||
                        (value != null && value.equals(v)))
                        //如果比对成功返回true
                        return true;
                }
            }
        }
    	//没找到返回false
        return false;
    }
```

#### keySet 方法

```java
//获取HashMap所有key的Set集合
public Set<K> keySet() {
    	//keySet是从父类AbstractMap继承
        Set<K> ks = keySet;
        if (ks == null) {
            //这句很关键，KeySet是HashMap的内部类，所以KeySet对象能够拿到HashMap对象里面所有的数据
            ks = new KeySet();
            keySet = ks;
        }
        return ks;
    }
```

#### values 方法

```java
//获取HashMap所有value的Collection集合 
public Collection<V> values() {
    	//values是从父类AbstractMap继承
        Collection<V> vs = values;
        if (vs == null) {
            //这句很关键，Values是HashMap的内部类，所以Values对象能拿到HashMap对象里面所有的数据
            vs = new Values();
            values = vs;
        }
        return vs;
    }
```

#### entrySet 方法

```java
//获取HashMap所有键值对的Set集合
public Set<Map.Entry<K,V>> entrySet() {
        Set<Map.Entry<K,V>> es;
    	//entrySet = new EntrySet() 是关键点
    	//EntrySet是HashMap的内部类，所以EntrySet对象能拿到HashMap对象里面所有的数据
        return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
    }
```

#### putIfAbsent 方法

```java
@Override
	//存入数据，如果储在不修改
    public V putIfAbsent(K key, V value) {
        //通过putVal方法实现
        return putVal(hash(key), key, value, true, true);
    }
```

#### replace 方法

```java
 @Override
	//修改节点的值，需要比对旧值
    public boolean replace(K key, V oldValue, V newValue) {
        //e key对应的节点
        //v 节点的值
        Node<K,V> e; V v;
        //通过getNode获取对应的节点
        if ((e = getNode(hash(key), key)) != null &&
            ((v = e.value) == oldValue || (v != null && v.equals(oldValue)))) {
            //如果节点不为null 并且 节点的值等于oldVale
            //把节点的值设置为新值
            e.value = newValue;
            //钩子，此处未实现
            afterNodeAccess(e);
            //成功返回true
            return true;
        }
        //不存在或者值不相等返回false
        return false;
    }

    @Override
	//修改节点的值
    public V replace(K key, V value) {
        //e key对应的节点
        Node<K,V> e;
        //通过getNode获取对应的节点
        if ((e = getNode(hash(key), key)) != null) {
            //保存旧值
            V oldValue = e.value;
            //设置新值
            e.value = value;
            //钩子，此处未实现
            afterNodeAccess(e);
            //返回旧值
            return oldValue;
        }
        //没有对应的节点，返回null
        return null;
    }
```

#### loadFactor 方法

```java
final float loadFactor() { return loadFactor; }
```

#### capacity 方法

```java
 final int capacity() {
        return (table != null) ? table.length :
            (threshold > 0) ? threshold :
            DEFAULT_INITIAL_CAPACITY;
    }
```

