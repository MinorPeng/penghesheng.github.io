---
title: Java内部类
tag: Java

---

<meta name="referrer" content="no-referrer" />



[TOC]

# 内部类

内部类是定义在另一个类中的类。

**内部类的好处**：

1. 内部类方法可以访问该类定义所在的作用域中的数据，包括私有数据
2. 内部类可以对同一个包中的其他类隐藏起来
3. 当想要定义一个回调函数且不想编写大量代码时，使用匿名内部类比较便捷

先来个简单的例子

```java
/**
 * @author 14512 on 2018/7/26.
 */
public class OutClass {
    private String mName;
    protected String mAge;

    public InnerDemo(String name, String age) {
        this.mName = name;
        this.mAge = age;
    }

    public String getmName() {
        return mName;
    }

    public String getmAge() {
        return mAge;
    }

    class InnerClass {
        private String name;

        InnerClass() {
            mName = "内部访问外部name";
            mAge = "内部访问外部age";
            name = mName;
        }

        String getName() {
            return name;
        }
    }
}

//测试Test
/**
 * @author 14512 on 2018/7/25.
 */
public class Test {

    public static void main(String[] args) {
        OutClass outClass = new OutClass("外部name", "外部age");
        System.out.println(outClass.getmName()+"\t"+outClass.getmAge());
        //一种获取内部类的方法
        OutClass.InnerClass innerClass = outClass.new InnerClass();
        System.out.println(outClass.getmName()+"\t"+outClass.getmAge());
        System.out.println(innerClass.getName());
    }
}

/**
 * 输出结果：
 * 外部name	外部age  
 * 内部访问外部name	内部访问外部age  
 * 内部访问外部name  
 * Process finished with exit code 0
 */
```

这就是一个简单的内部类，简单来说，就是在一个类的内部再定义一个类。可以看到，内部类可以毫无顾忌的访问外部类的成员变量，不管外部的成员变量修饰的权限是什么。

上面的例子，程序在编译后生成的是两个Class文件，一个OutClass.Class，一个OutClass$InnerClass.Class。

内部类可以拥有private访问权限、protected访问权限、public访问权限及包访问权限。比如上面的例子，如果成员内部类Inner用private修饰，则只能在外部类的内部访问，如果用public修饰，则任何地方都能访问；如果用protected修饰，则只能在同一个包下或者继承外部类的情况下访问；如果是默认访问权限，则只能在同一个包下访问

## 成员内部类

```java
/**
 * @author 14512 on 2018/7/26.
 */
public class OutClass {
    private String mName;
    protected String mAge;
    private InnerClass innerClass = null;
    ...

    //获取内部类
    public InnerClass getInnerClass() {
        if (innerClass == null) {
            innerClass = new InnerClass();
        }
        return innerClass;
    }
    ...
}

//测试Test
/**
 * @author 14512 on 2018/7/25.
 */
public class Test {

    public static void main(String[] args) {
        OutClass outClass = new OutClass("外部name", "外部age");
        //获取内部类
        OutClass.InnerClass innerClass0 = outClass.getInnerClass();
        System.out.println(outClass.getmName()+"\t"+outClass.getmAge());
        //另一种获取内部类的方法
        OutClass.InnerClass innerClass = outClass.new InnerClass();
        ...
    }
}
```

代码注释中给出两种获取内部类的方法，一般建议写成外部类的公有方法。

- 注意
    成员内部类中不能存在任何static变量和方法（若要，就是静态类）
    成员内部类依附于外部类，必须先有外部类才能创建内部类

## 局部内部类

有这样一种内部类，它是嵌套在方法和作用域内的，对于这个类的使用主要是应用与解决比较复杂的问题，想创建一个类来辅助我们的解决方案，到那时又不希望这个类是公共可用的，所以就产生了局部内部类，局部内部类和成员内部类一样被编译，只是它的作用域发生了改变，它只能在该方法和属性中被使用，出了该方法和属性就会失效。

- 定义在方法内

```java
public class Parcel5 {
    public Destionation destionation(String str){
        class PDestionation implements Destionation{
            private String label;
            private PDestionation(String whereTo){
                label = whereTo;
            }
            public String readLabel(){
                return label;
            }
        }
        return new PDestionation(str);
    }

    public static void main(String[] args) {
        Parcel5 parcel5 = new Parcel5();
        Destionation d = parcel5.destionation("chenssy");
    }
}
```

- 定义在作用域类

```java
public class Parcel6 {
    private void internalTracking(boolean b){
        if(b){
            class TrackingSlip{
                private String id;
                TrackingSlip(String s) {
                    id = s;
                }
                String getSlip(){
                    return id;
                }
            }
            TrackingSlip ts = new TrackingSlip("chenssy");
            String string = ts.getSlip();
        }
    }

    public void track(){
        internalTracking(true);
    }

    public static void main(String[] args) {
        Parcel6 parcel6 = new Parcel6();
        parcel6.track();
    }
}
```

