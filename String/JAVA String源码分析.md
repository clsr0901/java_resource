### 1. 类的定义

```java
public final class String implements java.io.Serializable, Comparable, CharSequence
```

从String的定义中，可以看出

- String 是一个final修饰的类，不能被继承

- String 实现了java.io.Serializable 接口，可以被序列化

- String实现了 Comparable<String> 可以用于比较大小（按顺序比较单个字符的ASCII码）

- String 实现了CharSequence 接口，表示String是一个可读的char序列，因为String就是一个char数组。

### 2. 字段属性

```java
 /** 用来存储字符串. */
    private final char value[];

    /** 缓存字符串的hash码 */
    private int hash; // Default to 0

    /** 序列化标识 */
    private static final long serialVersionUID = -6849794470754667710L;
```

从String字段属性中可以看出, String的本质就是一个final修饰的不可变的char数组

### 3. 构造方法

![String构造方法.jpg](https://user-gold-cdn.xitu.io/2020/3/1/170941a2e5657525?w=268&h=342&f=png&s=831)

从构造方法中可以看出

- 空构造函数，默认为“”空字符串

- 可以传入一个String字符串，类似于深拷贝

- 可以传入char数组，还可以指定偏移位置和拷贝的长度

- 可以传入int数组，还可以指定偏移位置和拷贝的长度

- 可以传入byte ascii数组，还可以指定偏移位置和拷贝的长度

- 可以传入byte数组，还可以指定偏移位置和拷贝的长度，并且指定编码格式，默认为utf-8

- 可以传入一个StringBuilder对象

- 可以传入一个StringBuffer对象

### 4. 方法

#### hashCode 方法

```java
   public int hashCode() {

        int h = hash;

        if (h == 0 && value.length > 0) {

            char val[] = value;



            for (int i = 0; i < value.length; i++) {

                h = 31 * h + val[i];

            }

            hash = h;

        }

        return h;

    }
```

String 的hash算法的核心就是中间for循环的h = 31 * h + val[i]，通俗表达就是s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]，为什么这里会选择31作为乘积因子？

- 31是一个不大不小的质数，是作为hashcode乘积的优选质数之一

- 31可以被JVM优化，31*i=(i << 5) - i。因为位移运算比乘法运算性能更好

#### equals 方法

```java
   public boolean equals(Object anObject) {

        if (this == anObject) {

            return true;

        }

        if (anObject instanceof String) {

            String anotherString = (String)anObject;

            int n = value.length;

            if (n == anotherString.value.length) {

                char v1[] = value;

                char v2[] = anotherString.value;

                int i = 0;

                while (n-- != 0) {

                    if (v1[i] != v2[i])

                        return false;

                    i++;

                }

                return true;

            }

        }

        return false;

    }
```

- 如果两个引用都相同，返回ture

- 如果两个字符串的长度和每个字符都相同，返回ture

- 其它的返回false

#### length 方法

```java
 public int length() {

        return value.length;

    }
```

- length方法就是返回char数组的长度

#### isEmpty 方法

```java
public boolean isEmpty() {

        return value.length == 0;

    }
```

- 判断字符串为空本质就是看char的长度是否等于0

#### charAt 方法

```java
public char charAt(int index) {

        if ((index < 0) || (index >= value.length)) {

            throw new StringIndexOutOfBoundsException(index);

        }

        return value[index];

    }
```

- 获取给定位置的字符

#### compareTo 方法

```java
public int compareTo(String anotherString) {

        int len1 = value.length;

        int len2 = anotherString.value.length;

        int lim = Math.min(len1, len2);

        char v1[] = value;

        char v2[] = anotherString.value;



        int k = 0;

        while (k < lim) {

            char c1 = v1[k];

            char c2 = v2[k];

            if (c1 != c2) {

                return c1 - c2;

            }

            k++;

        }

        return len1 - len2;

    }
```

- 比较两个字符串，从短的字符串长度进行循环比较

- 如果循环中两个字符不相等，则返回两个字符的Unicode值之差

- 如果循环中都相等，则返回两个字符串长度之差

- compareToIgnoreCase 则是先转换为大写再进行比较，比较过程同上面一样

#### startsWith 方法

