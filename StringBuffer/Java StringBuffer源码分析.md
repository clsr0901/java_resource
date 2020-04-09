### 1. 类的定义

```java
public final class StringBuffer extends AbstractStringBuilder implements java.io.Serializable, CharSequence
```

从StringBuffer定义中可以看出

* StringBuffer是final修饰的类，不可继承
* 继承了AbstractStringBuilder类
- 实现了java.io.Serializable接口，可以序列化
- 实现了CharSequence接口，表示StringBuffer一个可读的char序列

### 2. 字段属性

```java
//从父类AbstractStringBuilder继承
 char[] value;
 //从父类继承
 int count;
 //缓存，toString的时候才缓存，修改就置为null
 private transient char[] toStringCache;
 //序列化版本号
 static final long serialVersionUID = 4383685877147921099L;
```

从上面可以看出

* StringBuffer 本质是一个char数组
* 保存了char数组中可以用字符的长度，注意：<font color="red">count不是char数组的长度</font>
* toStringCache缓存了char数组可用字符的值，transient表示不可序列化

### 3. 构造方法

```java
//空构造函数，默认指定char的长度为16
 public StringBuffer() {
 super(16);
 }
 //传入char数组的初始化长度
 public StringBuffer(int capacity) {
 super(capacity);
 }
 //传入一个字符串，默认char数组长度为字符串长度加16
 //将str保存到char数组中去
 public StringBuffer(String str) {
 super(str.length() + 16);
 append(str);
 }
 //传入一个CharSequence对象，默认char数组长度为CharSequence对象长度加16
 //将CharSequence对象保存到char数组中去
 public StringBuffer(CharSequence seq) {
 this(seq.length() + 16);
 append(seq);
 }
```

从构造方法中可以看出

- StringBuffer中的char数组默认长度是16

- 可以自己指定默认长度

- 构造方法可以传入String对象和CharSequence对象，预留16个char的长度

### 4. 方法

* StringBuffer中所有的方法不都是使用synchronized修饰的，因此StringBuffer是线程安全的

#### length 方法

```java
@Override
    public synchronized int length() {
        //返回count表示可用字符的数量，跟char数组长度不一定相等

        return count;
    }
```

#### capacity 方法

```java
@Override
    public synchronized int capacity() {
        //value的长度表示容量

        return value.length;
    }
```

#### ensureCapacity 方法

```java
@Override
    public synchronized void ensureCapacity(int minimumCapacity) {
        //调用父类方法

        super.ensureCapacity(minimumCapacity);
    }
```

```java
 public void ensureCapacity(int minimumCapacity) {
         //参数验证

        if (minimumCapacity > 0)
            ensureCapacityInternal(minimumCapacity);
    }
 private void ensureCapacityInternal(int minimumCapacity) {
        // overflow-conscious code
        //确保容量，容量不足则扩容到minimumCapacity

        if (minimumCapacity - value.length > 0) {
            value = Arrays.copyOf(value,
                    newCapacity(minimumCapacity));
        }
    }
```

#### charAt 方法

```java
@Override
    public synchronized char charAt(int index) {
        //参数检查

        if ((index < 0) || (index >= count))
            throw new StringIndexOutOfBoundsException(index);
        //直接返回

        return value[index];
    }
```

#### append 方法

```java
@Override
    public synchronized StringBuffer append(String str) {
        //将缓存置为null

        toStringCache = null;
        //调用父类的方法

        super.append(str);
        return this;
    }

    public AbstractStringBuilder append(String str) {
        //参数检查
        if (str == null)
            return appendNull();
        //获取参数的长度
        int len = str.length();
        //char数组长度检查，如果长度不够会进行扩容，扩容的大小为count+len
        ensureCapacityInternal(count + len);
        //将传入的str添加到char数组后面
        str.getChars(0, len, value, count);
        //重新设置当前可用字符的数量
        count += len;
        return this;
    }
```

#### delete 方法

```java
 @Override
    public synchronized StringBuffer delete(int start, int end) {
        //将缓存置为null

        toStringCache = null;
        //调用父类方法

        super.delete(start, end);
        return this;
    }

public AbstractStringBuilder delete(int start, int end) {
        //参数检查
        if (start < 0)
            throw new StringIndexOutOfBoundsException(start);
        //如果结尾大于可用字符的数量把end的值设置为count
        if (end > count)
            end = count;
        //参数检查
        if (start > end)
            throw new StringIndexOutOfBoundsException();
        //计算需要删除的数量
        int len = end - start;
        if (len > 0) {
            //把start+len作为起始位置，count作为结束位置去覆盖start作为起始位置，覆盖数量为count-end
            //类似于后面的前移，覆盖中间删除部分
            System.arraycopy(value, start + len, value, start, count - end);
            //重新计算可用字符长度，这里删除的话应该减小
            count -= len;
        }
        return this;
    }
```

#### replace 方法