- 注意
    局部内部类就像是方法里面的一个局部变量一样，是不能有public、protected、private以及static修饰符的

## 匿名内部类

匿名内部类应该是平时我们编写代码时用得最多的，在编写事件监听的代码时使用匿名内部类不但方便，而且使代码更加容易维护

匿名内部类是唯一一种没有构造器的类。正因为其没有构造器，所以匿名内部类的使用范围非常有限，大部分匿名内部类用于接口回调。匿名内部类在编译的时候由系统自动起名为Outter$1.class。一般来说，匿名内部类用于继承其他类或是实现接口，并不需要增加额外的方法，只是对继承方法的实现或是重写。

```java
//定义一个接口
interface Inner {
    int getNumber();
}

public static void main(String[] args) {
    new Inner() {
            @Override
            public int getNumber() {
                return 0;
            }
    };
}
```

- 注意
    1. 匿名内部类没有访问修饰符
    2. new 匿名内部类，这个类首先是要存在的
    3. 当所在方法的形参需要被匿名内部类使用，那么这个形参就必须为final
    4. 匿名内部类没有构造方法

## 静态内部类（嵌套内部类）

有时候，使用内部类只是为了把一个类隐藏在另一个类的内部，并不需要内部类引用外围类对象。因此可以将内部类声明为static。静态内部类可以有静态域和方法；但是它的创建是不需要依赖于外部类，也不能使用任何外部类的非static成员变量和方法

```java
/**
 * @author 14512 on 2018/7/26.
 */
public class OutClass {
    private String mName;
    protected String mAge;
    private InnerClass innerClass = null;
    ...
    //静态内部类
    static class StaticInnerClass {
        private String staticName;
        public static String staticStr = "静态内部类静态常量";

        public String getStaticName() {
            //staticName = mName;  //错误，无法访问外部类的成员变量
            //staticName = getName();  //错误，无法访问外部类的方法
            return "静态内部类属性";
        }
    }
}

//测试
public class Test {

     public static void main(String[] args) {
        OutClass outClass = new OutClass("外部name", "外部age");
        System.out.println(outClass.getmName()+"\t"+outClass.getmAge());
        //一种获取非静态内部类的方法
        OutClass.InnerClass innerClass = outClass.new InnerClass();
        System.out.println(outClass.getmName()+"\t"+outClass.getmAge());
        System.out.println(innerClass.getName());
        //静态内部类的获取，实例化
        OutClass.StaticInnerClass staticInnerClass = new OutClass.StaticInnerClass();
        System.out.println(staticInnerClass.getStaticName());
    }
}

/**
 * 结果：
 *
 * 外部name	外部age
 * 内部访问外部name	内部访问外部age
 * 内部访问外部name
 * 静态内部类属性
 */
```

## 闭包

闭包（Closure）是一种能被调用的对象，它保存了创建它的作用域的信息

例如：一个接口程序员和一个基类作家都有一个相同的方法work，相同的方法名，但是其含义完全不同，这时候就需要闭包。

```java
class Writer {//作家基类
    void work(){};
}
interface programmer{//程序员接口
    void work();
}

public class WriterProgrammer extends Writer {
    @Override
    public void work(){
        //写作
    }
    public void code(){
        //写代码
    }
    class ProgrammerInner implements programmer{
        @Override
        public void work(){
            code();
        }
    }
}
```

