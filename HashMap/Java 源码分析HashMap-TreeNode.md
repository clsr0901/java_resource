### 1. 类的定义

```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V>
```

从定义中可以看出

* TreeNode 是一个HashMap的一个静态内部类，且使用final修改，不能被继承

* TreeNode继承LinkedHashMap.Entry，表明TreeNode是一个键值对的对象

### 2. 字段属性

```java
        //父节点
        TreeNode<K,V> parent;  // red-black tree links
        //左子节点

        TreeNode<K,V> left;
        //右子节点

        TreeNode<K,V> right;
        //前一个节点

        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        //红黑标记

        boolean red;
        //hash值，父类继承
        final int hash;
        //key 父类继承

        final K key;
        //value 父类继承

        V value;
        //下一个节点 父类继承

        Node<K,V> next;
```

### 3. 构造函数

```java
TreeNode(int hash, K key, V val, Node<K,V> next) {
            //调用父类的构造函数

            super(hash, key, val, next);
        }
```

### 4. 方法

#### root 方法

```java
 //获取根节点
 final TreeNode<K,V> root() {
             //从当前节点循环，一直向上找父节点，直到节点的父节点为null，则此节点就是根节点
             //无限循环，把当前节点赋值给r

            for (TreeNode<K,V> r = this, p;;) {
                //查找r的父节点

                if ((p = r.parent) == null)
                    //如果父节点为空，此节点为根节点

                    return r;
                //把父节点赋值给r，继续循环

                r = p;
            }
        }
```
