```
getEntryUsingComparator
```

### 1. 类的定义

```java
public class TreeMap<K,V>
    extends AbstractMap<K,V>
    implements NavigableMap<K,V>, Cloneable, java.io.Serializable
```

从类的定义中可以看出

* TreeMap是一个泛型类
* TreeMap继承自AbstractMap
* TreeMap实现NavigableMap接口，表示TreeMap具有方向性，支持导航
* TreeMap实现Cloneable接口，表示TreeMap支持克隆
* TreeMap实现java.io.Serializable接口，表示TreeMap支持序列化

### 2.字段属性

```java
//比较器，TreeMap的顺序由比较器决定，如果比较器为空，顺序由key自带的比较器决定
private final Comparator<? super K> comparator;
//根节点，Entry是一个红黑树结构
private transient Entry<K,V> root;
//节点的数量
private transient int size = 0;
//修改统计
private transient int modCount = 0;
//缓存键值对集合
private transient EntrySet entrySet;
//缓存key的Set集合
private transient KeySet<K> navigableKeySet;
private transient NavigableMap<K,V> descendingMap;
```

从字段属性中可以看出

* TreeMap是一个红黑树结构
* TreeMap保存了根节点root

### 3.构造方法

```java
//默认的构造方法
public TreeMap() {
    	//比较器默认为null
        comparator = null;
    }
//传入一个比价器对象
public TreeMap(Comparator<? super K> comparator) {
    	//设置比较器
        this.comparator = comparator;
    }
//传入一个Map对象
public TreeMap(Map<? extends K, ? extends V> m) {
    	//比较器设为null
        comparator = null;
    	//添加Map对象的数据
        putAll(m);
    }
//传入一个SortedMap对象
public TreeMap(SortedMap<K, ? extends V> m) {
    	//把SortedMap的比较器赋值给当前的比较器
        comparator = m.comparator();
        try {
            //添加SortedMap对象的数据
            buildFromSorted(m.size(), m.entrySet().iterator(), null, null);
        } catch (java.io.IOException cannotHappen) {
        } catch (ClassNotFoundException cannotHappen) {
        }
    }
```

从构造方法中可以看出

* TreeMap默认比较器为null，实质上使用的是Key自带的比较器，如果默认比较器为空，Key的对象必须实现Comparable接口
* TreeMap可以指定比较器进行初始化
* TreeMap可以接收Map对象来初始化
* TreeMap可以接收SortedMap对象来初始化

### 4.方法

#### size  方法

```java
//获取元素数量
public int size() {
    	//直接返回缓存的size
        return size;
    }
```

#### containsKey 方法

```java
//是否存在key对应的节点
public boolean containsKey(Object key) {
    	//通过getEntry来获取key对应的节点
        return getEntry(key) != null;
    }
```

#### get 方法

```java
//根据传入的key查找值
public V get(Object key) {
    	//通过getEntry来查找对应key的节点
        Entry<K,V> p = getEntry(key);
    	//如果节点不存在返回null，存在返回节点的value
        return (p==null ? null : p.value);
    }
```

#### getEntry 方法

```java
//根据传入的key获取对应的节点
final Entry<K,V> getEntry(Object key) {
        if (comparator != null)
            //如果存在比较器，调用getEntryUsingComparator方法来查找节点
            return getEntryUsingComparator(key);
    	//检查key的合法性
        if (key == null)
            throw new NullPointerException();
    	//key的对象必须实现Comparable接口
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
    	//获取根节点的副本
        Entry<K,V> p = root;
    	//通过while循环查找
        while (p != null) {
            //key进行比较
            int cmp = k.compareTo(p.key);
            if (cmp < 0)
                //如果传入的key比当前节点的key小，往左边查找
                //把当前节点指向左子节点
                p = p.left;
            else if (cmp > 0)
                //如果传入的key比当前节点的key大，往右边查找
                //把当前节点指向右子节点
                p = p.right;
            else
                //查找成功，返回当前节点
                return p;
        }
    	//没找到，返回null
        return null;
    }
```

#### getEntryUsingComparator 方法

```java
//使用TreeMap自带的比较器查找节点
final Entry<K,V> getEntryUsingComparator(Object key) {
        @SuppressWarnings("unchecked")
            K k = (K) key;
    	//获取比较器的副本
        Comparator<? super K> cpr = comparator;
        if (cpr != null) {
            //如果比较器不为null
            //获取根节点的副本
            Entry<K,V> p = root;
            //使用while循环查找
            while (p != null) {
                //使用TreeMap自带的比较器进行比较
                int cmp = cpr.compare(k, p.key);
                if (cmp < 0)
                    //如果传入的key比当前节点的key小，往左边查找
                    //把当前节点指向左子节点
                    p = p.left;
                else if (cmp > 0)
                    //如果传入的key比当前节点的key大，往右边查找
                    //把当前节点指向右子节点
                    p = p.right;
                else
                    //查找成功，返回当前节点
                    return p;
            }
        }
    	//没找到，返回null
        return null;
    }
```

#### getCeilingEntry 方法