```java
public boolean startsWith(String prefix, int toffset) {

        char ta[] = value;

        int to = toffset;

        char pa[] = prefix.value;

        int po = 0;

        int pc = prefix.value.length;

        // Note: toffset might be near -1>>>1.

        if ((toffset < 0) || (toffset > value.length - pc)) {

            return false;

        }
        //开始循环，循环次数为prefix的长度

        while (--pc >= 0) {
        //char数组从offset出开始
        //prefix的char数组从0开始

            if (ta[to++] != pa[po++]) {

                return false;

            }

        }

        return true;

    }
```

- 如果偏移量小于0返回false

- 如果偏移量加上prefix字符串的长度超过字符串char数组的长度，返回false

- 循环比较中，如果出现字符不相等，返回false

- 其它返回ture

- startsWith(String prefix) 的本质是 startsWith(prefix, 0)

- endsWith(String suffix) 的本质是 startsWith(suffix, value.length - suffix.value.length)

#### indexOf 方法

```java
public int indexOf(int ch, int fromIndex) {

        final int max = value.length;
        //如果fromIndex小于0， 就从0开始

        if (fromIndex < 0) {

            fromIndex = 0;

        } else if (fromIndex >= max) {
        //如果formIndex超过插入数组的长度，直接返回-1

            // Note: fromIndex might be near -1>>>1.

            return -1;

        }

        //如果传入的ch参数是单个字符，则直接进行遍历比较

        if (ch < Character.MIN_SUPPLEMENTARY_CODE_POINT) {

            // handle most cases here (ch is a BMP code point or a

            // negative value (invalid code point))

            final char[] value = this.value;
            //从formIndex出开始遍历

            for (int i = fromIndex; i < max; i++) {
            //找到返回1

                if (value[i] == ch) {

                    return i;

                }

            }
            //没找到返回-1

            return -1;

        } else {
        //多个字符，分为高位和低位分别进行比较

            return indexOfSupplementary(ch, fromIndex);

        }

    }
```

```java
private int indexOfSupplementary(int ch, int fromIndex) {

        if (Character.isValidCodePoint(ch)) {

            final char[] value = this.value;

            final char hi = Character.highSurrogate(ch);

            final char lo = Character.lowSurrogate(ch);

            final int max = value.length - 1;

            for (int i = fromIndex; i < max; i++) {

                if (value[i] == hi && value[i + 1] == lo) {

                    return i;

                }

            }

        }

        return -1;

    }
```

- indexOf(int ch) 的本质是 indexOf(ch, 0)

#### substring 方法

```java
public String substring(int beginIndex) {
        //检验起始下标合法性
        if (beginIndex < 0) {

            throw new StringIndexOutOfBoundsException(beginIndex);

        }
        //计算截取字符的长度

        int subLen = value.length - beginIndex;
        //校验截取字符长的合法性

        if (subLen < 0) {

            throw new StringIndexOutOfBoundsException(subLen);

        }
        //通过构造方法，返回新的字符传

        return (beginIndex == 0) ? this : new String(value, beginIndex, subLen);

    }
```

```java
public String substring(int beginIndex, int endIndex) {
        //检验起始下标合法性

        if (beginIndex < 0) {

            throw new StringIndexOutOfBoundsException(beginIndex);

        }
        //检验结束下标合法性

        if (endIndex > value.length) {

            throw new StringIndexOutOfBoundsException(endIndex);

        }
        //计算截取字符的长度

        int subLen = endIndex - beginIndex;
        //校验截取字符长的合法性

        if (subLen < 0) {

            throw new StringIndexOutOfBoundsException(subLen);

        }
        //通过构造方法，返回新的字符传

        return ((beginIndex == 0) && (endIndex == value.length)) ? this

                : new String(value, beginIndex, subLen);

    }
```

#### concat 方法

```java
public String concat(String str) {
        //获取连接字符串的长度

        int otherLen = str.length();
        //如果连接字符串的长度为0，直接返回当前字符串

        if (otherLen == 0) {

            return this;

        }
        //扩容，并讲value的内容拷贝到扩容后的数组中

        int len = value.length;

        char buf[] = Arrays.copyOf(value, len + otherLen);
        //将str的内容拷贝到buf后面去

        str.getChars(buf, len);
        //将新的字符数组创建成新的字符串并返回

        return new String(buf, true);

    }
```

#### replace 方法