```java
 @Override
    public synchronized StringBuffer replace(int start, int end, String str) {
        //将缓存置为null
        toStringCache = null;
        //调用父类方法
        super.replace(start, end, str);
        return this;
    }

public AbstractStringBuilder delete(int start, int end) {
        //参数检查
        if (start < 0)
            throw new StringIndexOutOfBoundsException(start);
        //如果结尾大于可用字符的数量把end的值设置为count
        if (end > count)
            end = count;
        //参数检查
        if (start > end)
            throw new StringIndexOutOfBoundsException();
        //计算需要删除的数量
        int len = end - start;
        if (len > 0) {
            //把start+len作为起始位置，count作为结束位置去覆盖start作为起始位置，覆盖数量为count-end
            //类似于后面的前移，覆盖中间删除部分
            System.arraycopy(value, start + len, value, start, count - end);
            //重新计算可用字符长度，这里删除的话应该减小
            count -= len;
        }
        return this;
    }
```

#### substring 方法

```java
@Override
    public synchronized String substring(int start, int end) {
        //调用父类方法
        return super.substring(start, end);
    }

public String substring(int start, int end) {
        //参数验证

        if (start < 0)
            throw new StringIndexOutOfBoundsException(start);
        if (end > count)
            throw new StringIndexOutOfBoundsException(end);
        if (start > end)
            throw new StringIndexOutOfBoundsException(end - start);
        //从valule中返回start到end的字符串
        return new String(value, start, end - start);
    }
```

* substring(int start)的实质是sbustring(start, count)

#### insert 方法

```java
@Override
    public synchronized StringBuffer insert(int offset, String str) {
        //缓存设置为null
        toStringCache = null;
        //调用父类方法
        super.insert(offset, str);
        return this;
    }

public AbstractStringBuilder insert(int offset, String str) {
        //验证参数合法性
        if ((offset < 0) || (offset > length()))
            throw new StringIndexOutOfBoundsException(offset);
        //如果传入的String为空，替换为”null“字符串
        if (str == null)
            str = "null";
        //获取插入字符串的长度
        int len = str.length();
        //检查char数组容量，容量不够则扩容
        ensureCapacityInternal(count + len);
        //把offset作为起始位置，count作为结束位置去覆盖offset + len作为起始位置，覆盖数量为count - offset
        //类似于offset后面的后移，空出插入部分
        System.arraycopy(value, offset, value, offset + len, count - offset);
        //把str的内容填充进去
        str.getChars(value, offset);
        //重新计算count的值
        count += len;
        return this;
    }
```

#### indexOf 方法

```java
@Override
    public int indexOf(String str) {
        //注意，这个方法没有用synchronized修饰
        //调用父类方法
        return super.indexOf(str);
    }

@Override
    public synchronized int indexOf(String str, int fromIndex) {
        //调用父类方法
        return super.indexOf(str, fromIndex);
    }

public int indexOf(String str, int fromIndex) {
        //最后调用String的indexOf
        return String.indexOf(value, 0, count, str, fromIndex);
    }
```

#### lastIndexOf 方法

```java
@Override
    public int lastIndexOf(String str) {
        //注意，这个方法没有用synchronized修饰
        //调用父类方法

        return lastIndexOf(str, count);
    }

 @Override
    public synchronized int lastIndexOf(String str, int fromIndex) {
        //调用父类方法
        return super.lastIndexOf(str, fromIndex);
    }

public int lastIndexOf(String str, int fromIndex) {
        //最后调用的是String的lastIndexOf方法

        return String.lastIndexOf(value, 0, count, str, fromIndex);
    }
```

#### reverse 方法

```java
 @Override
    public synchronized StringBuffer reverse() {
        //将缓存置为null

        toStringCache = null;
        //调用父类方法

        super.reverse();
        return this;
    }

public AbstractStringBuilder reverse() {
        //是否含有unicode编码,包含Unicode编码的话，替换后高低位会改变位置
        boolean hasSurrogates = false;
        int n = count - 1;
        //采取二分法 n-1>>1 等同于 (n-1)/2
        for (int j = (n - 1) >> 1; j >= 0; j--) {
            //根据对称性左右替换
            int k = n - j;
            char cj = value[j];
            char ck = value[k];
            value[j] = ck;
            value[k] = cj;
            //判断是否含有unicode编码
            if (Character.isSurrogate(cj) ||
                    Character.isSurrogate(ck)) {
                hasSurrogates = true;
            }
        }
        //包含unicode编码
        if (hasSurrogates) {
            reverseAllValidSurrogatePairs();
        }
        return this;
    }

/**
     * Outlined helper method for reverse()
     */
    private void reverseAllValidSurrogatePairs() {
        for (int i = 0; i < count - 1; i++) {
            char c2 = value[i];
            //如果包含Unicode编码，将高低位位置改变回来
            if (Character.isLowSurrogate(c2)) {
                char c1 = value[i + 1];
                if (Character.isHighSurrogate(c1)) {
                    value[i++] = c1;
                    value[i] = c2;
                }
            }
        }
    }
```

#### toString 方法

```java
@Override
    public synchronized String toString() {
        //如果缓存为空，重新设置缓存

        if (toStringCache == null) {
            toStringCache = Arrays.copyOfRange(value, 0, count);
        }
        //直接使用缓存的数组

        return new String(toStringCache, true);
    }
```