```java
//返回仅仅大于等于传入的key的节点
final Entry<K,V> getCeilingEntry(K key) {
    	//获取根节点副本
        Entry<K,V> p = root;
    	//while循环 p为null退出
        while (p != null) {
            //比较传入的key和当前遍历节点的key
            int cmp = compare(key, p.key);
            if (cmp < 0) {
                //如果传入的key小于当前遍历节点的key
                if (p.left != null)
                    //如果当前遍历节点的左子节点不为null，往左继续遍历
                    //把当前遍历节点指向左子节点
                    p = p.left;
                else
                    //比当前节点小，但是当前节点不存在左子节点，返回当前节点
                    return p;
            } else if (cmp > 0) {
                //如果传入的key大于当前遍历节点的key
                if (p.right != null) {
                    //如果当前遍历节点右子节点不为null，往右继续遍历
                    //把当前遍历节点指向右子节点
                    p = p.right;
                } else {
                    //如果当前遍历节点右子节点为null
                    //获取当前遍历节点的父节点
                    Entry<K,V> parent = p.parent;
                    //当前遍历节点的副本
                    Entry<K,V> ch = p;
                    //while循环遍历
                    //一直往上找，知道父节点是祖父节点的左子节点
                    while (parent != null && ch == parent.right) {
                        ch = parent;
                        parent = parent.parent;
                    }
                    //返回祖父节点
                    return parent;
                }
            } else
                //传入的key等于当前遍历节点的key，返回当前节点
                return p;
        }
        return null;
    }
```

#### getFloorEntry 方法

```java
//返回仅仅小于等于传入的key的节点
final Entry<K,V> getFloorEntry(K key) {
    	//获取根节点的副本
        Entry<K,V> p = root;
    	//while循环 p为null退出
        while (p != null) {
            //比较传入的key和当前遍历节点的key
            int cmp = compare(key, p.key);
            if (cmp > 0) {
                //如果传入的key大于当前遍历节点的key
                if (p.right != null)
                    //如果当前遍历的key存在右子节点，往右继续遍历
                    //把当前遍历节点指向右子节点
                    p = p.right;
                else
                    //如果当前遍历节点不存在右子节点，返回当前遍历节点
                    return p;
            } else if (cmp < 0) {
                //如果传入的key小于当前遍历节点的key
                if (p.left != null) {
                    //如果当前遍历节点的左子节点不为null，往左继续遍历
                    //当前遍历节点指向左子节点
                    p = p.left;
                } else {
                    //如果当前遍历节点左子节点为null
                    //获取当前遍历节点的父节点
                    Entry<K,V> parent = p.parent;
                     //当前遍历节点的副本
                    Entry<K,V> ch = p;
                    //while循环遍历
                    //一直往上找，直到父节点是祖父节点的右子节点
                    while (parent != null && ch == parent.left) {
                        ch = parent;
                        parent = parent.parent;
                    }
                    //返回祖父节点
                    return parent;
                }
            } else
                //如果传入的key等于当前遍历节点，返回当前遍历节点
                return p;

        }
        return null;
    }
```

#### getHigherEntry 方法

```java
//返回仅仅大于传入的key的节点
final Entry<K,V> getHigherEntry(K key) {
    	//获取根节点的副本
        Entry<K,V> p = root;
    	//while循环，p为null退出
        while (p != null) {
            //比较传入的key和当前遍历节点key
            int cmp = compare(key, p.key);
            if (cmp < 0) {
                //如果传入的key小于当前遍历节点的key
                if (p.left != null)
                    //如果当前遍历节点的左子节点不为null，继续往左遍历
                    //当前遍历节点指向左子节点
                    p = p.left;
                else
                    //如果当前遍历节点的左子节点为null
                    //返回当前节点
                    return p;
            } else {
                //如果传入的key大于或等于当前遍历节点的key
                if (p.right != null) {
                    //如果当前遍历节点的右子节点不为null，往右继续遍历
                    //当前遍历节点指向右子节点
                    p = p.right;
                } else {
                    //如果当前遍历节点的右子节点为null
                    //获取当前遍历节点的父节点
                    Entry<K,V> parent = p.parent;
                    //保存当前遍历节点的副本
                    Entry<K,V> ch = p;
                    //while循环遍历
                    //一直往上找，直到父节点是祖父节点的左子子节点
                    while (parent != null && ch == parent.right) {
                        ch = parent;
                        parent = parent.parent;
                    }
                    //返回祖父节点
                    return parent;
                }
            }
        }
        return null;
    }
```

#### getLowerEntry

```java
//返回仅仅小于传入的key的节点
final Entry<K,V> getLowerEntry(K key) {
    	//获取根节点的副本
        Entry<K,V> p = root;
   	    //while循环，p为null退出
        while (p != null) {
            //比较传入的key和当前遍历节点key
            int cmp = compare(key, p.key);
            if (cmp > 0) {
                //如果传入的key大于当前遍历节点的key
                if (p.right != null)
                    //如果当前遍历的key存在右子节点，往右继续遍历
                    //把当前遍历节点指向右子节点
                    p = p.right;
                else
                    //如果当前遍历节点不存在右子节点，返回当前遍历节点
                    return p;
            } else {
                //如果传入的key小于或等于当前遍历节点的key
                if (p.left != null) {
                    //如果当前遍历节点的左子节点不为null，往左继续遍历
                    //当前遍历节点指向左子节点
                    p = p.left;
                } else {
                     //获取当前遍历节点的父节点
                    Entry<K,V> parent = p.parent;
                    //当前遍历节点的副本
                    Entry<K,V> ch = p;
                    //while循环遍历
                    //一直往上找，直到父节点是祖父节点的右子节点
                    while (parent != null && ch == parent.left) {
                        ch = parent;
                        parent = parent.parent;
                    }
                    //返回祖父节点
                    return parent;
                }
            }
        }
        return null;
    }
```

