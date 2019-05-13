---
title: Java三大特性、抽象类以及接口相关
tag: Java
---

# Java三大特性、抽象类以及接口相关

**声明**：部分面试题来自微信公众号**码农每日一题**

## 封装

概念：对象属性的封装隐藏，方法的公开；属性私有化后，则其他类不能直接使用对象名.属性名访问，必须通过提供的公开方法。控制在程序中属性的读和修改的访问级别。

目的：增强安全性和简化编程，使用者不必了解具体的实现细节，而只是要通过外部接口，一特定的访问权限来使用类的成员。

基本要求：把所有的属性私有化，对每个属性提供getter和setter方法

一个小例子：

```java
//封装之前
public class TestDemo {
    public String name;
    public String address;
    public String age;

    public TestDemo(String name, String address, String age) {
        this.name = name;
        this.address = address;
        this.age = age;
    }

    TestDemo() {}
}

//调用是这样的
public class Test {

    public static void main(String[] args) {
        TestDemo testDemo = new TestDemo();
        String name = testDemo.name;
        String address = testDemo.address;
        String age = testDemo.age;
        testDemo.name = "hello";
    }
}


//封装之后
public class TestDemo {
    private String name;
    private String address;
    private String age;  //构造私有属性，并且不对外公开获取，只能从构造函数设置

    public TestDemo(String name, String address, String age) {
        this.name = name;
        this.address = address;
        this.age = age;
    }

    TestDemo() {

    }

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

}
//使用是这样的
public class Test {

    public static void main(String[] args) {
        TestDemo testDemo = new TestDemo();
        TestDemo testDemo1 = new TestDemo("Hello", "world"， "man");

        String name = testDemo1.getName();
        String address = testDemo1.getAddress();
        //String age = testDemo1.getAge();  //无法获取age
        testDemo1.setName("world");  //调用公开方法进行属性的修改
    }
}
```

## 继承

实现代码的复用，可以基于已经存在的类构造一个新类，继承已存在的类就是复用这个类的方法和域，在此基础上还可以添加一些新的方法和域，一个类只能继承一个类

**缺点**：

1. 父类变，子类就必须变。
2. 继承破坏了封装，对于父类而言，它的实现细节对与子类来说都是透明的。
3. 继承是一种强耦合关系。

访问修饰符：

1. private：仅对本类可见
2. protected：对本包和所有子类可见
3. public：对所有类可见
4. 默认（无修饰）：对本包可见

继承和权限：
| | | | | | | | |
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| | Public | 无修饰 | Private | Protected | final | abstract | static |
| 类继承 | 可继承 | 同一包中的可继承 | 不能修饰类 | 不能修饰类 | 不能派生子类（不可继承） | 一般可继承 | 不能修饰类 |
| 方法重载 | 可重载 | 可重载 | 不能重载 | 可重载 | 不可重载 | 可重载 | 可重载（修饰主函数就不能重载） |
| 成员变量 | 父类属性被隐藏，使用调用super | 父类属性被隐藏，使用调用super | 子类不能直接访问父类的私有变量（通过父类构造器初始化父类私有变量） | 父类属性被隐藏，使用调用super | 必须赋初值 | 不能修饰成员变量 | 每个实例共享这个类变量 |

### 类、超类、子类

引例：一家公司，有经理、普通员工等等，但是所有人都是老板的雇员，以此设计它们之间的关系
分析：不管是经理、普通员工都是雇员的一类，但是各种身份又有一些不同的属性、不同的功能

- 定义父类（超类） Employee

```java
public class Employee {
    /**
     * 姓名
     */
    private String name;
    /**
     * 工资
     */
    private double salary;

    public Employee(String name, double salary) {
        this.name = name;
        this.salary = salary;
    }

    public String getName() {
        return name;
    }

    public double getSalary() {
        return salary;
    }
}
```

- 定义子类 Manager

```java
public class Manager extends Employee {
    /**
     * 奖金信息，特有的属性
     */
    private double bonus;

    public Manager(String name, double salary) {
        //Manager的构造器不能访问父类的私有域，所以通过父类的构造器进行初始化
        super(name, salary);
        bonus = 0;
    }

    /**
     * 重写父类的方法
     */
    @Override
    public double getSalary() {
        //调用父类的方法
        double baseSalary = super.getSalary();
        return baseSalary + bonus;
    }

    public void setBonus(double bonus) {
        this.bonus = bonus;
    }
}
```

