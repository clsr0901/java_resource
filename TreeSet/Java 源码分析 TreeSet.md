### 1. 类的定义

```java
public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable
```

从类的定义中可以看出

* TreeSet是一个泛型类
* TreeSet继承了AbstractSet
* TreeSet实现了NavigableSet接口
* TreeSet实现了Cloneable接口，表示TreeSet支持克隆
* TreeSet实现了java.io.Serializable接口，表示TreeSet支持序列化

### 2. 字段属性

```java
//存储数据的容器，可以看出TreeSet使用的是NavigableMap来存储
//重点：NavigableMap是一个接口，TreeMap是NavigableMap的实现类，所以TreeSet存储使用的容器是TreeMap
private transient NavigableMap<E,Object> m;
//这个很熟悉了，HashSet里面也有它。相当于一个占位符，TreeSet只是用Map中的key，value默认的就是这个
private static final Object PRESENT = new Object();
```

从字段属性中可以看出

* TreeSet实际上是一个TreeMap，是一个只是用key的TreeMap

### 3. 构造方法

```java
//传入一个NavigableMap对象
TreeSet(NavigableMap<E,Object> m) {
        this.m = m;
    }
//这个很重要，默认构造器
public TreeSet() {
    	//默认构造器使用的是TreeMap
        this(new TreeMap<E,Object>());
    }
//传入一个比较器，这个是TreeMap中要使用的
public TreeSet(Comparator<? super E> comparator) {
    	//默认使用TreeMap
        this(new TreeMap<>(comparator));
    }
//传入一个Collection集合对象
public TreeSet(Collection<? extends E> c) {
    	//调用默认的构造器初始化
        this();
    	//添加集合中的元素
        addAll(c);
    }
//传入一个SortedSet对象
public TreeSet(SortedSet<E> s) {
    	//使用SortedSet对象的构造器初始化
        this(s.comparator());
    	//添加集合中的元素
        addAll(s);
    }
```

从构造方法中可以看出

* 可以使用NavigableMap对象来初始化TreeSet，默认使用的是NavigableMap的实现类TreeMap
* 可以使用一个比较器来初始化TreeSet
* 可以使用一个Collection对象量初始化TreeSet
* 可以使用一个SortedSet对象初始化TreeSet

### 4. 方法

#### size 方法

```java
//获取元素数量
public int size() {
    	//返回m的元素数量，m是存储数据的容器
        return m.size();
    }
```

#### isEmpty 方法

```java
//判断当前是否为空
public boolean isEmpty() {
    	//返回m是否为空
        return m.isEmpty();
    }
```

#### contains 方法

```java
//判断是否包含传入的元素
public boolean contains(Object o) {
    	//把传入的元素作为key，判断m中是否包含该key
        return m.containsKey(o);
    }
```

#### add 方法

```java
//添加元素
public boolean add(E e) {
    	//传入的元素作为key，PRESENT作为默认的value添加到map中
        return m.put(e, PRESENT)==null;
    }
```

#### remove 方法

```java
//移除传入的元素
public boolean remove(Object o) {
    	//把传入的对象作为key，map中移除该key
        return m.remove(o)==PRESENT;
    }
```

#### addAll 方法

```java
//添加集合中所有的元素
public  boolean addAll(Collection<? extends E> c) {
        // m instanceof TreeMap 只有TreeMap才能添加
    	//c instanceof SortedSet 传入的必须是SortedSet对象，应该是要使用SortedSet对象的比较器
    	//m中必须没有元素
    	//c中包含元素
    	//满足上面这四条才会调用这个方法，不然会调用父类的方法添加元素
        if (m.size()==0 && c.size() > 0 &&
            c instanceof SortedSet &&
            m instanceof TreeMap) {
            //强转
            SortedSet<? extends E> set = (SortedSet<? extends E>) c;
            TreeMap<E,Object> map = (TreeMap<E, Object>) m;
            //获传入集合的取比较器
            Comparator<?> cc = set.comparator();
            //获取当前的比较强
            Comparator<? super E> mc = map.comparator();
            if (cc==mc || (cc != null && cc.equals(mc))) {
                //如果比较器相等
                //使用TreeMap的addAllForTreeSet方法添加
                map.addAllForTreeSet(set, PRESENT);
                return true;
            }
        }
    	//调用父类的方法
        return super.addAll(c);
    }
```