#### containsValue 方法

```java
//是否存在对应的值
public boolean containsValue(Object value) {
    	//使用for循环查找
    	//getFirstEntry 获取最左子节点，即最小的节点
    	//successor 获取当前节点的仅比当前节点大的下一个节点
    	//for循环是根据key从小到大遍历
        for (Entry<K,V> e = getFirstEntry(); e != null; e = successor(e))
            //使用valEquals 比对值
            if (valEquals(value, e.value))
                //如果存在，返回true
                return true;
    	//不存在，返回false
        return false;
    }
```

#### getFirstEntry 方法

```java 
//获取第一个节点，也是key最小的节点，即最左子节点
final Entry<K,V> getFirstEntry() {
    	//获取根节点的副本
        Entry<K,V> p = root;
        if (p != null)
            //如果根节点不为null
            //从根节点开始遍历整棵树
            while (p.left != null)
                //把节点指向左子节点
                p = p.left;
    	//返回一直往左找到的节点
        return p;
    }
```

#### firstEntry 方法

```java
//深拷贝第一个节点
public Map.Entry<K,V> firstEntry() {
    	//getFirstEntry 获取第一个节点，也是key最小的节点，即最左子节点
    	//exportEntry 返回一个新的SimpleImmutableEntry对象
        return exportEntry(getFirstEntry());
    }
```

#### getLastEntry 方法

```java
//获取最右一个节点，也是key最大的节点，即最右子节点
final Entry<K,V> getLastEntry() {
    	//获取根节点的副本
        Entry<K,V> p = root;
        if (p != null)
            //如果根节点不为null
            //从根节点开始遍历整棵树
            while (p.right != null)
                //把节点指向右子节点
                p = p.right;
    	//返回一直往右找到的节点
        return p;
    }
```

####  lastEntry 方法

```java
//深拷贝最后一个节点
public Map.Entry<K,V> lastEntry() {
    	//getLastEntry 获取最右一个节点，也是key最大的节点，即最右子节点
    	//exportEntry 返回一个新的SimpleImmutableEntry对象
        return exportEntry(getLastEntry());
    }
```



#### successor 方法

```java
//查找key仅比当前节点大的节点
static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
        if (t == null)
            //传入节点为null，直接返回null
            return null;
    	//先找右子节点的最左子节点
        else if (t.right != null) {
            //如果右子节点不为null
            //保存右子节点的副本
            Entry<K,V> p = t.right;
            //使用while循环一直往左找
            while (p.left != null)
                p = p.left;
            //返回最后找到的节点
            return p;
        } else {
            //如果当前节点是右节点
            //while循环一直往上找，直到找到父类是左子节点，左边的始终小于右边
            Entry<K,V> p = t.parent;
            Entry<K,V> ch = t;
            while (p != null && ch == p.right) {
                ch = p;
                p = p.parent;
            }
            return p;
        }
    }
```

#### valEquals 方法

```java
//比较两个对象是否相等
static final boolean valEquals(Object o1, Object o2) {
        return (o1==null ? o2==null : o1.equals(o2));
    }
```

#### comparator 方法

```java
//获取比较器
public Comparator<? super K> comparator() {
        return comparator;
    }
```

#### firstKey 方法

```java
//获取第一个节点（最左子节点）的key
public K firstKey() {
    	//getFirstEntry 获取第一个节点
    	//key 获取节点的key
        return key(getFirstEntry());
    }
```

#### lastKey 方法

```java
//获取最后一个节点(最右子节点)的key
public K lastKey() {
    	//getLastEntry 获取最后一个节点
    	//key 获取节点的key
        return key(getLastEntry());
    }
```

#### key 方法

```java
//获取节点的key
static <K> K key(Entry<K,?> e) {
    	//节点为null，抛出异常
        if (e==null)
            throw new NoSuchElementException();
        return e.key;
    }
```

#### keyOrNull 方法

```java
//获取节点的key，节点为null返回null
static <K,V> K keyOrNull(TreeMap.Entry<K,V> e) {
        return (e == null) ? null : e.key;
    }
```

#### pollFirstEntry 方法

```java
//获取第一个元素并删除
public Map.Entry<K,V> pollFirstEntry() {
    	//getFirstEntry 获取第一个节点，也是key最小的节点，即最左子节点
        Entry<K,V> p = getFirstEntry();
    	//根据p生成一个新的Entry对象
        Map.Entry<K,V> result = exportEntry(p);
        if (p != null)
            //如果p不为null，删除p
            deleteEntry(p);
    	//返回新生成的对象
        return result;
    }
```

#### pollLastEntry 方法

```java
//获取最后一个元素并删除
public Map.Entry<K,V> pollLastEntry() {
    	//getLastEntry 获取最右一个节点，也是key最大的节点，即最右子节点
        Entry<K,V> p = getLastEntry();
    	//根据p生成一个新的Entry对象
        Map.Entry<K,V> result = exportEntry(p);
        if (p != null)
            //如果p不为null，删除p
            deleteEntry(p);
    	//返回新生成的对象
        return result;
    }
```

