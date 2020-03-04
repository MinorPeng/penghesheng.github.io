---
title: Java操作符总结
tag: Java
category: Java

---

<meta name="referrer" content="no-referrer" />



[TOC]

# Java 操作符

几乎所有的操作符都只能操作基本类型，但“=”、“==”、“!=”可以操作所有的对象（String类支持“+”、“+=”）

## 赋值（=）

取右边的值，**复制**给左边。右值可以是任何常数、变量或表达式，左边却必须是一个已命名的变量（有一个物理空间用来存储右边的值）
比如`a = 4;`将4赋值给a变量，但是不能`4 = a;`，因为4是一个常数，不是变量就没有指向一块空间。基本类型的赋值操作，就是将一个地方的内容复制到另一个地方，比如基本数据类型`a = b;`就是将b指向的内容复制到a指向的地址，两个地址不同。
对象的赋值操作就有些不同了，比如对象类型操作`a = b;`，是将b的引用复制给a，两个同时指向一个地址。
[![image.png](https://upload-images.jianshu.io/upload_images/4061843-0673971e98e68759.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-0673971e98e68759.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)image.png
别名现象：如果a原本指向了一块地址，b指向的另一个地址，在执行`a=b;`后，a就会指向b的地址，a原来指向的地址就没有引用了，也就是丢失了（GC会自动清理这块地址），这种现象就叫别名现象

## 算术操作符

a=10， b=20
| 操作符 | 描述 | 例子 |
| :-: | :-: | :-: |
| + | 加法 | a + b = 30 |
| - | 减法 | a - b = -10 |
| | 乘法 | a b = 200 |
| / | 除法 | b / a = 2 |
| % | 取余 | b % a = 0 |
| ++ | 自增 | a++或++a结果都为11（但是有区别） |
| – | 自减 | a–或–a结果都为9（有区别） |
| | | |

- 自增自减
    a++和a–是后缀自增自减：先进行表达式运算，再进行自增自减
    ++a和–a是前缀自增自减：先进行自增自减，再进行表达式运算

## 关系操作符

关系操作符的结果是一个boolean值，计算的是操作数的值之间的关系

| 操作符 |                             描述                             |         例子         |
| :----: | :----------------------------------------------------------: | :------------------: |
|   ==   |      检查如果两个操作数的值是否相等，如果相等则条件为真      | 20 == 20（结果true） |
|   !=   |    检查如果两个操作数的值是否相等，如果值不相等则条件为真    | 20 != 30（结果true） |
|   >    |   检查左操作数的值是否大于右操作数的值，如果是那么条件为真   | 20 > 30（结果false） |
|   <    |   检查左操作数的值是否小于右操作数的值，如果是那么条件为真   | 20 < 30（结果true）  |
|   >=   | 检查左操作数的值是否大于或等于右操作数的值，如果是那么条件为真 | 20 >= 20（结果true） |
|   <=   | 检查左操作数的值是否小于或等于右操作数的值，如果是那么条件为真 | 20 <= 20（结果true） |
|        |                                                              |                      |

- ==和!=适用于所有对象
    基本类型判断值，引用类型判断内存地址是否相等

## 逻辑操作符

假设布尔变量A为真，变量B为假
| 操作符 | 描述 | 例子 |
| :-: | :-: | :-: |
| && | 称为逻辑与运算符。当且仅当两个操作数都为真，条件才为真 | (A && B)为假 |
| || | 称为逻辑或操作符。如果任何两个操作数任何一个为真，条件为真 | (A || B)为真 |
| ! | 称为逻辑非运算符。用来反转操作数的逻辑状态。如果条件为true，则逻辑非运算符将得到false | !(A && B)为真 |
| | | |

- 短路
    一旦能够明确无误地确定整个表达式的值，就不会计算后面的部分
    比如`boolean a = (20 > 30) && (20 > 10);`当发现前面20>30已经是false时，又是&&运算符，所以直接判定整个表达式为false，不会去计算20还是否大于10了

## 位运算符

假设整数变量A的值为60和变量B的值为13
| 操作符 | 描述 | 例子 |
| :-: | :-: | :-: |
| ＆ | 如果相对应位都是1，则结果为1，否则为0 | (A＆B)，得到12，即0000 1100 |
| | | 如果相对应位都是0，则结果为0，否则为1 | (A | B)得到61，即 0011 1101 |
| ^ | 如果相对应位值相同，则结果为0，否则为1 | (A ^ B)得到49，即 0011 0001 |
| 〜 | 按位取反运算符翻转操作数的每一位，即0变成1，1变成0 | (〜A)得到-61，即1100 0011 |
| << | 按位左移运算符。左操作数按位左移右操作数指定的位数 | A << 2得到240，即 1111 0000 |
| >> | 有符号按位右移运算符。左操作数按位右移右操作数指定的位数 | A >> 2得到15即 1111 |
| >>> | 无符号按位右移补零操作符。左操作数的值按右操作数指定的位数右移，移动得到的空位以零填充 | A>>>2得到15即0000 1111 |
| | | |

按位操作符&、|、^可以与=联合使用，以便合并运算和赋值（~说一元操作符，不可以和=联用）
左移操作符（<<）：在低位补0（数学上相当于乘以2的n次方，移n位）
有符号右移操作符（>>）：若符号为正，则在高位插0（数学上相当于移n位，除以2的n次方）；若符号为-，则在高位插1
无符号右移操作符（>>>）：使用“零扩展”，无论正负，都在高位插0
对char、byte、short类型的数值进行移位处理，会先转换为int型，得到的结果也是int型
long移位后仍是long
移位操作符可以与=联合使用

## 直接常量

```java
public class Main {

    public static void main(String[] args) {
        int i1 = 0x2f;
        System.out.println("i1:"+Integer.toBinaryString(i1));
        int i2 = 0X2F;
        System.out.println("i2:"+Integer.toBinaryString(i2));
        int i3 = 0177;
        System.out.println("i3:"+Integer.toBinaryString(i3));
        char c = 0xffff;
        System.out.println("c:"+Integer.toBinaryString(c));
        byte b = 0x7f;
        System.out.println("b:"+Integer.toBinaryString(b));
        short s = 0x7fff;
        System.out.println("s:"+Integer.toBinaryString(s));
        long n1 = 200L;
        System.out.println("n1:"+Long.toBinaryString(n1));
        long n2 = 200l;
        System.out.println("n2:"+Long.toBinaryString(n2));
        long n3 = 200;
        System.out.println("n3:"+Long.toBinaryString(n3));
        float f1 = 1;
        float f2 = 1F;
        float f3 = 1f;
        double d1 = 1d;
        double d2 = 1D;
    }
}
```

结果：

> i1:101111
> i2:101111
> i3:1111111
> c:1111111111111111
> b:1111111
> s:111111111111111
> n1:11001000
> n2:11001000
> n3:11001000

小写l容易造成混淆，所以long一般用大写的L。大写（小写）F都表示float，大写（小写）D都代表double

- 指数记数法
    1.39E-43f，表示e为自然数对数的基数，-43表示幂次，1.39表示系数

## 条件运算符 （? :）

> boolean-exp ? value0 : value1

如果boolean-exp（布尔表达式）的结果为true，就计算value0，否则计算value1

```java
public class Test {
   public static void main(String[] args){
      int a , b;
      a = 10;
      // 如果 a 等于 1 成立，则设置 b 为 20，否则为 30
      b = (a == 1) ? 20 : 30;
      System.out.println( "Value of b is : " +  b );

      // 如果 a 等于 10 成立，则设置 b 为 20，否则为 30
      b = (a == 10) ? 20 : 30;
      System.out.println( "Value of b is : " + b );
   }
}
```

结果：

> Value of b is : 30
> Value of b is : 20

## 字符串操作符 +和+=

用于连接字符串，如果一个表达式以一个字符串起头，那么后续的所有操作数都必须是字符串型（编译器会自动转为字符串）

```java
public class StringOperators {
    public static void main(String[] args) {
        int x = 0, y = 1, z = 2;
        String s = "\tx,y,z\t";
        System.out.println(s + x + y + z);
        System.out.println(x + s);
        s += "summed = ";
        System.out.println(s + (x + y + z));
        System.out.println(" " + x);
    }
}
/**
 * 结果：
 *    x,y,z    012
 * 0  x,y,z  
 *    x,y,z    summed = 3
 *  0
 */
```

## 类型转化操作符

类型转化原意是”模型铸造“，在适当的时候将一种数据类型自动转换为另一种

```java
public class Casting {
    public static void main(String[] args) {
        int i = 200;
        long lng = (long) i;  //对变量进行类型转换 int向long转换，可以不用声明
        lng = i;  //"Wideing" so cast not really required
        long lng2 = (long) 200;  //对数值进行类型转换
        lng2 = 200;
        //A "narrowing conversion"
        i = (int) lng2;  //cast required  long向int转换需要声明，强制转换，int范围比long小
    }
}
```

- 窄化转换
    将能容纳更多信息的数据转换成无法容纳更多信息的类型，可能对导致信息丢失
    例如long向int转换
- 扩展转换
    与窄化转换相反
    例如int向long转化
- 截尾和舍入
    将float或double转型为int型时，总是对数字执行截尾（29.7转int是29，不会保留小数位），如果想要得到舍入的结果就要使用Math.round()方法

## instanceof操作符

用于操作对象实例，检查该对象是否是一个特定类型（类类型或接口类型）
使用格式如下：
`( Object reference variable ) instanceof (class/interface type)`

如果运算符左侧变量所指的对象，是操作符右侧类或接口(class/interface)的一个对象，那么结果为真
例如：

```java
String name = "James";
boolean result = name instanceof String; // 由于 name 是 String 类型，所以返回真
```

如果被比较的对象兼容于右侧类型,该运算符仍然返回true
例如：

```java
class Vehicle {}

public class Car extends Vehicle {
   public static void main(String[] args){
      Vehicle a = new Car();
      boolean result =  a instanceof Car;
      System.out.println( result);  //返回true
   }
}
```

## 操作符优先级

优先级从高到低
| 类别 | 操作符 | 关联性 |
| :-: | :-: | :-: |
| 后缀 | () [] . (点操作符) | 左到右 |
| 一元 | + + - ！〜 | 从右到左 |
| 乘性 | / %| 左到右 |
| 加性 | + - | 左到右 |
| 移位 | >> >>> << | 左到右 |
| 关系 | >> = << = | 左到右 |
| 相等 | == != | 左到右 |
| 按位与 | & | 左到右 |
| 按位异或 | ^ | 左到右 |
| 按位或 | | | 左到右 |
| 逻辑与 | && | 左到右 |
| 逻辑或 | || | 左到右 |
| 条件 | ? : | 从右到左 |
| 赋值 | = += -= = /= %= >>= <<= &= ^= |= | 从右到左 |
| 逗号 | ，| 左到右 |

## 相关面试题

1. 如何比较两个对象的大小.或者换句话说,如何让对象之间有可比性
    实现Comparable接口
2. 定简单说说 Java 中 & 与 && 有什么区别？| 与 || 呢？
    &是按位与，是位运算，&&是逻辑与，是逻辑运算
    |是按位或，是位运算，||是逻辑或，是逻辑运算
    在进行逻辑判断时，&和|判断的是左右两边参与位运算的结果是否为true，而&&左边为false就不会处理右边的内容，||左边为true也不处理右边，&&、||是逻辑短路运算符
    优先级高到低：&、|、&&、||
3. Java 或者 Android 开发中可以通过哪些方式来保证并发安全的自增自减操作？
    java默认的自增自减是非并发安全
    通过synchronized代码块或方法保证并发安全
    通过组活动使用Lock锁
    通过JADK提供的AtomicInteger类

## 相关参考

**Java 核心编程思想**
[Java 运算符](http://www.runoob.com/java/java-operators.html)