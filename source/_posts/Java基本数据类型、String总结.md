---
title: Java基本数据类型、String总结
tag: Java

---

<meta name="referrer" content="no-referrer" />



[TOC]

# Java基本数据类型、String总结

**声明**：部分面试题来自微信公众号**码农每日一题**

## 数据基本类型

|              |       |            |                                              |           |
| :----------: | :---: | :--------: | :------------------------------------------: | :-------: |
| 内置数据类型 | bit数 | 占用字节数 |                 范围和默认值                 |  封装类   |
|     byte     |   8   |     1      |       max：-2^7，min：2^7-1，默认值：0       |   Byte    |
|    short     |  16   |     2      |       max：-2^15，min：2^15，默认值：0       |   Short   |
|     int      |  32   |     4      |      max：-2^31，min：2^31-1，默认值：0      |  Integer  |
|     long     |  64   |     8      |     max：-2^63，min：2^63-1，默认值：0L      |   Long    |
|    float     |  32   |     4      |          单精度浮点数，默认值：0.0f          |   Float   |
|    double    |  64   |     8      | 双精度浮点数，浮点数的默认类型，默认值：0.0d |  Double   |
|   boolean    |   8   |     1      |         只有true和false，默认值false         |  Boolean  |
|     char     |  16   |     2      |     max：\u0000 (0)，min：\uffff (65535)     | Character |