#### lowerEntry 方法

```java
//获取仅仅小于传入的key的节点，并包装成一个新的对象
public Map.Entry<K,V> lowerEntry(K key) {
    	//getLowerEntry 获取仅仅小于传入的key的节点
    	//exportEntry 包装成一个新的对象
        return exportEntry(getLowerEntry(key));
    }
```

####  lowerKey 方法

```java
//获取仅仅小于传入的key的节点的key
public K lowerKey(K key) {
    	//getLowerEntry 获取仅仅小于传入的key的节点
    	//keyOrNull 获取节点的key
        return keyOrNull(getLowerEntry(key));
    }
```

#### floorEntry 方法

```java
//返回仅仅小于等于传入的key的节点,并包装成一个新的对象
public Map.Entry<K,V> floorEntry(K key) {
    	//getFloorEntry 返回仅仅小于等于传入的key的节点
    	//exportEntry 包装成一个新的对象
        return exportEntry(getFloorEntry(key));
    }
```

#### floorKey 方法

```java
//返回仅仅小于等于传入的key的节点的key
public K floorKey(K key) {
    	//getFloorEntry 返回仅仅小于等于传入的key的节点
    	//keyOrNull 获取节点的key
        return keyOrNull(getFloorEntry(key));
    }
```

#### ceilingEntry 方法

```java
//返回仅仅大于等于传入的key的节点, 并包装成一个新的对象
public Map.Entry<K,V> ceilingEntry(K key) {
    	//getCeilingEntry 返回仅仅大于等于传入的key的节点
    	//exportEntry 包装成一个新的对象
        return exportEntry(getCeilingEntry(key));
    }
```

#### ceilingKey 方法

```java
//返回仅仅大于等于传入的key的节点的key
public K ceilingKey(K key) {
    	//getCeilingEntry 返回仅仅大于等于传入的key的节点
    	//keyOrNull 获取节点的key
        return keyOrNull(getCeilingEntry(key));
    }
```

#### higherEntry 方法

```java
//返回仅仅大于传入的key的节点, 并包装成一个新的对象
public Map.Entry<K,V> higherEntry(K key) {
    	//getHigherEntry 返回仅仅大于传入的key的节点
    	//exportEntry 包装成一个新的对象
        return exportEntry(getHigherEntry(key));
    }
```

#### higherKey 方法

```java
//返回仅仅大于传入的key的节点的key
public K higherKey(K key) {
    	//getHigherEntry 返回仅仅大于传入的key的节点
    	//keyOrNull 获取节点的key
        return keyOrNull(getHigherEntry(key));
    }
```

#### keySet 方法

```java
//获取所有key的Set集合
public Set<K> keySet() {
    	//调用navigableKeySet 方法获取
        return navigableKeySet();
    }
```

#### navigableKeySet 方法

```java
//返回key的Set集合
public NavigableSet<K> navigableKeySet() {
     	//先使用缓存
        KeySet<K> nks = navigableKeySet;
    	//如果缓存为空，重新创建KeySet对象
    	//注意，KeySet是TreeMap的内部类，可以获取TreeMap对象所有的属性
    	//KeySet是NavigableSet的子类，KeySet实现了NavigableSet接口
        return (nks != null) ? nks : (navigableKeySet = new KeySet<>(this));
    }
```

### descendingKeySet 方法

```java
//获取descendingMap
public NavigableSet<K> descendingKeySet() {
        return descendingMap().navigableKeySet();
    }
```

#### descendingMap 方法

```java
//返回descendingMap 
public NavigableMap<K, V> descendingMap() {
        NavigableMap<K, V> km = descendingMap;
    	//如果缓存为空，重新创建DescendingSubMap对象
    	//注意，DescendingSubMap是TreeMap的内部类，可以获取TreeMap对象所有的属性
        return (km != null) ? km :
            (descendingMap = new DescendingSubMap<>(this,
                                                    true, null, true,
                                                    true, null, true));
    }
```

#### values 方法

```java
//获取所有value的Colletion集合
public Collection<V> values() {
    	//使用缓存
        Collection<V> vs = values;
        if (vs == null) {
            //如果缓存为空，重新创建Values对象
    	   //注意，Values是TreeMap的内部类，可以获取TreeMap对象所有的属性
            vs = new Values();
            values = vs;
        }
    	//返回value的Colletion集合
        return vs;
    }
```

#### entrySet 方法

```java
//获取所有键值对的Entry对象
public Set<Map.Entry<K,V>> entrySet() {
    	//使用缓存
        EntrySet es = entrySet;
    	//如果缓存为空，重新创建EntrySet对象
    	//注意，EntrySet是TreeMap的内部类，可以获取TreeMap对象所有的属性
        return (es != null) ? es : (entrySet = new EntrySet());
    }
```

#### replace 方法

