### 1.类的定义

```java
public class LinkedList extends AbstractSequentialList implements List, Deque, Cloneable, java.io.Serializable
```

从定义中可以看出

- LinkedList是一个泛型类

- LinkedList继承了AbstractSequentialList抽象类

- LinkedList实现了List接口

- LinkedList实现了Deque接口，表示LinkedList可以用作队列

- LinkedList实现了Cloneable接口，表示LinkedList支持克隆

- LinkedList实现了java.io.Serializable接口，表示LinkedList支持序列化

![LinkedList的继承关系图](https://user-gold-cdn.xitu.io/2020/3/2/1709afceb4d39219?w=574&h=462&f=png&s=17404)

### 2. 字段属性

```java
//大小，不会被序列化
transient int size = 0;  
//头节点， 不会被序列化
transient Node<E> first;  
//尾节点， 不会被序列化 
transient Node<E> last;
```

* LinkedList内部维护了链表头节点和尾节点

* LinkedList内维护了自己的大小

* Node是LinkedList的一个静态内部类，它内部维护了左右节点，它是LinkedList的核心，标志着LinkedList是一个双向链表

### 3. 构造方法

```java
 //默认不带参的构造方法
 public LinkedList() {

    }
  //传入一个集合的构造方法
 public LinkedList(Collection<? extends E> c) {
        this();
        //把集合中的元素添加到链表中去，下面会详细介绍这个方法

        addAll(c);
    }
```

* LinkedList只有两个构造方法，一个不带参数，一个允许接收一个集合对象

### 4. 静态内部类Node

```java
private static class Node<E> {
        //具体的值

        E item;
        //下一个节点，本文中用左节点表示

        Node<E> next;
        //上一个节点，本文中用右节点表示

        Node<E> prev;
        //构造方法

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

* Node是LinkedList的核心

* Node内部维护了具体的值和左右节点，标志着LinkedList是一个双向链表

### 5. 方法

#### getFirst 方法

```java
//这个是Deque接口中的方法，获取头节点的值
public E getFirst() {
        //获取头节点

        final Node<E> f = first;
        //如果头节点为空，抛出异常

        if (f == null)
            throw new NoSuchElementException();
         //返回头节点的值

        return f.item;
    }
```

#### getLast 方法

```java
//这个是Deque接口中的方法，获取尾节点的值
public E getLast() {
        //获取尾节点

        final Node<E> l = last;
        //如果尾节点为空，抛出异常

        if (l == null)
            throw new NoSuchElementException();
        //返回尾节点的值

        return l.item;
    }
```

#### removeFirst 方法

```java
 //这个是Deque接口中的方法 移除头节点，返回头节点的值
 public E removeFirst() {
        //获取头节点

        final Node<E> f = first;
        //如果头节点为空抛出异常

        if (f == null)
            throw new NoSuchElementException();
        //移除头节点

        return unlinkFirst(f);
    }
```

#### unlinkFirst 方法

```java
//移除不为空的头节点
private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        //获取节点的值

        final E element = f.item;
        //获取下一个节点，后面这个节点会替代头节点

        final Node<E> next = f.next;
        //把节点的值和下一个节点的引用置为null

        f.item = null;
        f.next = null; // help GC
        //把头节点指向下一个节点

        first = next;

        if (next == null)
            //如果头节点下一个节点为null，把尾节点置为null
            last = null;
        else
            //把右节点的左引用置为null

            next.prev = null;
        //元素数量减1

        size--;
        //修改统计数量加1

        modCount++;
        //返回头节点的值

        return element;
    }
```

#### removeLast 方法

```java
//这是Deque接口中的方法，移除尾节点，返回尾节点的值
public E removeLast() {
        //获取尾节点

        final Node<E> l = last;
        //尾节点为空，抛出异常

        if (l == null)
            throw new NoSuchElementException();
        //移除尾节点

        return unlinkLast(l);
    }