```java
public String replace(char oldChar, char newChar) {
        //先判断两个字符是否相等

        if (oldChar != newChar) {

            int len = value.length;

            int i = -1;

            char[] val = value; /* avoid getfield opcode */

            //遍历比较，获取需要替换的下标位置

            while (++i < len) {

                if (val[i] == oldChar) {

                    break;

                }

            }

            if (i < len) {

                char buf[] = new char[len];
                //赋值需要替换的下标以前的字符

                for (int j = 0; j < i; j++) {

                    buf[j] = val[j];

                }
                //循环遍历剩下的字符

                while (i < len) {

                    char c = val[i];
                    //如果字符等于oldChar则替换为newChar，其它则不变

                    buf[i] = (c == oldChar) ? newChar : c;

                    i++;

                }
                //返回替换后的新字符串

                return new String(buf, true);

            }

        }

        return this;

    }
```

#### matches 方法

```java
public boolean matches(String regex) {
        //使用正则表达式进行匹配
        return Pattern.matches(regex, this);

    }
```

#### contains 方法

```java
public boolean contains(CharSequence s) {
        //调用indexOf方法

        return indexOf(s.toString()) > -1;

    }
```

#### replaceAll 方法

```java
public String replaceAll(String regex, String replacement) {
    //使用正则表达式替换
        return Pattern.compile(regex).matcher(this).replaceAll(replacement);

    }
```

#### split 方法

```java
public String[] split(String regex, int limit) {

        /* fastpath if the regex is a

         (1)one-char String and this character is not one of the

            RegEx's meta characters ".$|()[{^?*+\\", or

         (2)two-char String and the first char is the backslash and

            the second is not the ascii digit or ascii letter.

         */

        char ch = 0;
        //单个字符且不匹配".$|()[{^?*+\\"

        if (((regex.value.length == 1 &&

             ".$|()[{^?*+\\".indexOf(ch = regex.charAt(0)) == -1) ||
             //两个字符，第一个为'\\',第二个为0-9或者a-z或者A-Z

             (regex.length() == 2 &&

              regex.charAt(0) == '\\' &&

              (((ch = regex.charAt(1))-'0')|('9'-ch)) < 0 &&

              ((ch-'a')|('z'-ch)) < 0 &&

              ((ch-'A')|('Z'-ch)) < 0)) &&

            (ch < Character.MIN_HIGH_SURROGATE ||

             ch > Character.MAX_LOW_SURROGATE))

        {

            int off = 0;

            int next = 0;

            boolean limited = limit > 0;

            ArrayList list = new ArrayList<>();
            //遍历查找匹配ch的下标

            while ((next = indexOf(ch, off)) != -1) {
                //如果limit<=0或者list的长度小于limit-1，按段添加

                if (!limited || list.size() < limit - 1) {

                    list.add(substring(off, next));

                    off = next + 1;

                } else {    // last one

                    //assert (list.size() == limit - 1);
                    //截取最后一段添加到集合中

                    list.add(substring(off, value.length));

                    off = value.length;

                    break;

                }

            }

            //没有匹配的字符返回本身

            if (off == 0)

                return new String[]{this};



            // 如果limit<=0时，list的size小于limit时，截取添加剩余的字符串

            if (!limited || list.size() < limit)

                list.add(substring(off, value.length));



            // Construct result

            int resultSize = list.size();
            //当limit==0时，循环移除末尾长度为0的集合

            if (limit == 0) {

                while (resultSize > 0 && list.get(resultSize - 1).length() == 0) {

                    resultSize--;

                }

            }
            //将最后得到的数组返回

            String[] result = new String[resultSize];

            return list.subList(0, resultSize).toArray(result);

        }
        //多个字符（不匹配上面if条件的多个字符）使用正则表达式切割

        return Pattern.compile(regex).split(this, limit);

    }
```

- split(String regex) 的本质是split(regex, 0)

#### trim 方法

```java
public String trim() {
        //获取char数组长度，也叫结束位置

        int len = value.length;
        //其实位置

        int st = 0;

        char[] val = value;    /* avoid getfield opcode */

        //如果起始位置小于长度并且当前字符小于" "，起始位置后移一位

        while ((st < len) && (val[st] <= ' ')) {

            st++;

        }
        //如果起始位置小于长度并且结束位置的字符小于" "，结束位置前移一位

        while ((st < len) && (val[len - 1] <= ' ')) {

            len--;

        }
        //返回起始位置到结束位置的字符串

        return ((st > 0) || (len < value.length)) ? substring(st, len) : this;

    }
```