```java
@Override
	//修改节点的值，需要比对旧值
    public boolean replace(K key, V oldValue, V newValue) {
        //通过getEntry 获取节点
        Entry<K,V> p = getEntry(key);
        if (p!=null && Objects.equals(oldValue, p.value)) {
            //如果节点不为null，并且节点的值跟传入的oldValue相等
            //把新值赋值给节点
            p.value = newValue;
            //修改成功，返回true
            return true;
        }
        //节点不存在，返回false
        return false;
    }

@Override
	//修改节点的值，不需要比对旧值
    public V replace(K key, V value) {
        //通过getEntry 获取节点
        Entry<K,V> p = getEntry(key);
        if (p!=null) {
            //如果节点不为null
            //获取节点的旧值
            V oldValue = p.value;
            //把新值赋值给节点
            p.value = value;
            //返回旧值
            return oldValue;
        }
        //节点不存在，返回null
        return null;
    }
```

#### forEach 方法

```java
 @Override
    public void forEach(BiConsumer<? super K, ? super V> action) {
        //检查参数的合法性
        Objects.requireNonNull(action);
        //获取修改统计的副本
        int expectedModCount = modCount;
        //遍历节点，每个节点都调用BiConsumer方法
        for (Entry<K, V> e = getFirstEntry(); e != null; e = successor(e)) {
            action.accept(e.key, e.value);
			//如果遍历过程中，有其他线程修改，抛出异常
            if (expectedModCount != modCount) {
                throw new ConcurrentModificationException();
            }
        }
    }
```

#### replaceAll 方法

```java
 @Override
    public void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) {
        //检查参数的合法性
        Objects.requireNonNull(function);
        //获取修改统计的副本
        int expectedModCount = modCount;
		//遍历节点，每个节点都调用BiFunction方法		
        for (Entry<K, V> e = getFirstEntry(); e != null; e = successor(e)) {
            e.value = function.apply(e.key, e.value);
			//如果遍历过程中，有其他线程修改，抛出异常
            if (expectedModCount != modCount) {
                throw new ConcurrentModificationException();
            }
        }
    }
```

#### exportEntry 方法

```java
//把Entry对象包装成一个新的对象，深拷贝？
static <K,V> Map.Entry<K,V> exportEntry(TreeMap.Entry<K,V> e) {
        return (e == null) ? null :
            new AbstractMap.SimpleImmutableEntry<>(e);
    }
```

#### putAll 方法

```java
//添加Map对象的元素
public void putAll(Map<? extends K, ? extends V> map) {
    	//获取传入map对象的大小
        int mapSize = map.size();
        if (size==0 && mapSize!=0 && map instanceof SortedMap) {
            //如果map为SortedMap
            //获取map的比较器
            Comparator<?> c = ((SortedMap<?,?>)map).comparator();
            if (c == comparator || (c != null && c.equals(comparator))) {
                //如果map的比较器和当前的比较器相等
                //把修改统计加1
                ++modCount;
                try {
                    //使用buildFromSorted添加元素
                    buildFromSorted(mapSize, map.entrySet().iterator(),
                                    null, null);
                } catch (java.io.IOException cannotHappen) {
                } catch (ClassNotFoundException cannotHappen) {
                }
                return;
            }
        }
    	//如果只是普通的map对象，使用父类的方法添加元素
        super.putAll(map);
    }
//父类的putAll方法
public void putAll(Map<? extends K, ? extends V> m) {
    	//循环遍历Map元素，获取键值对，调用put方法添加
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet())
            //调用put方法添加元素
            put(e.getKey(), e.getValue());
    }
```

#### put 方法

```java
//添加键值对元素
public V put(K key, V value) {
    	//获取根节点副本
        Entry<K,V> t = root;
        if (t == null) {
            //如果根节点为null
            //类型检查和空检查，如果当前比较器为null，key的对象必须实现Comparable接口
            compare(key, key); // type (and possibly null) check
		   //根据key和value创建新的节点，并把创建的新节点作为根节点
            root = new Entry<>(key, value, null);
            //把元素数量置为1
            size = 1;
            //修改统计加1
            modCount++;
            //返回null
            return null;
        }
    //根节点不为null的情况
        int cmp;
    	//parent 插入节点的父节点
        Entry<K,V> parent;
        // split comparator and comparable paths
    	//获取当前的比较器
        Comparator<? super K> cpr = comparator;
        if (cpr != null) {
            //如果存在比较器
            //while循环遍历查找
            do {
                //把parent指向当前节点
                parent = t;
                //表当前节点的key和传入的key
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)
                    //如果传入的key小于当前节点的key，往左查找
                    //把当前节点指向左子节点
                    t = t.left;
                else if (cmp > 0)
                    //如果传入的key大于当前节点的key，往右查找
                    //把当前节点指向右子节点
                    t = t.right;
                else
                    //如果查找到了，把当前节点的值替换为传入的值
                    return t.setValue(value);
            } while (t != null);
        }
    	//不存在比较器，使用key的比较器
        else {
            if (key == null)
                //如果传入key为空，抛出异常
                throw new NullPointerException();
            //强转key
            @SuppressWarnings("unchecked")
                Comparable<? super K> k = (Comparable<? super K>) key;
            //while循环遍历查找
            do {
                //把parent指向当前节点
                parent = t;
                //使用key自带的比较器进行比较
                cmp = k.compareTo(t.key);
                if (cmp < 0)
                    //如果传入的key小于当前节点的key，往左查找
                    //把当前节点指向左子节点
                    t = t.left;
                else if (cmp > 0)
                    //如果传入的key大于当前节点的key，往右查找
                    //把当前节点指向右子节点
                    t = t.right;
                else
                    //如果查找到了，把当前节点的值替换为传入的值
                    return t.setValue(value);
            } while (t != null);
        }
    	//把最后遍历的节点作为父节点，创建新的节点
        Entry<K,V> e = new Entry<>(key, value, parent);
        if (cmp < 0)
            //如果插入的key比parent的key小，新的节点作为左子节点
            parent.left = e;
        else
            //如果插入的key比parent的key大，新的节点作为右子节点
            parent.right = e;
    	//插入过后进行平衡操作
        fixAfterInsertion(e);
    	//元素数量加1
        size++;
    	//修改统计加1
        modCount++;
    	//返回null
        return null;
    }
```