```

#### unlinkLast 方法

```java
//移除尾节点，并返回尾节点的值
private E unlinkLast(Node<E> l) {
        // 获取尾节点的值

        final E element = l.item;
        //获取尾节点的左节点，后面这个节点会替代尾节点

        final Node<E> prev = l.prev;
        //设置尾节点的值为null

        l.item = null;
        //设置尾节点的左引用为null

        l.prev = null; // help GC
        //把尾节点指向左节点

        last = prev;

        if (prev == null)
            //如果尾节点的左节点为空，把头节点置为null

            first = null;
        else
            //把左节点的右引用置为null

            prev.next = null;
        //元素数量减一

        size--;
        //修改数量加1

        modCount++;
        //返回尾节点的值

        return element;
    }
```

#### addFirst 方法

```java
//这是Deque接口中的方法
public void addFirst(E e) {
        //添加头节点

        linkFirst(e);
    }
```

#### linkFirst 方法

```java
//添加头节点
private void linkFirst(E e) {
        //获取头节点的副本

        final Node<E> f = first;
        //创建新的节点，值为传入的值，右节点为头节点的副本，左节点为空

        final Node<E> newNode = new Node<>(null, e, f);
        //把内部维护的头节点指向新创创建的节点

        first = newNode;

        if (f == null)
        //如果头节点的副本为空，把内部维护的尾节点指向新创建的节点

            last = newNode;
        else
        //把头节点的副本的左引用指向新创建的节点

            f.prev = newNode;
        //元素数量加1

        size++;
        //修改数量加1

        modCount++;
    }
```

#### addLast 方法

```java
//这是Deque接口中的方法
public void addLast(E e) {
        //在尾部添加元素

        linkLast(e);
    }
```

#### linkLast 方法

```java
//添加尾节点
void linkLast(E e) {
        //获取尾节点的副本

        final Node<E> l = last;
        //创建新的节点，传入的值作为值，尾节点的副本作为左节点，右节点为null

        final Node<E> newNode = new Node<>(l, e, null);
        //把内部维护的尾节点指向新创建的节点

        last = newNode;

        if (l == null)
            //如果尾节点的副本为空，把内部维护的头节点指向新创建的节点
            first = newNode;
        else
            //尾节点的副本的右节点指向新创建的节点

            l.next = newNode;
        //元素数量加1

        size++;
        //修改统计加1

        modCount++;
    }
```

#### contains 方法

```java
 //是否包含该对象
 public boolean contains(Object o) {
         //调用indexof方法，-1表示不存在

        return indexOf(o) != -1;
    }
```

#### indexOf 方法

```java
//获取对象的下标
public int indexOf(Object o) {
        //默认下标为0

        int index = 0;
        if (o == null) {
            //对象为空，使用==比较
            //for循环遍历，x==null表示尾节点，遍历结束

            for (Node<E> x = first; x != null; x = x.next) {
                //查找成功返回下标

                if (x.item == null)
                    return index;
                //当前遍历不存在，下标加1

                index++;
            }
        } else {
            //对象不为空，使用equals比较

            for (Node<E> x = first; x != null; x = x.next) {
                //查找成功返回下标
                if (o.equals(x.item))
                    return index;
                //当前遍历不存在，下标加1

                index++;
            }
        }
        //不存在返回-1

        return -1;
    }
```

#### lastIndexOf 方法

```java
//获取指定值的最后一个元素的下标
public int lastIndexOf(Object o) {
        //默认下标为size

        int index = size;
        if (o == null) {
            //如果传入的值为null，使用==比较，x == null表示头节点的前一个节点

            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (x.item == null)
                    //比较成功，返回当前的下标

                    return index;
            }
        } else {
            //如果传入的值为null，使用equals比较，x == null表示头节点的前一个节点

            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (o.equals(x.item))
                    //比较成功，返回当前的下标

                    return index;
            }
        }
        //未找到，返回-1

        return -1;
    }
```

#### size 方法

```java
//返回元素数量
public int size() {
        //LinkedList内部维护了元素数量，直接返回

        return size;
    }
```

#### add 方法

```java
//添加元素
public boolean add(E e) {
        //添加元素到链表的末尾

        linkLast(e);
        //添加成功返回true

        return true;
    }
    
//为指定位置添加元素
public void add(int index, E element) {
        //检查下标

        checkPositionIndex(index);

        if (index == size)
            //如果下标等于size，从尾节点添加
            linkLast(element);
        else
            //在指定节点前面添加，通过node获取指定位置的节点

            linkBefore(element, node(index));
    }
