### 1. 类的定义

```java
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>
```

从定义可以看出

* LinkedHashMap是泛型类
* LinkedHashMap继承自HashMap
* LinkedHashMap实现了Map接口

### 2. 构造方法

```java
//传入初始化容量和负载因子
public LinkedHashMap(int initialCapacity, float loadFactor) {
    	//调用父类的构造方法
        super(initialCapacity, loadFactor);
    	//迭代时的顺序方式 accessOrder true 表示访问顺序 false 插入顺序
        accessOrder = false;
    }
//传入初始化容量
public LinkedHashMap(int initialCapacity) {
    	//调用父类的构造方法
        super(initialCapacity);
    	//迭代时的顺序方式 accessOrder true 表示访问顺序 false 插入顺序
        accessOrder = false;
    }
//默认构造方法
public LinkedHashMap() {
    	//调用父类默认的构造方法
        super();
    	//迭代时的顺序方式 accessOrder true 表示访问顺序 false 插入顺序
        accessOrder = false;
    }
//传入一个Map对象
public LinkedHashMap(Map<? extends K, ? extends V> m) {
    	//调用父类默认的构造方法
        super();
    	//迭代时的顺序方式 accessOrder true 表示访问顺序 false 插入顺序
        accessOrder = false;
    	//加入map中的元素
        putMapEntries(m, false);
    }
//传入初始化容量 负载因子 迭代顺序方式
public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
    	//调用父类的构造方法
        super(initialCapacity, loadFactor);
    	//设置迭代访问顺序方式
        this.accessOrder = accessOrder;
    }
```

从构造方法可以看出

* LinkedHashMap 迭代时默认的顺序时插入时的顺序
* 所有的构造方法都调用了父类HashMap的构造方法，可以指定初始化容量和负载因子

### 3.字段属性

```java
	//序列化版本号
	private static final long serialVersionUID = 3801124242820219131L;
	//头节点
    transient LinkedHashMap.Entry<K,V> head;
	//尾节点
    transient LinkedHashMap.Entry<K,V> tail;
	//迭代时的顺序方式 accessOrder true 表示访问顺序 false 插入顺序
    final boolean accessOrder;
```

从上面可以看出

* LinkedHashMap中的元素都是LinkedHashMap.Entry对象

* LinkedHashMap中保存了头节点和尾节点

* LinkedHashMap支持序列化，从HashMap中继承而来

* 可以指定迭代时的顺序

  * LinkedHashMap.Entry时LikedHashMap的一个静态内部类

    ```java
    static class Entry<K,V> extends HashMap.Node<K,V> {
        	//前后节点
            Entry<K,V> before, after;
            Entry(int hash, K key, V value, Node<K,V> next) {
                super(hash, key, value, next);
            }
        }
    ```

    从Entry的定义中可以看出

    * Entry是一个泛型类
    * Entry继承自HashMap.Node
    * Entry保存了前后节点，因此Entry是一个双向链表

### 4.方法

#### containsValue 方法

```java
//是否包含传入的value 
public boolean containsValue(Object value) {
     	//for循环遍历 从头节点开始
        for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {
            //获取当前遍历节点的值
            V v = e.value;
            if (v == value || (value != null && value.equals(v)))
                //如果当前节点的值等于查找的值，返回true
                return true;
        }
     	//没找到，返回false
        return false;
    }
```

#### get 方法

```java
//根据传入的key查找对应的vaule
public V get(Object key) {
        Node<K,V> e;
    	//通过调用父类HashMap的getNode方法查询对应的节点
        if ((e = getNode(hash(key), key)) == null)
            //没有对应的节点，返回null
            return null;
        if (accessOrder)
            //如果迭代顺序是访问顺序,移动当前节点到链表的末尾
            afterNodeAccess(e);
    	//返回节点的value
        return e.value;
    }
```

#### getOrDefault 方法

```java
//根据传入的key查找对应的value，如果不存在返回默认值
public V getOrDefault(Object key, V defaultValue) {
       Node<K,V> e;
    	//通过调用父类HashMap的getNode方法查询对应的节点
       if ((e = getNode(hash(key), key)) == null)
           //没找到对应的节点，返回默认值
           return defaultValue;
       if (accessOrder)
           //如果迭代顺序是访问顺序,移动当前节点到链表的末尾
           afterNodeAccess(e);
       //找到，返回节点的value
       return e.value;
   }
```

####  afterNodeAccess 方法