继承关系：[![继承层次](https://upload-images.jianshu.io/upload_images/4061843-1952b22e7fa34fe4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-1952b22e7fa34fe4.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)继承层次

强制类型转化：

1. 只能在继承层次内进行类型转换
2. 在将超类转换成子类之前，应该使用instance检查

### Object：所有类的超类

Object类是Java中所有类的始祖，如果一个类没有明确指出超类，那么默认超类是Object。

- euqals方法
    用于检测一个对象是否等于另一个对象，判断两个对象是都具有相同的引用

    ```java
    public boolean equals(Object obj) {
        return (this == obj);
    }
    ```

    一般equals和==是不一样的，但是在Object中两者是一样的

- toString

    ```java
    public String toString() {
        return getClass().getName() + "@" + Integer.toHexString(hashCode());
    }
    ```

- getClass 获得运行时类型
    `public final native Class<?> getClass();`

- hashCode
    `public native int hashCode();`
    该方法用于哈希查找，可以减少在查找中使用equals的次数，重写了equals方法一般都要重写hashCode方法。这个方法在一些具有哈希功能的Collection中用到。

    一般必须满足obj1.equals(obj2)==true。可以推出obj1.hash- Code()==obj2.hashCode()，但是hashCode相等不一定就满足equals。不过为了提高效率，应该尽量使上面两个条件接近等价。

- notify 唤醒在该对象上等待的某个线程
    `public final native void notify();`

- notifyAll 唤醒在该对象上等待的所有线程
    `public final native void notifyAll();`

- wait(long timeout) 在规定时间内没有获得锁就返回
    `public final native void wait(long timeout) throws InterruptedException;`

- wait(long timeout, int nanos)
    在规定时间内没有获得锁就返回，有一个附加时间

    ```java
    public final void wait(long timeout, int nanos) throws InterruptedException {
        if (timeout < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }
    
        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }
    
        if (nanos > 0) {
            timeout++;
        }
    
        wait(timeout);
    }
    ```

- wait() 一直等待，直到获得锁或者被中断

    ```java
    public final void wait() throws InterruptedException {
        wait(0);
    }
    ```

- finalize 用于释放资源
    `protected void finalize() throws Throwable { }`

- clone
    `protected native Object clone() throws CloneNotSupportedException;`
    实现对象的浅复制，只有实现了Cloneable接口才可以调用该方法。主要是JAVA里除了8种基本类型传参数是值传递，其他的类对象传参数都是引用传递，我们有时候不希望在方法里讲参数改变，这是就需要在类中复写clone方法。

### 抽象类、接口

#### 抽象类

类也可以继承抽象类，抽象类可以继承类，也可以继承抽象类，可以实现接口。

- abstract
    使用abstract class 定义的类都是抽象类，可以含有、也可以不含有抽象方法
    还是上面的例子，不管是经理还是雇员，都是人，所以人可以定义为一个抽象类

    ```java
    //抽象类Person
    abstract class Person {
        public abstract String getDescription();
    
        private String name;
    
        public Person(String name) {
            this.name = name;
        }
    
        public String getName() {
            return name;
        }
    }
    
    //Employee继承Person
    public class Employee extends Person {
        private double salary;
    
        public Employee(String name, double salary) {
            super(name);
            this.salary = salary;
        }
    
        @Override
        public String getDescription() {
            return "a employee";
        }
    
        public double getSalary() {
            return salary;
        }
    }
    
    //定义新的Student类继承Person
    public class Student extends Person {
        private String major;
    
        public Student(String name, String major) {
            super(name);
            this.major = major;
        }
    
        public String getMajor() {
            return major;
        }
    
        @Override
        public String getDescription() {
            return "a student" + major;
        }
    }
    ```

- 注意问题：

    1. 抽象类不能被实例化，实例化的工作应该交由它的子类来完成，它只需要有一个引用即可。
    2. 抽象方法必须由子类来进行重写
    3. 只要包含一个抽象方法的抽象类，该方法必须要定义成抽象类，不管是否还包含有其他方法
    4. 抽象类中可以包含具体的方法，当然也可以不包含抽象方法
    5. 子类中的抽象方法不能与父类的抽象方法同名
    6. abstract不能与final并列修饰同一个类
    7. abstract 不能与private、static、final或native并列修饰同一个方法

#### 接口

类可以实现接口，抽象类也可以实现接口，但是接口只能实现接口。类、抽象类、接口都可以实现多个接口

- 接口示例：
    Comparable接口

    ```java
    public interface Comparable {
        int compareTo(Object other);
    }
    
    //jdk 1.5后使用泛型定义
    public interface Comparable<T> {
        int compareTo(T other);
    }
    ```

    使用 implements关键字

    ```java
    public class Employee extends Person implements Comparable {
        ...
    
        public int compareTo(Object other) {
            Employee other = (Employee) other;  //使用泛型后不用强转了，比较的方式自定义
            return Double.compare(salary, other.salary);
        }
    }
    ```

- 接口的默认方法
    可以为接口提供一个默认实现，必须用default修饰符标记

    ```java
    public interface Comparable {
        default int compareTo(Object other) {
            return 0;
        }
    }
    ```

    默认方法可能导致冲突，例如现在一个接口中将方法定义为默认方法，然后又在超类或另一个接口中定义了同样的方法

- 接口的继承
    一个接口可以继承另一个接口

    ```java
    public interface Work {
        public void work();
    }
    
    public interface Study extends Work {
        public void study();
    }
    
    //实现Study的时候会重写两个方法
    public class Student extends Person implements Study {
        ...
        public void work() {}
    
        public void study() {}
    }
    ```

    Study接口自己声明了一个方法，又继承了Work，所以实现Study接口的时候，一共有两个方法需要重写

- 接口与回调
    参考[一个经典例子让你彻彻底底理解java回调机制](https://blog.csdn.net/xiaanming/article/details/8703708)

- 注意事项：

    1. 一个Interface的所有方法访问权限自动被声明为public。确切的说只能为public，当然你可以显示的声明为protected、private，但是编译会出错！
    2. 接口中可以定义“成员变量”，或者说是不可变的常量，因为接口中的“成员变量”会自动变为为public static final。可以通过类命名直接访问：ImplementClass.name
    3. 接口中不存在实现的方法
    4. 实现接口的非抽象类必须要实现该接口的所有方法。抽象类可以不用实现
    5. 不能使用new操作符实例化一个接口，但可以声明一个接口变量，该变量必须引用（refer to)一个实现该接口的类的对象。可以使用 instanceof 检查一个对象是否实现了某个特定的接口。例如：if(anObject instanceof Comparable){}。
    6. 在实现多接口的时候一定要避免方法名的重复

## 多态

态性是指允许不同子类型的对象对同一消息作出不同的响应。简单的说就是用同样的对象引用调用同样的方法但是做了不同的事情。

例子：利用抽象类的例子写一个测试类

```java
public class Test {
    public static void main(String[] args) {
        Person person = new Employee("Employee", 5000);  //向上转型为Person，person指向Employee
        System.out.println(person.getDescription());  //最终调用的是Employee重写的方法

        Employee employee = (Employee) person;  //向下转型，需要强制转换
        System.out.println(employee.getDescription());  //调用的是Employee重写的方法

        person = new Student("Student", "java");  //向上转型，将person重新指向一个新的Student
        System.out.println(person.getDescription());  //调用Student重写的方法
    }
}

/** 结果
 * a employee
 * a employee
 * a studentjava
 * /
```

Employee是Person的子类， Person person = new Employee(“Employee”, 5000) 编译时变量和运行时变量不一样，所以多态发生了。

> （1）Person person作为一个引用类型数据，存储在JVM栈的本地变量表中。
> （2）new Employee(“Employee”, 5000)作为实例对象数据存储在堆内存中
> Employee的对象实例数据（接口、方法、field、对象类型等）的地址也存储在堆中
> Employee的对象的类型数据（对象实例数据的地址所执行的数据）存储在方法区中，方法区中对象类型数据中有一个指向该类方法的方法表。
>
> （3）Java虚拟机规范中并未对引用类型访问具体对象的方式做规定，目前主流的实现方式主要有两种：
> １. 通过句柄访问
> 在这种方式中，JVM堆中会专门有一块区域用来作为句柄池，存储相关句柄所执行的实例数据地址（包括在堆中地址和在方法区中的地址）。这种实现方法由于用句柄表示地址，因此十分稳定。
> ２.通过直接指针访问
> 通过直接指针访问的方式中，reference中存储的就是对象在堆中的实际地址，在堆中存储的对象信息中包含了在方法区中的相应类型数据。这种方法最大的优势是速度快，在HotSpot虚拟机中用的就是这种方式。
>
> （4）实现过程
> 首先虚拟机通过reference类型（Person的引用）查询java栈中的本地变量表，得到堆中的 对象类型数据的地址，从而找到方法区中的对象类型数据（Employee的对象类型数据），然后查询方法表定位到实际类（Employee类）的方法运行。

### 多态存在的三个必要条件

1. 继承
    如例子中的Employee是Person的子类
2. 重写
    Employee重写了Person的getDescription方法
3. 父类引用指向子类对象
    Person person = new Employee(“Employee”, 5000);

### 多态的优点

1. 消除类型之间的耦合关系
2. 可替换性
3. 可扩充性
4. 接口性
5. 灵活性
6. 简化性

### 多态性

多态性分为编译时的多态性和运行时的多态性。

> 1. 运行时的多态性
>     方法重写（override）实现的是运行时的多态性（也称为后绑定），运行时的多态是面向对象最精髓的东西，要实现多态需要做两件事：1. 方法重写（子类继承父类并重写父类中已有的或抽象的方法）；2. 对象造型（用父类型引用引用子类型对象，这样同样的引用调用同样的方法就会根据子类对象的不同而表现出不同的行为）。
> 2. 编译时的多态
>     方法重载（overload）实现的是编译时的多态性（也称为前绑定）。

- 方法调用绑定
    将一个方法主体关联起来被称作绑定
    （1）前期绑定（静态绑定）：在程序执行之前进行绑定（默认的绑定方式）
    （2）后期绑定（动态绑定或运行时绑定）：在运行时根据对象的类型进行绑定

    Java中除了static方法和final方法（private方法属于final方法）之外，其他所有的方法都是后期绑定

- 方法的型构：指方法的组成结构，具体包括方法的名称和参数，涵盖参数的数量、类型以及出现的顺序，但是不包括方法的返回值类型，访问权限修饰符，以及 abstract、static、final 等修饰符。

    1. 相同型构

        ```java
        public void method(String s) {}
        
        public String method(String s) {}
        ```

    2. 不同型构

        ```java
        public void method(int i, String s) {}
        
        public void method(String s, int i) []
        ```

- 重载
    指在同一个类中定义了一个以上具有相同名称，但是型构不同的方法。

- 重写
    指在继承情况下，子类中定义了与其父类中方法具有相同型构的新方法，就称为子类把父类的方法重写了（子类必须重写父类为抽象类的方法）。这是实现多态必须的步骤

## 相关面试问题

1. 父类的静态方法能否被子类重写
    不能，声明为static的方法不能被重写，但是能够被再次声明
    用static修饰的方法不依赖于任何对象就可以进行访问（就是说你不需要实例化，就可以通过类名访问），在JVM中，这两个静态方法的存储空间是两个，即使没有对象，它也存在。如果试图在子类重写父类的静态方法，编译器不会报错，但是并不会得到相应的结果，当你用父类引用调用的仍是父类的方法，new一个子类调用时，调用的就是子类的静态方法。也就是说你试图重写父类的静态方法和父类的静态方法完全就是两个方法。重写仅对非静态方法有用

    小提示：JVM中的静态分派（重载）和动态分派（重写）

2. 静态属性和静态方法是否可以被继承？是否可以被重写？以及原因？
    静态属性和静态方法可以，重写不可以
    [java中静态属性和和静态方法的继承问题 以及多态的实质](http://www.cnblogs.com/kabi/p/5181941.html)

3. Java中实现多态的机制是什么
    继承、重写、父类引用指向子类对象

4. 在Java中，什么时候用重载，什么时候用重写？
    子类继承父类，重写父类方法
    方法重载，不同的参数，同样的方法名

5. Override和Overload的含义与区别。
    Override是重写，子类重写父类的方法
    Overload是重载，方法重载

    | 区别点   | 重载方法 | 重写方法                                       |
    | -------- | -------- | ---------------------------------------------- |
    | 参数列表 | 必须修改 | 一定不能修改                                   |
    | 异常     | 可以修改 | 可以减少或删除，一定不能抛出新的或者更广的异常 |
    | 访问     | 可以修改 | 一定不能做更严格的限制（可以降低限制）         |
    | 返回类型 | 可以修改 | 一定不能修改                                   |

6. Java可以多继承吗
    不可以，最多继承一个类，但是可以实现多个接口

7. object类的equal 和hashcode 方法重写，为什么？
    hashcode用于哈希查找，可以减少在查找中使用equals的次数，所以通常两个一起重写，以便用户可以将对象插入散列表中

8. object有哪些公用方法
    euqals、toString、hashCode、getClass、wait、notify、notifyAll、

9. 抽象类与接口的区别；应用场景；抽象类是否可以没有方法和属性
    （1）只能继承一个抽象类，但是可以实现多个接口。
    （2）在abstract class 中可以有自己的数据成员，也可以有非abstarct的方法，而在interface中，只能够有静态的不能被修改的数据成员（也就是必须是static final的，不过在 interface中一般不定义数据成员），所有的方法都是public abstract的。
    （3）抽象类中的变量默认是 friendly 型，其值可以在子类中重新定义，也可以重新赋值。接口中定义的变量默认是public static final 型，且必须给其赋初值，所以实现类中不能重新定义，也不能改变其值。
    （4）abstract class和interface所反映出的设计理念不同。其实abstract class表示的是”is-a”关系，interface表示的是”like-a”关系，门和报警的关系。

    （5）现抽象类和接口的类必须实现其中的所有方法。抽象类中可以有非抽象方法。接口中则不能有实现方法

    应用场景：当你关注一个事物的本质的时候，用抽象类；当你关注一个操作的时候，用接口。

    可以没有方法和属性

10. abstract 的方法是否可同时是 static、是否可同时是 native、是否可同时是 synchronized 的？为什么？
    都不可以
    用static声明方法表明这个方法在不生成类的实例时可直接被类调用，而abstract方法不能被调用，两者矛盾。
    因为native 暗示这些方法是有实现体的，只不过这些实现体是非java 的，但是abstract却显然的指明这些方法无实现体。
    从synchronized的功能也可以看出，用synchronized的前提是该方法可以被直接调用，显然和abstract连用

11. 抽象关键字不可以和哪些关键字共存？
    abstract 不能与private、static、final或native并列修饰同一个方法

12. 接口与回调；回调的原理；写一个回调demo
    参考[一个经典例子让你彻彻底底理解java回调机制](https://blog.csdn.net/xiaanming/article/details/8703708)

13. 接口的意义
    1、重要性：在Java语言中， abstract class 和interface 是支持抽象类定义的两种机制。正是由于这两种机制的存在，才赋予了Java强大的 面向对象能力。

    2、简单、规范性：如果一个项目比较庞大，那么就需要一个能理清所有业务的架构师来定义一些主要的接口，这些接口不仅告诉开发人员你需要实现那些业务，而且也将命名规范限制住了（防止一些开发人员随便命名导致别的程序员无法看明白）。

    3、维护、拓展性：比如你要做一个画板程序，其中里面有一个面板类，主要负责绘画功能，然后你就这样定义了这个类，可是在不久将来，你突然发现这个类满足不了你了，然后你又要重新设计这个类，更糟糕是你可能要放弃这个类，那么其他地方可能有引用他，这样修改起来很麻烦，如果你一开始定义一个接口，把绘制功能放在接口里，然后定义类时实现这个接口，然后你只要用这个接口去引用实现它的类就行了，以后要换的话只不过是引用另一个类而已，这样就达到维护、拓展的方便性。

    4、安全、严密性：接口是实现软件松耦合的重要手段，它描叙了系统对外的所有服务，而不涉及任何具体的实现细节。这样就比较安全、严密一些(一般软件服务商考虑的比较多)。

14. 权限修饰符protected和默认的区别？proctected 修饰的方法，假如子类和父类不在一个包下，子类可以访问父类中这个方法吗？
    protected不能用于修饰类，默认可以修饰类，修饰的类同一包中可继承。修饰方法和变量时差不错

    可以，是子类或者同一个包就可以

15. 父类的方法是public，子类重写后，改为protect，会不会报错？反过来呢？
    会
    反过来不会

    父类中声明为 public 的方法在子类中也必须为 public。

    父类中声明为 protected 的方法在子类中要么声明为 protected，要么声明为 public，不能声明为 private

16. 静态绑定和动态绑定的区别
    （1）静态绑定发生在编译时期，动态绑定发生在运行时
    （2）使用private或static或final修饰的变量或者方法，使用静态绑定。而虚方法（可以被子类重写的方法）则会根据运行时的对象进行动态绑定。
    （3）静态绑定使用类信息来完成，而动态绑定则需要使用对象信息来完成。
    （4）重载(Overload)的方法使用静态绑定完成，而重写(Override)的方法则使用动态绑定完成
    （5）真实的对象不能用于静态绑定，相反，类型信息比如变量的类型可以用于本地的方法。另一方面，动态绑定用一个真是的对象去找到那个真正的方法的