#### subSet 方法

```java
//获取指定开始到结尾位置的子集 
public NavigableSet<E> subSet(E fromElement, boolean fromInclusive,
                                  E toElement,   boolean toInclusive) {
    	//使用TreeMap的subMap方法来获取指定开始到结尾位置的子集
    	//再用TreeSet包装成一个新的对象返回
        return new TreeSet<>(m.subMap(fromElement, fromInclusive,
                                       toElement,   toInclusive));
    }

public SortedSet<E> subSet(E fromElement, E toElement) {
        return subSet(fromElement, true, toElement, false);
    }
```

#### headSet 方法

```java
//获取小于等于传入节点的子集
public NavigableSet<E> headSet(E toElement, boolean inclusive) {
    	//使用TreeMap的headMap方法来获取小于等于传入节点的子集
    	//再用TreeSet包装成一个新的对象返回
        return new TreeSet<>(m.headMap(toElement, inclusive));
    }

public SortedSet<E> headSet(E toElement) {
        return headSet(toElement, false);
    }
```

#### tailSet 方法

```java
//获取大于等于传入节点的子集
public NavigableSet<E> tailSet(E fromElement, boolean inclusive) {
    	//使用TreeMap的tailMap方法来获取大于等于传入节点的子集
    	//再用TreeSet包装成一个新的对象返回
        return new TreeSet<>(m.tailMap(fromElement, inclusive));
    }

public SortedSet<E> tailSet(E fromElement) {
        return tailSet(fromElement, true);
    }
```

#### comparator方法

```java
//获取当前比较器 
public Comparator<? super E> comparator() {
    	//返回m的比较器
        return m.comparator();
    }
```

#### first 方法

```java
//获取第一个元素
public E first() {
    	//调用m的firstKey方法
        return m.firstKey();
    }
```

#### last 方法

```java
//获取最后一个元素
public E last() {
    	//调用m的lastKey方法
        return m.lastKey();
    }
```

#### lower 方法

```java
//获取仅仅比传入元素小的元素
public E lower(E e) {
    	//调用m的lowerKey方法
        return m.lowerKey(e);
    }
```

#### floor 方法

```java
//获取仅仅比传入元素小的元素
public E floor(E e) {
    	//调用m的floorKey方法
        return m.floorKey(e);
    }
```

#### ceiling 方法

```java
//获取仅仅比传入元素大的元素
public E ceiling(E e) {
    	//调用m的ceilingKey方法
        return m.ceilingKey(e);
    }
```

####  higher 方法

```java
//获取仅仅比传入元素大的元素
public E higher(E e) {
    	//调用m的higherKey方法
        return m.higherKey(e);
    }
```

####  pollFirst 方法

```java
//获取并删除第一个元素
public E pollFirst() {
    	//调用m的pollFirstEntry方法获取第一个节点
        Map.Entry<E,?> e = m.pollFirstEntry();
    	//节点不存在返回null，存在返回对应的key
        return (e == null) ? null : e.getKey();
    }
```

#### pollLast 方法

```java
//获取并删除最后一个节点
public E pollLast() {
    	//调用m的pollFirstEntry方法获取最后一个节点
        Map.Entry<E,?> e = m.pollLastEntry();
    	//节点不存在返回null，存在返回对应的key
        return (e == null) ? null : e.getKey();
    }
```

#### clone 方法

```java
//克隆
public Object clone() {
        TreeSet<E> clone;
        try {
            //调用父类的克隆方法
            clone = (TreeSet<E>) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new InternalError(e);
        }
		//把对象的map包装成一个新的TreeMap赋值个克隆对象
    	//浅拷贝，m只拷贝了对象的引用
        clone.m = new TreeMap<>(m);
    	//返回克隆对象
        return clone;
    }
```



方法小结：

* 从方法中可以看出，TreeSet实质是调用TreeMap对应的方法
* 可以把TreeSet看作是一个不管value的TreeMap