```java
void afterNodeAccess(Node<K,V> e) { // move node to last
    	//最后一个节点
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            //如果是迭代顺序是访问顺序 并且 传入的当前节点不是尾节点
            //p 当前节点的副本
            //b 当前节点的前一个节点
            //a 当前节点的后一个节点
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            //把当前节点的后一个节点引用置为null
            p.after = null;
            if (b == null)
                //如果当前节点的前一个引用位null，表示当前节点是头节点
                //把头节点设置为当前节点的后一个节点
                head = a;
            else
                //当前节点不是头节点
                //把当前节点的前一个节点的后一个节点的引用指向当前节点的后一个节点，即跨过当前节点
                b.after = a;
            if (a != null)
                //如果当前节点的后一个节点不为null
                //把当前节点的后一个节点的前一个节点的引用指向当前节点的前一个节点，即跨过当前节点
                a.before = b;
            else
                //如果当前节点的后一个节点为null，表示当前节点为尾节点
                //把最后一个节点指向当前节点
                last = b;
            if (last == null)
                //如果最后一个节点为null，表示只有一个元素
                //把头节点指向当前节点
                head = p;
            else {
                //最后一个节点不为null
                //下面两步的作用是，把当前节点挂到最后一个节点后面
                //把当前节点的前一个节点指向最后一个节点
                p.before = last;
                //把最后一个节点的下一个节点指向当前节点
                last.after = p;
            }
            //尾节点指向当前节点
            tail = p;
            //修改统计加1
            ++modCount;
        }
    }
```

#### clear 方法

```java
//清除所有元素
public void clear() {
    	//调用父类的清除方法
        super.clear();
    	//再把头节点和尾节点置为null
        head = tail = null;
    }
```

#### keySet 方法

```java
//获取所有key的Set集合
public Set<K> keySet() {
        Set<K> ks = keySet;
        if (ks == null) {
            //这里是重点，LinkedKeySet是LinkedHashMap的内部类
            //所以LinkedKeySet可以获取LinkedHashMap所有的属性
            //而且在使用的时候才会获取，不会在new的时候获取
            ks = new LinkedKeySet();
            keySet = ks;
        }
        return ks;
    }
```

#### values 方法

```java
//获取所有value的Collection集合
public Collection<V> values() {
        Collection<V> vs = values;
        if (vs == null) {
            //这里是重点，LinkedValues是LinkedHashMap的内部类
            //所以LinkedValues可以获取LinkedHashMap所有的属性
            //而且在使用的时候才会获取，不会在new的时候获取
            vs = new LinkedValues();
            values = vs;
        }
        return vs;
    }
```

#### entrySet 方法

```java
//获取所有键值对的Map集合
public Set<Map.Entry<K,V>> entrySet() {
        Set<Map.Entry<K,V>> es;
    	//entrySet = new LinkedEntrySet() 这里是重点，LinkedEntrySet是LinkedHashMap的内部类
        //所以LinkedEntrySet可以获取LinkedHashMap所有的属性
        //而且在使用的时候才会获取，不会在new的时候获取
        return (es = entrySet) == null ? (entrySet = new LinkedEntrySet()) : es;
    }
```

#### forEach 方法

```java
public void forEach(BiConsumer<? super K, ? super V> action) {
    	//参数校验
        if (action == null)
            throw new NullPointerException();
    	//获取修改统计的副本
        int mc = modCount;
        for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after)
            //根据Entry的链表特性来遍历
            //遍历的节点调用BiConsumer的方法
            action.accept(e.key, e.value);
        if (modCount != mc)
            //如果遍历中被其它线程修改了，抛出异常
            throw new ConcurrentModificationException();
    }
```

####  reinitialize 方法

```java
//重新初始化，覆盖父类的方法
void reinitialize() {
    	//调用父类的方法
        super.reinitialize();
    	//把头节点和尾节点都置为null
        head = tail = null;
    }
```

#### newNode 方法

```java
//创建一个新的节点，覆盖父类的方法
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    	//注意，这里没有调用父类的方法
    	//因为：LinkedHashMap的所有元素都是LinkedHashMap.Entry对象，是HashMap.Node的子类
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    	//把当前节点放到链表的尾部
        linkNodeLast(p);
    	//返回新创建的节点
        return p;
    }
```

#### linkNodeLast 方法

```java
//把传入的节点添加到链表尾部
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    	//保存尾节点的副本
        LinkedHashMap.Entry<K,V> last = tail;
    	//尾节点指向当前节点
        tail = p;
        if (last == null)
            //如果链表为空
            //把头节点指向当前节点
            head = p;
        else {
            //尾节点不为空
            //下面两步操作，把当前节点添加到链表的尾部
            //当前节点的前一个节点引用指向最后一个节点
            p.before = last;
            //最后一个节点的后一个节点的引用指向当前节点
            last.after = p;
        }
    }
```

### 5. 小结

* 有很多方法在HashMap中实现，LinkedHashMap只重写了方法中的一部分方法，这些需要结合HashMap一起看，建议在项目中用到的时候进行查看，这部分方法后续在项目中用到会补充进来