```

#### linkBefore 方法

```java
//添加到指定节点的前面
void linkBefore(E e, Node<E> succ) {
        //获取指定节点的左节点

        final Node<E> pred = succ.prev;
        //新建节点，传入的值作为新节点的值，指定节点的左节点作为左节点，指定节点作为右节点

        final Node<E> newNode = new Node<>(pred, e, succ);
        //指定节点的左节点指向新建节点

        succ.prev = newNode;
        if (pred == null)
            //如果指定节点的左节点为null，头节点指向新建节点

            first = newNode;
        else
            //指定节点的左节点执行新建节点

            pred.next = newNode;
        //元素数量加1

        size++;
        //修改数量加1

        modCount++;
    }
```

#### remove 方法

```java
//移除指定值的节点
public boolean remove(Object o) {

        if (o == null) {
             //对象为空，使用==比较
             //for循环遍历，x==null表示尾节点，遍历结束
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    //查找到，移除元素

                    unlink(x);
                    return true;
                }
            }
        } else {
            //对象不为空，使用equals比较
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    //查找到，移除元素
                    return true;
                }
            }
        }
        //不存在，返回false

        return false;
    }
 
//移除指定下标的节点，返回移除节点的值
public E remove(int index) {
        //检查下标

        checkElementIndex(index);
        //移除指定元素，通过node来获取指定下标的元素

        return unlink(node(index));
    }

//移除第一个元素，并返回第一个元素的值
public E remove() {
        //调用remvoeFirst方法

        return removeFirst();
    }
```

#### unlink 方法

```java
//在列表中移除改元素，并返回元素的值
E unlink(Node<E> x) {
        // 获取该元素的值
        final E element = x.item;
        //获取该元素的右节点

        final Node<E> next = x.next;
        //获取该元素的左节点

        final Node<E> prev = x.prev;

        if (prev == null) {
            //如果左节点为空，表示当前元素为头节点
            //直接把头节点指向该节点的右节点

            first = next;
        } else {
            //该节点的左节点的右节点指向该节点的右节点

            prev.next = next;
            //该节点的左节点置为null

            x.prev = null;
        }

        if (next == null) {
            //如果右节点为空，表示当前节点为尾节点
            //直接把尾节点指向该节点的左节点

            last = prev;
        } else {
            //该节点右节点的左节点指向该节点的左节点

            next.prev = prev;
            //该节点的右节点置为null

            x.next = null;
        }
        //把当前节点的值置为null

        x.item = null;
        //元素数量减1

        size--;
        //修改数量加1

        modCount++;
        //返回当前节点的值

        return element;
    }
```

移除节点的核心

* 当前节点的左节点的右节点指向当前节点的右节点

* 当前节点的右节点的左节点指向当前节点的左节点

* 把当前节点的值、左节点、右节点都置为null

* 通俗的讲就是让左右跨过自己重新连接，但是要考虑是否是尾节点和头节点的情况

#### addAll 方法

```java
//添加集合元素，默认从尾部添加
public boolean addAll(Collection<? extends E> c) {
        //从尾部添加

        return addAll(size, c);
    }
    