参考[java内部类与内部类的闭包和回调](https://www.jianshu.com/p/367b138fe909)

## 内部类相关面试题

1. Java中Class类是用来干嘛的
    class类的实例表示java应用运行时的类(class ans enum)或接口(interface and annotation)（每个java类运行时都在JVM里表现为一个class对象，可通过类名.class,类型.getClass(),Class.forName(“类名”)等方法获取class对象）。数组同样也被映射为为class 对象的一个类，所有具有相同元素类型和维数的数组都共享该 Class 对象。基本类型boolean，byte，char，short，int，long，float，double和关键字void同样表现为 class 对象。

- 内部类

    1. 开发中使用 Java 匿名内部类有哪些注意事项（经验）
        （1）使用匿名内部类时必须是继承一个类或实现一个接口（二者不可兼得且只能继承一个类或者实现一个接口）。

        （2）匿名内部类中是不能定义构造函数的，如需初始化可以通过构造代码块处理。

        （3）匿名内部类中不能存在任何的静态成员变量和静态方法。

        （4）匿名内部类为局部内部类，所以局部内部类的所有限制同样对匿名内部类生效。

        （5）匿名内部类不能是抽象类，必须要实现继承的类或者实现接口的所有抽象方法

    2. 非静态内部类里面为什么不能有静态属性和静态方法？
        非静态内部类要依赖于外部类，必须先创建了外部类引用才会有非静态内部类。但是静态属性和静态方法（也就是static修饰）是在类加载时就开始存于内存，且是单独的一块存储空间，而此时类加载还未完成；如果这时调用了静态属性或静态方法，但是外部类并没有实例化，内部类也没加载，却试图调用内部类的静态成员和方法，就矛盾了。

    3. 什么是内部类？内部类的作用
        内部类是定义在另一个类中的类。

        内部类方法可以访问该类定义所在的作用域中的数据，包括私有数据
        内部类可以对同一个包中的其他类隐藏起来
        当想要定义一个回调函数且不想编写大量代码时，使用匿名内部类比较便捷
        可以避免修改接口而实现同一个类中两种同名方法的调用

    4. 内部类和静态内部类和匿名内部类，以及项目中的应用
        内部类就是定义在类中的类
        用static修饰的内部类就叫静态内部类
        没有构造器，没有名字的内部类就叫匿名内部类（view的点击事件）

    5. 成员内部类、静态内部类、局部内部类和匿名内部类的理解，以及项目中的应用
        将一个内部类当作外部类的成员变量，看作一个属性使用，就叫成员内部类
        static修饰的内部类就叫静态内部类
        内部类作用于某一个方法或某一个作用于的时候，就叫局部内部类
        没有名字的内部类叫匿名内部类

    6. 静态内部类的设计意图。
        为了把一个类隐藏在另一个类的内部，并不需要内部类引用外围类对象

    7. Static class 与non static class的区别
        静态内部类不需要有指向外部类的引用。但非静态内部类需要持有对外部类的引用。非静态内部类能够访问外部类的静态和非静态成员。静态内部类类不能访问外部类的非静态成员。他只能访问外部类的静态成员。一个非静态内部类不能脱离外部类实体被创建，一个非静态内部类可以访问外部类的数据和方法，因为他就在外部类里面。静态内部类可以独立于外部类

    8. 闭包和局部内部类的区别
        利用内部类实现闭包，局部内部类就是一个闭包结构

    9. Java 匿名内部类为什么不能直接使用构造方法，匿名内部类有没有构造方法？
        匿名内部类没有构造方法（没有名字），而且每次创建的匿名内部类同时被实例化后只能使用一次，所以就无法创建一个同名的构造方法，但是可以直接调用父类的构造方法。
        从JVM来说，匿名内部类是有构造方法的，是通过编译器在编译时生成的

    10. Java 中为什么成员内部类可以直接访问外部类的成员？
        编译后，成员内部类有一个指向外部类对象的引用，且成员内部类编译后构造方法有一个指向外部类对象的引用参数

    11. Java 1.8 之前为什么方法内部类和匿名内部类访问局部变量和形参时必须加 final？
        在这之前，普通局部变量或者形参的作用域是方法内，方法结束，局部变量和形参都会消失，而其匿名内部类或方法内的内部类的声明周期还没有结束（其他地方仍有引用），匿名内部类或者方法内部类如果想继续使用方法的局部变量就需要一些手段，所以 Java 在编译匿名内部类或者方法内部类时就有一个规定来解决生命周期问题，即如果访问的外部类方法的局部变量值在编译期能确定则直接在匿名内部类或者方法内部类里面创建一个常量拷贝，如果访问的外部类方法的局部变量值无法在编译期确定则通过构造器传参的方式来对拷贝进行初始化赋值。由此说明在匿名内部类或者方法内部类中访问的外部类方法的局部变量或者形参是内部类自己的一份拷贝，和外部类方法的局部变量或者形参不是一份，所以如果在匿名内部类或者方法内部类对变量做修改操作就一定会导致数据不一致性（外部类方法的参数不会跟着被修改，引用类型仅是引用，值修改不存在问题），为了杜绝数据不一致性导致的问题 Java 就要求使用 final 来保证，所以必须是 final 的。在 Java 1.8 开始我们可以不加 final 修饰符了，系统会默认添加，Java 将这个功能称为 Effectively final。

    12. 为什么Java里的匿名内部类只能访问final修饰的外部变量？
        匿名内部类和外部方法形成了一个闭包，因此匿名内部类能够访问外部方法的变量。但是匿名内部类的生命周期可能比外部的类要长，所以都会修改数据，容易导致数据不一致，所以Java就强制要求为final

部分参考[Java内部类详解](http://www.cnblogs.com/dolphin0520/p/3811445.html)