#### fixAfterInsertion 方法

```java
//插入节点后，平衡红黑树
private void fixAfterInsertion(Entry<K,V> x) {
    	//把插入的节点默认为红节点
        x.color = RED;
    	//while循环操作
    	//如果当前节点不为空，当前节点不是根节点，当前节点的父节点是红节点就一直循环
    	//直到父节点是黑节点或者达到根节点，才退出循环
        while (x != null && x != root && x.parent.color == RED) {
            //****************如果父节点是祖父节点的左子节点*****************
            if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
                //获取祖父节点的右节点，也就是右叔叔节点
                Entry<K,V> y = rightOf(parentOf(parentOf(x)));
                if (colorOf(y) == RED) {
                    //如果右叔叔节点是红节点
                    //把父节点染色为黑节点
                    setColor(parentOf(x), BLACK);
                    //把右叔叔节点染色为黑节点
                    setColor(y, BLACK);
                    //把祖父节点染色为红节点
                    setColor(parentOf(parentOf(x)), RED);
                    //把当前节点指向祖父节点，进行下一次循环
                    x = parentOf(parentOf(x));
                } else {
                    //如果右叔叔节点是黑节点
                    if (x == rightOf(parentOf(x))) {
                        //如果当前节点是父节点的右节点
                        //把当前节点指向父节点
                        x = parentOf(x);
                        //以当前节点为基础节点进行左旋
                        rotateLeft(x);
                    }
                    //把父节点染色成为黑节点
                    setColor(parentOf(x), BLACK);
                    //把祖父节点染色成为红节点
                    setColor(parentOf(parentOf(x)), RED);
                    //以祖父节点为基础节点进行右旋
                    rotateRight(parentOf(parentOf(x)));
                    //进行下一次循环
                }
                //*************************end************************
            } else {
                //====================如果父节点是祖父节点的右子节点============================
                //获取祖父节点的左子节点，即左叔叔节点
                Entry<K,V> y = leftOf(parentOf(parentOf(x)));
                if (colorOf(y) == RED) {
                    //如果左叔叔节点为红节点
                    //把父节点染色成为黑节点
                    setColor(parentOf(x), BLACK);
                    //把左叔叔节点染色成为黑节点
                    setColor(y, BLACK);
                    //把祖父节点染色成为红节点
                    setColor(parentOf(parentOf(x)), RED);
                    //把当前节点指向为祖父节点，进行下一次循环
                    x = parentOf(parentOf(x));
                } else {
                    //如果左叔叔节点是黑节点
                    if (x == leftOf(parentOf(x))) {
                        //如果当前节点是左子节点
                        //当前节点指向父节点
                        x = parentOf(x);
                        //以当前节点为基础节点进行右旋
                        rotateRight(x);
                    }
                    //把父节点染色成为黑节点
                    setColor(parentOf(x), BLACK);
                    //把祖父节点染色成为红节点
                    setColor(parentOf(parentOf(x)), RED);
                    //以祖父节点为基础节点进行左旋
                    rotateLeft(parentOf(parentOf(x)));
                    //进行下一次循环
                }
                //============================end=====================
            }
        }
    	//把根节点染色为黑节点
        root.color = BLACK;
    }
```

从上面可以看出 染色=>左旋=>右旋 或者是 染色=>右旋=>左旋

* 插入的节点默认为红节点，这样改动最小

* 如果叔叔节点是红节点

  * 把父节点和叔叔节点染色为黑节点，祖父节点染色为红节点。再以祖父节点为插入的新节点重新进行操作

* 如果叔叔节点是黑节点

  * 如果当前节点和叔叔节点同为左子节点
    * 把当前节点指向父节点
    * 以当前节点为基础节点进行右旋
    * 把父节点染色为黑节点
    * 把祖父节点染色为红节点
    * 以祖父节点为基础几点进行左旋
    * 然后再进行下一次循环
  * 如果当前节点为右子节点，叔叔节点为左子节点
    * 把父节点染色为黑节点
    * 把祖父节点染色为红节点
    * 以祖父节点为基础几点进行左旋
    * 然后再进行下一次循环
  * 如果当前节点和叔叔节点同为右子节点
    * 把当前节点指向父节点
    * 以当前节点为基础节点进行左旋
    * 把父节点染色为黑节点
    * 把祖父节点染色为红节点
    * 以祖父节点为基础几点进行左旋
    * 然后再进行下一循环

  * 如果当前节点为左子子节点，叔叔节点为右子节点
    * 把父节点染色为黑节点
    * 把祖父节点染色为红节点
    * 以祖父节点为基础几点进行左旋
    * 然后再进行下一循环

#### rotateLeft 方法