//从指定位置添加集合
public boolean addAll(int index, Collection<? extends E> c) {
        //检查下标的合法性

        checkPositionIndex(index);
        //把集合转换为数组

        Object[] a = c.toArray();
        //获取数组的长度

        int numNew = a.length;
        //如果数组为空，直接返回false

        if (numNew == 0)
            return false;
        //pred表示添加位置的前一个元素
       //succ表示添加位置的元素
       //添加的元素在pred后面，succ前面

        Node<E> pred, succ;
        if (index == size) {
            //如果index等于size，表示从尾节点开始添加
            //所以succ等于null，pred就等于last

            succ = null;
            pred = last;
        } else {
            //根据下标获取当前元素和当前元素的前一个元素

            succ = node(index);

            pred = succ.prev;
        }
        //遍历数组

        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            //根据当前遍历的值作为节点的值，pred作为左节点，右节点为null创建新的节点

            Node<E> newNode = new Node<>(pred, e, null);

            if (pred == null)
                //如果前一个节点为null，头节点指向新创建的节点

                first = newNode;
            else
                //前一个节点的右节点指向新创建的节点

                pred.next = newNode;
            //把新创建的节点作为下一个节点的前一个节点
            //因为新增，所以不考虑后面节点的关系

            pred = newNode;
        }
        //循环结束后

        if (succ == null) {
            //如果succ为null，循环中最后新建的节点作为尾节点

            last = pred;
        } else {
            //把succ添加到遍历中最后新建的节点后面
            //最后新建节点的右节点指向succ

            pred.next = succ;
            //succ的左节点指向最后新建节点

            succ.prev = pred;
        }
        //重新计算元素数量

        size += numNew;
        //修改统计加1

        modCount++;
        //添加成功，返回true

        return true;
    }
    
 //检查下标的合法性
 private void checkPositionIndex(int index) {
        if (!isPositionIndex(index))
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
    
private boolean isPositionIndex(int index) {
        return index >= 0 && index <= size;
    }
```

#### node 方法

```java
//获取指定位置的元素
Node<E> node(int index) {
        // 比较指定位置跟元素数量的一半的关系

        if (index < (size >> 1)) {
            //如果小于元素数量的一半，从头节点开始遍历查找
            //获取头节点

            Node<E> x = first;
            //开始遍历到index

            for (int i = 0; i < index; i++)
                x = x.next;
            //返回元素

            return x;
        } else {
            //如果大于元素数量的一半，从尾节点开始遍历查找

            Node<E> x = last;
            //开始遍历到index
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            //返回元素

            return x;
        }
    }
```

#### clear 方法

```java
//清空列表
public void clear() {
        //循环遍历所有元素

        for (Node<E> x = first; x != null; ) {
            //保存下一个节点

            Node<E> next = x.next;
            //遍历到的元素的值，左节点，右节点都置为null

            x.item = null;
            x.next = null;
            x.prev = null;
            //继续遍历下一个

            x = next;
        }
        //把头节点和尾节点都置为null

        first = last = null;
        //元素数量置为0

        size = 0;
        //修改数量加1

        modCount++;
    }
```

#### get 方法

```java
//获取指定位置的值
public E get(int index) {
        //检查下标

        checkElementIndex(index);
        //使用node获取指定位置的节点，然后返回节点的值

        return node(index).item;
    }
```

#### set 方法

```java
//为指定位置赋值
public E set(int index, E element) {
        //检查下标

        checkElementIndex(index);
        //获取指定下标的节点

        Node<E> x = node(index);
        //保存指定下标的值

        E oldVal = x.item;
        //为指定下标赋新值

        x.item = element;
        //返回指定下标的旧值

        return oldVal;
    }
```

#### peek 方法

```java
//获取头节点的值
public E peek() {
        //获取头节点

        final Node<E> f = first;
        //如果头节点为空，返回null，不为空返回头节点的值

        return (f == null) ? null : f.item;
    }
```

#### element 方法

```java
//获取第一个元素的值
public E element() {
        //调用getFirst方法

        return getFirst();
    }
```

#### poll 方法

```java
//删除第一个元素，并获取第一个元素的值
public E poll() {
        //获取头节点

        final Node<E> f = first;
        //头节点为空，返回null， 不为空调用unlinkFirst移除头节点并返回头节点的值

        return (f == null) ? null : unlinkFirst(f);
    }
```

#### offer 方法

```java
 //添加一个元素
 public boolean offer(E e) {
        //调用add方法添加

        return add(e);
    }
```

#### offerFirst 方法

```java
 //添加元素到最前
 public boolean offerFirst(E e) {
         //调用addFirst方法添加

        addFirst(e);
        //添加成功返回true

        return true;
    }
```

#### offerLast 方法

```java
//添加元素到最后
public boolean offerLast(E e) {
        //调用addLast方法添加

        addLast(e);
        //添加成功返回true

        return true;
    }
```

#### peekFirst 方法

```java
//获取头节点素的值
public E peekFirst() {
        //获取头节点

        final Node<E> f = first;
        //如果头节点weik，返回null，不为空，返回头节点的值

        return (f == null) ? null : f.item;
     }
```

#### peekLast 方法

```java
//获取尾节点的值
public E peekLast() {
        //获取尾节点

        final Node<E> l = last;
        //如果尾节点为空，返回null，不为空，返回尾节点的值

        return (l == null) ? null : l.item;
    }
```

#### pollFirst 方法

```java
//移除头节点，并返回头节点的值
public E pollFirst() {
        //获取头节点

        final Node<E> f = first;
        //如果头节点为空，返回null， 不为空，调用unlinkFirst移除头节点并返回头节点的值

        return (f == null) ? null : unlinkFirst(f);
    }
```

#### pollLast 方法

```java
//移除尾节点，并返回尾节点的值
public E pollLast() {
        //获取尾节点

        final Node<E> l = last;
         //如果尾节点为空，返回null， 不为空，调用unlinkList移除尾节点并返回尾节点的值

        return (l == null) ? null : unlinkLast(l);
    }
```

#### push 方法

```java
//添加元素到链表最前面
public void push(E e) {
        //调用addFirst添加元素

        addFirst(e);
    }
```

#### pop 方法

```java
//弹出第一个元素的值
public E pop() {
        //调用removeFirst移除第一个元素并返回第一个元素的值

        return removeFirst();
    }
```

#### removeFirstOccurrence 方法

```java
//移除第一个发现的值对应元素
public boolean removeFirstOccurrence(Object o) {
        //调用remove移除元素

        return remove(o);
    }
```

#### removeLastOccurrence 方法

```java
//移除最有一个发现的值对应元素
public boolean removeLastOccurrence(Object o) {
        if (o == null) {
            //如果值为null使用==比较

            for (Node<E> x = last; x != null; x = x.prev) {
                if (x.item == null) {
                    //找到后移除对应元素

                    unlink(x);
                    //找到返回true

                    return true;
                }
            }
        } else {
            //值不为null，使用equals比较

            for (Node<E> x = last; x != null; x = x.prev) {
                if (o.equals(x.item)) {
                    //找到后移除对应元素
                    unlink(x);
                    //找到返回true
                    return true;
                }
            }
        }
        //未找到返回false
        return false;
    }
```

#### clone 方法

```java
//克隆
public Object clone() {
        //调用父类方法，生成一个新的链表

        LinkedList<E> clone = superClone();

        // 把克隆的列表置为初始状态

        clone.first = clone.last = null;
        clone.size = 0;
        clone.modCount = 0;

        // 遍历列表，把元素添加到clone的链表中

        for (Node<E> x = first; x != null; x = x.next)
            clone.add(x.item);
        //返回colne的链表

        return clone;
    }
```

* LinkedList的clone属于浅拷贝，只拷贝了对象的引用

#### toArray 方法

```java
//转换为数组
public Object[] toArray() {
        //新建一个size大小的数组

        Object[] result = new Object[size];
        int i = 0;
        //遍历列表，把列表的元素添加到数组中去

        for (Node<E> x = first; x != null; x = x.next)
            result[i++] = x.item;
        //返回数组

        return result;
    }
 
//把列表转换为指定数组
public <T> T[] toArray(T[] a) {
        //如果数组长度小于size，通过反射构建size大小的数组

        if (a.length < size)
            a = (T[])java.lang.reflect.Array.newInstance(
                                a.getClass().getComponentType(), size);
                                
        int i = 0;
        Object[] result = a;
        //遍历链表，为数组赋值

        for (Node<E> x = first; x != null; x = x.next)
            result[i++] = x.item;
        //如果数组长度大于size，把数组size处的值置为null

        if (a.length > size)
            a[size] = null;
        //返回数组

        return a;
    }
```

#### listIterator 方法

```java
 //从指定位置获取迭代器
 public ListIterator<E> listIterator(int index) {
         //检查下标

        checkPositionIndex(index);
        //创建一个新的迭代器

        return new ListItr(index);
    }
```

* ListItr是LinkedList内部的一个私有内部类

* ListItr的类定义如下

```java
private class ListItr implements ListIterator<E>

```
