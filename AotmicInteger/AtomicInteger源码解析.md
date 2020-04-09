### 1. 类的定义

```java
public class AtomicInteger extends Number implements java.io.Serializable
```

从类的定义中可以看出

* AtomicInteger继承了Number类
* Number实现了java.io.Serializable接口，表示支持序列化

### 2. 字段属性

```java
//这个是用来实现CAS的
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;
//真正的值
private volatile int value;
//静态代码块初始化valueOffset
static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
```

从字段属性中可以看出

* AtomicInteger保存的值是使用volatile修饰的，表示AtomicInteger的原理是CAS加上volatile

### 3. 构造方法

```java
//传入一个初始值
public AtomicInteger(int initialValue) {
        value = initialValue;
    }
//默认构造方法
 public AtomicInteger() {
    }
```

从构造方法中可以看出

* AtomicInteger初始化可以指定初始值大小

#### 4. 方法

#### get 方法

```java
//获取值，直接返回value
public final int get() {
        return value;
    }
```

#### set 方法

```java
//设置新值
public final void set(int newValue) {
    	//直接赋值，因为value是volatile修饰的
        value = newValue;
    }
```

#### lazySet 方法

```java
public final void lazySet(int newValue) {
    	//这里调用的是unsafe.putOrderedInt()方法
        unsafe.putOrderedInt(this, valueOffset, newValue);
    }
```

#### getAndSet 方法

```java
//设置新值，并且返回旧值
public final int getAndSet(int newValue) {
    	//这里调用的是unsafe.getAndSetInt()方法
        return unsafe.getAndSetInt(this, valueOffset, newValue);
    }
```

#### compareAndSet 方法

```java
//CAS
public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
```

#### weakCompareAndSet 方法

```java
//实质还是CAS
public final boolean weakCompareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
```

#### getAndIncrement 方法

```java
//值加1然后返回旧值
public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }
```

#### getAndDecrement 方法

```java
//值减1然后返回旧值
public final int getAndDecrement() {
        return unsafe.getAndAddInt(this, valueOffset, -1);
    }
```

#### getAndAdd 方法

```java
//值增加指定的数然后返回旧值
public final int getAndAdd(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta);
    }
```

#### incrementAndGet 方法

```java
//值加1然后返回新值
public final int incrementAndGet() {
    	//这里可以看到unsafe.getAndAddInt（）方法返回的是旧值，然后手动加1
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }
```

#### decrementAndGet 方法

```java
//值减1然后返回新值
public final int decrementAndGet() {
    	//这里可以看到unsafe.getAndAddInt（）方法返回的是旧值，然后手动减1
        return unsafe.getAndAddInt(this, valueOffset, -1) - 1;
    }
```

#### addAndGet 方法

```java
//值增加指定数然后返回新值
public final int addAndGet(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
    }
```

#### getAndUpdate 方法

```java
//传入一个数值操作函数，获取旧值
public final int getAndUpdate(IntUnaryOperator updateFunction) {
        int prev, next;
        do {
            //获取旧值
            prev = get();
            //计算新值
            next = updateFunction.applyAsInt(prev);
            //循环使用CAS更新操作
        } while (!compareAndSet(prev, next));
    	//返回旧值
        return prev;
    }
```

#### updateAndGet 方法

```java
//传入一个数值操作函数，获取新值
public final int updateAndGet(IntUnaryOperator updateFunction) {
        int prev, next;
        do {
            //获取旧值
            prev = get();
            //计算新值
            next = updateFunction.applyAsInt(prev);
            //循环使用CAS更新操作
        } while (!compareAndSet(prev, next));
    	//返回新值
        return next;
    }
```

#### getAndAccumulate 方法

```java
public final int getAndAccumulate(int x,
                                      IntBinaryOperator accumulatorFunction) {
        int prev, next;
        do {
            //获取旧值
            prev = get();
            //计算新值
            next = accumulatorFunction.applyAsInt(prev, x);
            //循环使用CAS更新操作
        } while (!compareAndSet(prev, next));
    	//返回旧值
        return prev;
    }
```

#### accumulateAndGet 方法

```java
public final int accumulateAndGet(int x,
                                      IntBinaryOperator accumulatorFunction) {
        int prev, next;
        do {
            //获取旧值
            prev = get();
             //计算新值
            next = accumulatorFunction.applyAsInt(prev, x);
            //循环使用CAS更新操作
        } while (!compareAndSet(prev, next));
    	//返回新值
        return next;
    }
```

#### toString 方法

```java
public String toString() {
        return Integer.toString(get());
    }
```

#### intValue 方法

```java
//获取int类型的值，Number类中的方法，这里是重写
public int intValue() {
        return get();
    }
```

#### longValue 方法

```java
//获取long类型的值，Number类中的方法，这里是重写
public long longValue() {
    	//强转
        return (long)get();
    }
```

#### floatValue 方法

```java
//获取float类型的值，Number类中的方法，这里是重写
public float floatValue() {
        return (float)get();
    }
```

#### doubleValue 方法

```java
//获取double类型的值，Number类中的方法，这里是重写
public double doubleValue() {
        return (double)get();
    }
```

从方法中可以看出，所有修改值的操作都是基于unsafe对象操作的，这个里面都是native方法。CAS操作是基于内存地址的一个操作，通过C来操作的