```java
//左旋
private void rotateLeft(Entry<K,V> p) {
        if (p != null) {
            //如果当前节点不为null
            //获取当前节点的右子节点
            Entry<K,V> r = p.right;
            //把当前节点的右子节点指向当前节点的右子节点的左子节点
            p.right = r.left;
            if (r.left != null)
                //如果当前节点的右子节点的左子节点不为null
                //把当前节点的右子节点的左子节点的父节点指向当前节点
                r.left.parent = p;
            //当前节点的右子节点的父节点指向当前节点的父节点
            r.parent = p.parent;
            if (p.parent == null)
                //如果当前节点的父节点为null，表示当前节点为根节点
                //把根节点指向当前节点的右子节点
                root = r;
            else if (p.parent.left == p)
                //如果当前节点是父节点的左子节点
                //把当前节点父节点的左子节点指向当前节点的右子节点
                p.parent.left = r;
            else
                //如果当前节点是父节点的右子节点
                //把当前节点父节点的右子节点指向当前节点的右子节点
                p.parent.right = r;
            //当前节点的右节点的左子节点指向当前节点
            r.left = p;
            //当前节点的父节点指向右子节点
            p.parent = r;
        }
    }
```

从上面可以看出，左旋的本质是

* 当前节点p成为当前节点p的右子节点pr的左子节点
* 当前节点p的右子节点变成右子节点pr的左子节点prl

#### rotateRight 方法

```java
//右旋
private void rotateRight(Entry<K,V> p) {
        if (p != null) {
            //如果当前节点不为null
            //获取当前节点左子节点
            Entry<K,V> l = p.left;
            //当前节点的左子节点指向当前节点左子节点的右子节点
            p.left = l.right;
            //如果当前节点的左子节点的右子节点不为null
            //把当前节点的左子节点的右子节点的父节点指向当前节点
            if (l.right != null) l.right.parent = p;
            //当前节点的左子节点的父节点指向当前节点的父节点
            l.parent = p.parent;
            if (p.parent == null)
                //如果当前节点的父节点为null，表示当前节点是根节点
                //把根节点指向当前节点的左子节点
                root = l;
            else if (p.parent.right == p)
                //如果当前节点是父节点的右子节点
                //把当前节点的父节点的右子节点指向当前节点的左子节点
                p.parent.right = l;
            //如果当前节点是父节点的左子节点
            //把当前节点的父节点的左子节点指向当前节点的左子节点
            else p.parent.left = l;
            //当前节点的左子节点的右节点指向当前节点
            l.right = p;
            //当前节点的父节点指向当前节点的左子节点
            p.parent = l;
        }
    }
```

从上面可以看出，右旋的本质是

* 当前节点p成为当前节点p的左子节点pl的右子节点
* 当前节点p的左子节点变成左子节点pl的右子节点plr

####  remove 方法

```java
//移除对应key的节点
public V remove(Object key) {
    	//使用getEntry获取key对应的节点
        Entry<K,V> p = getEntry(key);
        if (p == null)
            //如果不存在对应的节点，返回null
            return null;
		//获取对应节点的值
        V oldValue = p.value;
    	//使用deleteEntry删除节点
        deleteEntry(p);
    	//返回旧值
        return oldValue;
    }
```

#### clear 方法

```java
//清除所有元素
public void clear() {
    	//修改统计加1
        modCount++;
    	//元素数量置为0
        size = 0;
    	//根节点置为null
        root = null;
    }
```

#### deleteEntry 方法

```java
//删除节点
private void deleteEntry(Entry<K,V> p) {
    	//修改统计加1
        modCount++;
    	//元素数量减1
        size--;
        // If strictly internal, copy successor's element to p and then make p
        // point to successor.
        if (p.left != null && p.right != null) {
            //如果p节点的左右子节点都存在
            //查找仅仅比p大的元素
            Entry<K,V> s = successor(p);
            //把p的key和value设置为s的key和value
            p.key = s.key;
            p.value = s.value;
            //把p指向s
            p = s;
        } // p has 2 children

        // Start fixup at replacement node, if it exists.
    	//设置要替换的节点，如果左子节点存在使用左子节点，不存在使用右子节点
        Entry<K,V> replacement = (p.left != null ? p.left : p.right);

        if (replacement != null) {
            //如果p存在子节点
            // Link replacement to parent
            //***************replacement连接到p的父节点********************
            replacement.parent = p.parent;
            if (p.parent == null)
                //如果p的父节点为null，表示p节点为根节点
                //把根节点置为replacement
                root = replacement;
            else if (p == p.parent.left)
                //如果p节点是父节点的左子节点
                //把p节点的父节点的左子节点指向replacement
                p.parent.left  = replacement;
            else
                 //如果p节点是父节点的右子节点
                //把p节点的父节点的右子节点指向replacement
                p.parent.right = replacement;
		   //***************end********************
            //把p节点对外的引用都置为null
            p.left = p.right = p.parent = null;
            // Fix replacement
            if (p.color == BLACK)
                //如果p节点是黑节点
                //调用fixAfterDeletion使红黑树达到平衡
                fixAfterDeletion(replacement);
        } else if (p.parent == null) { // return if we are the only node.
            //如果p的父节点为null，表示p节点为根节点
            //把root置为null即可
            root = null;
        } else { //  No children. Use self as phantom replacement and unlink.
            //p不存在子节点的情况下
            if (p.color == BLACK)
                //如果p节点是黑节点
                //调用fixAfterDeletion使红黑树达到平衡
                fixAfterDeletion(p);
            if (p.parent != null) {
                //如果p的父节点不为null
                if (p == p.parent.left)
                    //如果p是父节点的左子节点
                    //把父节点的左子节点置为null
                    p.parent.left = null;
                else if (p == p.parent.right)
                     //如果p是父节点的右子节点
                    //把父节点的右子节点置为null
                    p.parent.right = null;
                //把p的父节点引用置为null
                p.parent = null;
            }
        }
    }
```