**自动类型转换优先级** 参考[Java 基本数据类型](http://www.runoob.com/java/java-basic-datatypes.html)
从低到高：byte，short，char -> int -> long -> float -> double
需要注意：

> 1. 不能对boolean类型进行转换
> 2. 不能把对象类型传换成不相关类的对象
> 3. 把容量大的类型转换为容量小的类型必须使用强制转换
> 4. 转换过程中可能会出现溢出或损失精度
> 5. 浮点数到整数的转换是通过舍弃小数得到的，而不是四舍五入
> 6. 强制类型转换的数据类型必须是兼容的
> 7. 隐含强制类型转换，整数的默认类型是int

### 相关面试题

1. int、char、long各占多少字节数
    如上表格：分别4、2、8

2. 九种基本数据类型的大小，以及他们的封装类
    如上表，或者参考[九种基本数据类型，以及他们的封装类](https://blog.csdn.net/rabbit_in_android/article/details/49793813)

    > byte -> Byte 1字节
    > short -> Short 2字节
    > int -> Integer 4字节
    > long -> Long 8字节
    > float -> Float 4字节
    > double -> Double 8字节
    > boolean -> Boolean 1字节
    > char -> Character 2字节
    > void -> Void 没有字节数

3. 基本数据类型与对象的差别
    基本数据类型不是对象，也就是使用int、double、boolean等定义的变量、常量。
    基本数据类型没有可调用的方法。
    基本数据类型的值是直接保存在变量中，对象是引用类型，变量中保存的是实际对象的地址。参考[Java基本数据类型和引用类型的区别](https://www.jianshu.com/p/587625e390e1)

    赋值的区别：基本数据类型进行的赋值操作是复制、按值传递，修改一个的值不会影响另一个的值（赋值后是两个地址）；对象的赋值操作是引用，指向同一块地址，当其中一个对象进行修改后，修改的是地址里面的内容，另一个获取的内容也是修改后的。参考[JAVA中基本数据类型的引用与对象赋值的区别](https://blog.csdn.net/xiongmaodeguju/article/details/54409495)

4. 什么是自动装箱拆箱
    见下面

5. java 是否存在使得语句 i > j || i <= j 结果为 false 的 i、j 值
    存在，java 的数值 NaN 代表 not a number，无法用于比较，例如使 i = Double.NaN; j = i; 最后 i == j 的结果依旧为 false，

6. java 1.5 的自动装箱拆箱机制是编译特性还是虚拟机运行时特性？分别是怎么实现的？
    参考[Java 包装类型装箱拆箱基础面试题](https://mp.weixin.qq.com/s/akaEG8SPwlyaF2eXY_-AUA)

7. java 的 switch 语句能否作用在 byte 类型变量上，能否作用在 long 类型变量上，能否作用在 String 类型变量上？
    switch**不支持**long、boolean类型变量，支持byte、short、int、char、enum、Byte、Short、Integer、Character类型变量以及**Java 1.7开始支持String类型**。其中long类型的数据范围过大，大于int，会造成精度损失；byte范围小于int

    原理：原来用在 switch 语句中的字符串被替换成了对应的哈希值，而 case 子句的值也被换成了原来字符串常量的哈希值。经过这样的转换，Java 虚拟机所看到的仍然是与整数类型兼容的类型。在这里值得注意的是，在 case 子句对应的语句块中仍然需要使用 String 的 equals 方法来进行字符串比较。这是因为哈希函数在映射的时候可能存在冲突，多个字符串的哈希值可能是一样的。进行字符串比较是为了保证转换之后的代码逻辑与之前完全一样。

- byte、Byte
- long、Long
- boolean、Boolean
- int、Integer
    1. int型几个字节
        4个
    2. int与Integer的区别
        int是Java内置基本数据类型之一，Integer是int的封装类（包装类），int默认值为0，Integer默认值为null，Integer是一个类，可以new一个对象
    3. java 语句 Integer i = 1; i += 1; 做了哪些事情
        Integer i = 1;这一句声明了一个了Integer的对象，并赋值为1，然后jdk会进行自动装箱，将1转换为一个Integer对象，调用Integer.valueOf(int i)
        i += 1;这一句又进行了自动拆箱，由于i是一个对象引用，不能直接进行操作符运算，jdk会自动将i这个Integer对象拆箱为一个int值，其值为之前对象所拥有的值，然后将计算结果2赋值给i，由于i是一个Integer对象，2只是一个int型值，所以又会发生自动装箱，将2装进i这个对象引用
- double、Double
    1. 能否在不进行强制转换的情况下将一个 double 值赋值给 long 类型的变量？
        不行，从自动转换的优先级 从低到高：byte，short，char -> int -> long -> float -> double可以知道，double类型的范围比long更大，大向小转换必须强制转换
    2. java 中 3*0.1 == 0.3 将会返回什么？true 还是 false
        false，浮点数不能完全精确的表示出来，一般都会精度损失
- float、Float
    1. java 中 float f = 3.4; 是否正确？
        不正确，3.4是双精度数，float是单精度，属于向下转换，必须强制类型转换，或者3.4f（声明单精度浮点数）
- char、Character
    1. char占多少字节数
        2字节
    2. java 中 char 类型变量能不能储存一个中文的汉字，为什么？
        可以，char用于储存Unicode编码字符，Unicode字符集包含了汉字。但是有些生僻字不再Unicode字符集内，就不能储存。准确说Unicode包含了就可以存储。参考[Java:Unicode简介](https://blog.csdn.net/lubiaopan/article/details/4714909)

## String

1. String 为什么要设计成不可变的？
    （1）线程安全，不可变天生线程安全
    （2）String常被用做HashMap的key，如果可变会引发安全性问题
    （3）String常被用做数据库或接口的参数，可变会引发安全性问题
    （4）通过字符串常量池可以节省空间，String值是储存在常量池中的
    （5）每个String对应一个hashcode，再次使用不用重新计算，一定的性能优化

    [![只改变引用](https://user-gold-cdn.xitu.io/2018/3/11/162133576a03ec38?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)](https://user-gold-cdn.xitu.io/2018/3/11/162133576a03ec38?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)只改变引用

    为什么不可变？
    （1）String类本身是final的，不可以被继承
    （2）String类内部通过private final char value[]实现，从而保证了引用的不可变和对外的不可见
    （3）String内部通过良好的封装，不去改变value数组的值

    可以通过反射机制改变
    参考[Java中的String为什么是不可变的？ – String源码分析](https://blog.csdn.net/zhangjg_blog/article/details/18319521)[String 为什么要设计成不可变的](https://juejin.im/post/5aa1ee0c51882555677e2109)

2. String，Stringbuffer，Stringbuilder 区别
    [Java：String、StringBuffer和StringBuilder的区别](https://blog.csdn.net/kingzone_2008/article/details/9220691)
    [String,StringBuffer与StringBuilder的区别??](https://blog.csdn.net/rmn190/article/details/1492013)

3. switch能否用string做参数
    Java1.7以前不可以，1.7开始就可以。

4. string to integer
    先转换为int，再装箱为Integer

    ```java
    String str = "123";
    int i = Integer.parseInt(str);
    //1.5后会自动装箱，或者Integer integer = new Integer(i);或者Integer integer = Integer.valueOf(i);
    Integer integer = i;
    ```

5. String a=”abc”,String b = “abc”，创建了几个对象？
    一个对象，String a = “abc”创建一个对象在堆上，当执行String b = “abc”时，堆上已经有了”abc”这个对象，所以不会再新建，直接将b指向”abc”这个对象

6. 简单说说 String 的 isEmpty() 与 null 与 “” 的区别
    isEmpty()是JDK封装的方法，是字符串对象的方法，如果没有分配内存时（String s或String s = null），调用此方法会报空指针异常。值为空，变量是否被初始化
    null是判断字符串有没有分配内存空间，字符串是否指向一个内存地址，对象为空，表示数据未知或不可用，一般用于判断是不是实例化对象
    “”是一个有值的字符串，值比较特殊，是空字符串，但是有内存空间，有内存地址
    [![image.png](https://upload-images.jianshu.io/upload_images/4061843-1e1434d621a9d331.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-1e1434d621a9d331.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
    *注意*：null+”abc”，结果为”nulabc”（原因可以看源码）

7. 用 java 代码实现字符串的反转
    （1）使用JDK中StringBuffer、StringBuidler的反转方法reverse

    ```java
    public String reverse(String str) {
       if ((str == null) || (str.length() <= 1)) {
            return str;
        }
        return new StringBuffer(str).reverse().toString();
        //return new StringBuilder(str).reverse().toString();
    }
    ```

    （2）递归实现

    ```java
    public String reverse(String str) {
        if ((str == null) || (str.length() <= 1)) {
            return str;
        }
        return reverse(str.substring(1)) + str.charAt(0);
    }
    ```

8. 用 java 代码来检查输入的字符串是否回文（对称）
    （1）JDK现有API

    ```java
    public boolean isPalindrome(String str) {
        if (str == null) {
            return false;
        }
        StringBuilder strBuilder = new StringBuilder(str);
        strBuilder.reverse();
        return strBuilder.toString.equals(str);
    }
    ```

    （2）

    ```java
    public boolean isPalindrome(String str) {
        if (str == null) {
            return false;
        }
        int length = str.length();
        for (int i = 0; i < length / 2; i++) {
            if (str.charAt(i) != str.charAt(length - i -1)) {
                return false;
            }
        }
        return true;
    }
    ```

9. 用 java 代码写一个方法从字符串中删除给定字符？

    ```java
    public String removeChar(String str, char c) {
        if (str == null) {
            return null;
        }
        return str.replaceAll(Character.toString(c), "");
    }
    ```

10. 能不能继承String类
    不能，String类是final修饰的，被final修饰的类是不能被继承的

    ```java
    public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
        //此处省略
    }
    ```

11. 说说 String str = “hello world”; 和 String str = new String(“hello world”); 的区别？
    String str = “hello world”在编译期间生成了字面常量和符号引用，在运行期间字面常量”hello world”被存储在运行时常量池（只保存一份），后面如果有再创建相同的字符串（比如String str1 = “hello world”）不会再重新创建，只会将新的引用指向它。
    String str = new String(“hello world”)由于采用了new操作，所以每次new的时候都会产生一个新的对象，且new操作后，对象是放在堆内存中，每一个对象的地址都不同，哪怕是值相同（比如String str3 = new String(“hello world”)，str3与前面一个是不相同的两个对象）

    ```java
    public static void main(String[] args) {
        String str = "hello world";
        String str0 = "Hello world";
        String str1 = "hello world";
        String str2 = new String("hello world");
        String str3 = new String("hello world");
        //==比较的是内存地址，String不是基本类型所以不是比较的值
        System.out.println(str == str0);  //值不同，内存地址也应不同，在常量池中
        System.out.println(str == str1);  //相同的常量池，两个引用却是一个值
        System.out.println(str == str2);  //一个在常量池，一个在堆内存，地址不同
        System.out.println(str2 == str3);  //new操作，地址不同
        //由于String类重写了equals方法，所以equals比较的是字符串内容（值）
        System.out.println(str.equals(str0));  //值不同
        System.out.println(str.equals(str1));  //值相同
        System.out.println(str.equals(str2));  //值相同
        System.out.println(str2.equals(str3));  //值相同
    }
    ```

    结果：

    > false
    > true
    > false
    > false
    > false
    > true
    > true
    > true
    >
    > Process finished with exit code 0

12. 语句 String str = new String(“abc”); 一共创建了多少个对象？
    运行期间，new操作只创建了一个堆上的“abc”对象；在类加载过程中在运行时常量池中会先创建一个“abc”对象；创建了两个String对象。一个是“abc”，一个是指向“abc”的str。问题存在歧义

    ```java
    参考链接[请别再拿“String s = new String("xyz");创建了多少个String实例”来面试了吧](http://rednaxelafx.iteye.com/blog/774673/)
    ```

## 自动装箱拆箱

自动装箱拆箱是在Java 1.5开始引入

- 自动装箱：在1.5开始后，可以直接对基本类型的封装类进行直接赋值，jdk会自动将其转换。Java自动将原始类型值转换成对应的对象
- 自动拆箱：与装箱的过程正好相反，拆箱就是将对象转换为对应的原始类型

例子1：参考[Java自动装箱与拆箱及其陷阱](https://blog.csdn.net/JairusChan/article/details/7513045)
参考[深入剖析Java中的装箱和拆箱](https://www.cnblogs.com/dolphin0520/p/3780005.html)
参考[Java 包装类型装箱拆箱基础面试题](https://mp.weixin.qq.com/s/akaEG8SPwlyaF2eXY_-AUA)

```java
Integer i = 100;  //自动装箱，Java 1.5开始

int a = i;  //自动拆箱，Java 1.5
```

自动装箱过程中，程序会自动执行Integer.valueOf(int i)这个方法，来进行new对象，实现了自动装箱
自动拆箱过程中，进行赋值的时候，程序会自动先执行Integer.intValue()方法，来实现拆箱

装箱拆箱陷阱：
[![会导致如此的原因跟源码很有关系](https://upload-images.jianshu.io/upload_images/4061843-9084ea13c1d09fd1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-9084ea13c1d09fd1.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)会导致如此的原因跟源码很有关系

- 基本数据类型的自动装箱拆箱源码：
    （1）Integer

    ```java
    //装箱，缓存-128~127的值。可能缓存除此外的值（new了一个新的Integer对象）
    public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
    //拆箱
    public int intValue() {
        return value;
    }
    ```

    （2）Byte

    ```java
    //装箱
    public static Byte valueOf(byte b) {
        final int offset = 128;
        return ByteCache.cache[(int)b + offset];
    }
    //拆箱
    public byte byteValue() {
        return value;
    }
    ```

    （3）Long

    ```java
    //装箱，类似Integer，在-128~127时会强转为int缓存
    public static Long valueOf(long l) {
        final int offset = 128;
        if (l >= -128 && l <= 127) { // will cache
            return LongCache.cache[(int)l + offset];
        }
        return new Long(l);
    }
    //拆箱
    public long longValue() {
        return value;
    }
    ```

    （4）Character

    ```java
    //装箱，ASCII码<=127的时候一定缓存
    public static Character valueOf(char c) {
        if (c <= 127) { // must cache
            return CharacterCache.cache[(int)c];
        }
        return new Character(c);
    }
    //拆箱
    public char charValue() {
        return value;
    }
    ```

    （5）Short

    ```java
    //装箱，类似Long
    public static Short valueOf(short s) {
        final int offset = 128;
        int sAsInt = s;
        if (sAsInt >= -128 && sAsInt <= 127) { // must cache
            return ShortCache.cache[sAsInt + offset];
        }
        return new Short(s);
    }
    //拆箱
    public short shortValue() {
        return value;
    }
    ```

    （6）Double

    ```java
    //装箱，直接new了一个对象
    public static Double valueOf(double d) {
        return new Double(d);
    }
    //拆箱
    public double doubleValue() {
        return value;
    }
    ```

    （7）Float

    ```java
    //装箱，类似Double，直接new了一个对象
    public static Float valueOf(float f) {
        return new Float(f);
    }
    public float floatValue() {
        return value;
    }
    ```

    （8）String

    ```java
    //类似Double、Float，直接new对象，但是没怎么看懂String的拆箱（没找到）
    public static String valueOf(char data[]) {
        return new String(data);
    }
    ```

*一个简单的总结*：
Integer、Byte、Short、Long、Character五者的装箱拆箱过程基本差不多，出来Byte装箱不需要做判断，其余4个都需要判断是直接缓存还是new一个新的对象，缓存的时候对值有一定的要求；Double、Float没有对值进行判断，直接new了一个对象；String放在一起，它的装箱也有点类似Double