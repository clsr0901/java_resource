### 1. 类的定义

```java
public class ThreadLocal<T>
```

从类的定义中可以看出

* ThreadLocal是一个泛型类

### 2. 字段属性

```java
//这些字段属性都是在内部类ThreadLocalMap中使用的
private final int threadLocalHashCode = nextHashCode();
private static AtomicInteger nextHashCode = new AtomicInteger();
private static final int HASH_INCREMENT = 0x61c88647;
```

### 3. 构造方法

```java
public ThreadLocal() { }
```

### 4. 方法

####  get 方法

```java
//获取ThreadLocal中保存当前线程的副本
public T get() {
    	//获取当前线程
        Thread t = Thread.currentThread();
    	//通过getMap获取当前线程中的ThreadLocalMap对象
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            //如果ThreadLocalMap不为null
            //获取ThreadLocalMap中的Entry对象
            //可以看出ThreadLocalMap.Entry对象是一个key-value对象
            //key为ThreadLocal对象， value为当前线程保存的副本
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
    	//如果当前线程中的ThreadLocalMap对象为null
    	//调用setInitialValue方法，设置初始化值。把副本添加进当前线程
        return setInitialValue();
    }
```

#### setInitialValue 方法

```java
//设置初始值
private T setInitialValue() {
    	//获取初始值
        T value = initialValue();
    	//获取当前线程
        Thread t = Thread.currentThread();
    	//获取当前线程的ThreadLocalMap对象
        ThreadLocalMap map = getMap(t);
        if (map != null)
            //如果当前线程的ThreadLocalMap不为null
            //直接设置值
            map.set(this, value);
        else
            //如果当前线程的ThreadLocalMap为null
            //先创建ThreadLocalMap对象，再设置值
            createMap(t, value);
        return value;
    }
```

#### set 方法

```java
//设置值
//本质是把值和当前的ThreadLocal对象设置到当前线程的ThreadLocalMap中
public void set(T value) {
    	//获取当前线程
        Thread t = Thread.currentThread();
    	//获取当前线程的ThreadLocalMap对象
        ThreadLocalMap map = getMap(t);
        if (map != null)
            //如果当前线程的ThreadLocalMap不为null
            //直接设置值
            map.set(this, value);
        else
            //如果当前线程的ThreadLocalMap为null
            //先创建ThreadLocalMap对象，再设置值
            createMap(t, value);
    }
```

#### remove 方法

```java
//移除当前线程的ThreadLocal
public void remove() {
    	//获取当前线程的ThreadLocalMap对象
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             //移除当前ThreadLocal
             m.remove(this);
     }
```

#### getMap 方法

```java
//获取指定线程的ThreadLocalMap对象
ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

#### createMap 方法

```java
//为指定线程创建一个新的ThreadLocalMap对象
//并把当前的ThreadLocal对象和firstValue添加进去
void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

### 5. 小结

ThreadLocal的实质是操作当前线程的ThreadLocalMap对象，而ThreadLocalMap对象中的Entry存储当前的ThreadLocal对象作为key，当前线程的副本值作为value。

ThreadLocal中的值实际还是存储在Thread对象中，ThreadLocal只不过是作为Thread的一个代理对象来操作Threa中的值。

ThreadLocalMap实质还是一个Map对象，类似于HashMap，这里不打算讲，有兴趣的可以自己看看