#### fixAfterDeletion 方法

```java
//删除节点后，平衡红黑树
private void fixAfterDeletion(Entry<K,V> x) {
    	//如果x不是根节点并且x是黑节点一直进行while循环
        while (x != root && colorOf(x) == BLACK) {
            if (x == leftOf(parentOf(x))) {
                //如果当前节点是父节点的左子节点
                //获取当前节点的父节点的右子节点，即右兄弟节点
                Entry<K,V> sib = rightOf(parentOf(x));
                if (colorOf(sib) == RED) {
                    //如果右兄弟节点是红节点
                    //把右兄弟节点染色成为黑节点
                    setColor(sib, BLACK);
                    //把当前节点的父节点染色称为红节点
                    setColor(parentOf(x), RED);
                    //以父节点为基础节点进行左旋
                    rotateLeft(parentOf(x));
                    //重新设置右兄弟节点
                    sib = rightOf(parentOf(x));
                }

                if (colorOf(leftOf(sib))  == BLACK &&
                    colorOf(rightOf(sib)) == BLACK) {
                    //如果右兄弟节点的子节点都是黑节点
                    //把右兄弟节点染色成为红节点
                    setColor(sib, RED);
                    //把当前节点指向父节点
                    x = parentOf(x);
                    //进行下一次循环
                } else {
                    //右兄弟节点的子节点有红节点的情况
                    if (colorOf(rightOf(sib)) == BLACK) {
                        //如果右兄弟节点的右子节点为黑节点(左子节点为红节点)
                        //把右兄弟节点的左子节点染色成为黑节点
                        setColor(leftOf(sib), BLACK);
                        //把右兄弟节点染色成为红节点
                        setColor(sib, RED);
                        //以右兄弟节点为基础节点，进行右旋
                        rotateRight(sib);
                        //重新设置右兄弟节点
                        sib = rightOf(parentOf(x));
                    }
                    //把右兄弟节点的颜色染色成为父节点的颜色
                    setColor(sib, colorOf(parentOf(x)));
                    //把父节点染色成为黑节点
                    setColor(parentOf(x), BLACK);
                    //把右兄弟节点的右子节点染色成为黑节点
                    setColor(rightOf(sib), BLACK);
                    //以父节点为基础节点进行左旋
                    rotateLeft(parentOf(x));
                    //把指向root节点，退出while循环
                    x = root;
                }
            } else { // symmetric 跟上面的对称
                //如果当前节点是父节点的右子节点
                //获取当前节点的父节点左子节点，即左兄弟节点
                Entry<K,V> sib = leftOf(parentOf(x));
                if (colorOf(sib) == RED) {
                    //如果左兄弟节点是红节点
                    //把左兄弟节点染色成为黑节点
                    setColor(sib, BLACK);
                    //把父节点染色成为红节点
                    setColor(parentOf(x), RED);
                    //以父节点为基础节点进行右旋
                    rotateRight(parentOf(x));
                    //重新设置左兄弟节点
                    sib = leftOf(parentOf(x));
                }

                if (colorOf(rightOf(sib)) == BLACK &&
                    colorOf(leftOf(sib)) == BLACK) {
                    //如果左兄弟节点的子节点都是黑节点
                    //把左兄弟节点染色成为红节点
                    setColor(sib, RED);
                    //把当前节点指向父节点
                    x = parentOf(x);
                    //进行下一次循环
                } else {
                    //左兄弟节点的子节点有红节点的情况
                    if (colorOf(leftOf(sib)) == BLACK) {
                        //如果左兄弟节点的左子节点为黑节点（右子节点为红节点）
                        //把左兄弟节点的右子节点染色成为黑节点
                        setColor(rightOf(sib), BLACK);
                        //把左兄弟节点染色成为红节点
                        setColor(sib, RED);
                        //以左兄弟节点为基础几点，进行左旋
                        rotateLeft(sib);
                        //重新设置左兄弟节点
                        sib = leftOf(parentOf(x));
                    }
                    //把左兄弟节点的颜色染色为父节点的颜色
                    setColor(sib, colorOf(parentOf(x)));
                    //把父节点染色成为黑节点
                    setColor(parentOf(x), BLACK);
                    //把左兄弟节点的左子节点染色成为黑节点
                    setColor(leftOf(sib), BLACK);
                    //以父节点为基础节点，进行右旋
                    rotateRight(parentOf(x));
                    //把当前节点指向根节点，退出while循环
                    x = root;
                }
            }
        }
		//把当前节点x染色成为黑节点
        setColor(x, BLACK);
    }
```

小结：

* 对红黑树的删除后的平衡操作还不是很理解
* 染色，左旋，右旋是平衡的关键
* 后面数据结构章节再结合图来进行说明，立个flag先