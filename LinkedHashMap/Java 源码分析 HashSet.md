### 1. 类的定义

```java
public class HashSet<E> extends AbstractSet<E> implements Set<E>, Cloneable, java.io.Serializable
```

从定义中可以看出

* HashSet是一个泛型类
* HashSet继承了AbstractSet
* HashSet实现了Set接口
* HashSet实现了Cloneable接口，表示HashSet支持克隆
* HashSet实现了java.io.Serializable接口，表示HashSet支持序列化

### 2. 字段属性

```java
   //序列化版本号 
   static final long serialVersionUID = -5024744406713321676L;
    //使用HashMap存放数据，从泛型E可以看出HashSet的值就是HashMap中的key
    private transient HashMap<E,Object> map;
	//默认对象，存入map中的value
    private static final Object PRESENT = new Object();
```

从字段属性中可以看出

* HashSet底层使用HashMap存放数据
* HashSet的数据实际是HashMap中的key，value默认为创建的PRESENT对象

### 3.构造方法

```java
//默认构造方法
public HashSet() {
    	//初始化map
        map = new HashMap<>();
    }
//传入一个Collection的集合对象参数
public HashSet(Collection<? extends E> c) {
    	//使用Collection 参数的长度初始化map
    	//(c.size()/.75f) + 1作为默认长度，如果小于HashMap的默认长度16，使用HashMap的默认长度
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    	//把集合中的数据全部加入
        addAll(c);
    }

//传入初始化容量和负载因子
public HashSet(int initialCapacity, float loadFactor) {
    	//使用传入的容量和负载因子初始化HashMap
        map = new HashMap<>(initialCapacity, loadFactor);
    }

//传入初始化容量
public HashSet(int initialCapacity) {
    	//使用传入的初始化容量初始化HashMap
        map = new HashMap<>(initialCapacity);
    }

//这个构造方法的意义在于让LinkedHashSet使用
//传入初始化容量 负载因子初始化，dummy相当于一个占位符，没啥用可能是为了区分上面的两个参数的构造函数
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    	//注意，这个构造方法中map指向的是LinkedHashMap，多态
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
```

从构造方法中可以看出

* HashSet的构造函数本质就是初始化map
* map可以是一个HashMap也可以是一个LinkedHashMap
* <font color='red' size='4'>需要特别注意的这个LinkedHashMap,这个LinkedHashMap只是为了给子类LinkedHashSet使用。看LinkedHashSet的实现知道，所有的LinkedHashSet的构造方法都指向了 `HashSet(int initialCapacity, float loadFactor, boolean dummy)`这个父类的构造函数</font>

### 4.方法

####  iterator方法

```java
public Iterator<E> iterator() {
    	//本质是获取map的key的迭代器
        return map.keySet().iterator();
    }
```

####  size方法

```java
public int size() {
    	//本质是获取map的size
        return map.size();
    }
```

#### isEmpty方法

```java
public boolean isEmpty() {
    	//本质是判断map是否为空
        return map.isEmpty();
    }
```

#### contains 方法

```java
public boolean contains(Object o) {
    	//本质是通过map来判断
        return map.containsKey(o);
    }
```

#### add 方法

```java
public boolean add(E e) {
    	//调用map添加，注意value使用的是Object对象
    	//从这可以看出，HashSet的元素实际上就是HashMap的key
        return map.put(e, PRESENT)==null;
    }
```

#### remove方法

```java
public boolean remove(Object o) {
    	//本质是调用map的remove方法
        return map.remove(o)==PRESENT;
    }
```

#### clear 方法

```java
public void clear() {
    	//本质是清空map
        map.clear();
    }
```

####  clone方法

```java
@SuppressWarnings("unchecked")
    public Object clone() {
        try {
            //调用父类的克隆方法
            HashSet<E> newSet = (HashSet<E>) super.clone();
            //调用map的克隆方法
            newSet.map = (HashMap<E, Object>) map.clone();
            //从上面可以看出HashSet的克隆是浅拷贝，只克隆值得引用
            //返回克隆后的新对象
            return newSet;
        } catch (CloneNotSupportedException e) {
            throw new InternalError(e);
        }
    }
```