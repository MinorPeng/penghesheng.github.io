---
title: Java 泛型总结
tag: Java
---

# 泛型总结

**声明**：部分面试题来自微信公众号**码农每日一题**

Java 5开始引进，实现**参数化类型**的概念，是Java中的一种语法糖

## 简单泛型

- 创建

    ```java
    public class Holder<T> {
        private T t;
    
        public Holder(T t) {
            this.t = t;
        }
    
        public void set(T t) {
            this.t = t;
        }
    
        public T get() {
            return t;
        }
    
        public static void main(String[] args) {
            Holder<String> holder = new Holder<String>("abc");
            String str = holder.get();
        }
    }
    ```

- 元组
    将一组对象直接打包存储于其中的一个单一对象。这个容器对象允许读取其中的元素，但是不允许存放新的对象（这个概念也称为数据传递对象或信使）

    创建一个二维元组：

    ```java
    public class TwoTuple<A, B> {
        public final A first;
        public final B second;
        //有次序
        public TwoTuple(A a, B b) {
            this.first = a;
            this.second = b;
        }
    
        public String toString () {
            return "(" + first + ", " + second + ")";
        }
    }
    ```

    在之前的基础上创建一个三维元组：

    ```java
    public class ThreeTuple<A, B, C> extends TwoTuple<A, B> {
        public final C third;
    
        public ThreeTuple(A a, B b, C c) {
            super(a, b);
            third = c;
        }
    
        public String toString() {
            return "(" + first + ", " + second + ", " + third + ")";
        }
    }
    ```

- 泛型实现堆栈

    ```java
    public class LinkedStack<T> {
        private static class Node<U> {
            U item;
            Node<U> next;
    
            Node() {
                item = null;
                next = null;
            }
    
            Node(U item, Node<u> next) {
                this.item = item;
                this.next = next;
            }
    
            boolean end() {
                return (item == null && next == null);
            }
        }
    
        private Node<T> top = new Node<T>();
    
        public void push(T item) {
            top = new Node<T>(item, top);
        }
    
        public T pop() {
            T result = top.item;
            if (!top.end()) {
                top = top.next;
            }
            return result;
        }
    }
    ```

## 泛型接口

generator，生成器，专门负责创建对象的类

```java
public interface Generator<T> {
    T next();
}
```

## 泛型方法

- 参数固定

    ```java
    public class GenericMethods {
        public <T> void f(T x) {
        }
    }
    ```

- 可变参数类型

    ```java
    public class GenericVarargs {
        public static <T> List<T> makeList(T... args) {
            List<T> result = new ArrayList<T>();
            for (T item : args) {
                result.add(item);
            }
            return result;
        }
    }
    ```

- 简化元组的使用
    通过重载static方法创建元组

## 类型擦除

无论何时定义一个泛型类型，都自动提供了一个相应的原始类型。原始类型的名字就是删除类型参数后的泛型类型名。擦除类型变量，并替换为限定类型（无限定时用Object）

引例：C#中泛型在所有过程（程序源码、编译后的IL、运行期的CLR）中都切实存在，List\<int>和List\<String>就是两个不同的类型，它们在系统运行期生成，有自己的虚方法表和类型数据，这种实现称为类型膨胀，属于真泛型
java中的泛型只在源码中存在，在编译后的字节码文件中，会替换为原来的原生类型（裸类型），并且在相应的地方插入强制类型转换。所以ArrayList\<int>和ArrayList\<String>就是同一个类，这种实现称为类型擦除，属于伪泛型

例如：

```java
//泛型擦除前
Map<String, String> map = new HashMap<String, String>();
map.put("hello", "你好");
map.put("How are you", "最近怎么样");
System.out.println(map.get("hello"));
System.out.println(map.get("How are you"));

//编译后。类型擦除后
Map<String, String> map = new HashMap<String, String>();
map.put("hello", "你好");
map.put("How are you", "最近怎么样");
System.out.println((String) map.get("hello"));
System.out.println((String) map.get("How are you"));
```

在map根据键获取值的时候，编译后会自动加上String进行强转，也就是底层的Java并没有实现真正的泛型

再比如：

```java
public class Test {
    public static void method(List<String> list) {

    }

    public static void method(List<Integer> list) {

    }
}
```

这个方法重载是错的，不能被编译，通过Idea，你会发现会提示你List\<String>和List\<Integer>两个参数是一个类，所以不能进行重载。在JVM中说到当两个方法拥有不同的返回值后，便是可以编译的（java 6），但是我在Java 8中还是不能编译。

- 迁移兼容性
    在基于擦除的实现中，泛型类型被当作第二类类型处理，即不能在某些重要的上下文环境中使用的类型。只有在静态类型检查期间才会出现，在此之后，所有的泛型类型都会被擦除，替换为它们的非泛型上界。

- 擦除的问题
    泛型不能用于显示地引用运行时类型的操作中，例如转型、instanceof操作、new表达式。因为所有关于参数的类型信息都丢失了。

    1. 泛型类并没有自己独有的Class类对象。
        比如并不存在List\<String>.class或是List\<Integer>.class，而只有List.class。

    2. 静态变量是被泛型类的所有实例所共享的。
        对于声明为MyClass\<T>的类，访问其中的静态变量的方法仍然是 MyClass.myStaticVar。>不管是通过new MyClass\<String>还是new MyClass\<Intege\r>创建的对象，都是共享一个静态变量。

    3. 泛型的类型参数不能用在Java异常处理的catch语句中。（不能抛出或捕获泛型类的实例）
        因为异常处理是由JVM在运行时刻来进行的。由于类型信息被擦除，JVM是无法区分两个异常类型MyException\<String>和MyException\<Integer>的。对于JVM来说，它们都是 MyException类型的。也就无法执行与异常对应的catch语句。

        ```java
        //不能扩展Throwable
        public class Problem<T> entends Exception {}  //Error: can't extend Throwable
        
        //catch语句不能使用类型变量
        public static <T extends Throwable> void doWork(Class<T> t) {
            try {
                //do work
            } catch(T e) {
                //Error: can't catch type variable
            }
        }
        ```

        但是在异常规范中使用类型变量是允许的 

        ```java
        public static <T extends Throwable> void doWork(T t) throws T {
            try {
                //do work
            } catch(Throwable realCause) {
                t.initCause(realCause);
                throw t;
            }
        }	
        ```


4. 泛型不能用于显示地引用运行时类型的操作之中，例如转型、instanceof操作、new表达式  

5. 不能用基本类型实例化类型参数  
    没有Pair\<double\>，只有Pair\<Double\>，因为擦除后，Pair类含有Object类型的域，而Object不能存储double值  

6. 运行时类型查询只适用于原始类型  
    Java虚拟机中总有一个特定的非泛型类型  

7. 不能创建参数化类型的数组（但是可以声明参数化类型的数组）  
    因为擦除后，类型进行了强转  
    `Pair<String>[] table = new Pair<String>[10];`  
    擦除后，table的类型是Pair[]，可以把它转为Object[]，  
    `Object[] objarray = table;`  
    数组会记住它的元素类型，如果试图存储其他类型，就会抛出一个ArrayStoreException异常  
    `objarray[0] = "hello";  //Error`

8. 不能实例化类型变量  
    Java 8之后，最好的方法是让调用者提供一个构造器表达式  
    Pair<String> p = Pair.makePair(String::new);

//makePair方法接收一个Supplier<T>，这是一个函数式接口
public static <T> Pair<T> makePair(Supplier<T> constr) {
    return new Pair<>(constr.get(), constr.get());
}


​    或者传统的方法通过反射调用

9. 不能构造泛型数组  
    数组会填充null值，构造时看上去时安全的。但是数组本身也有类型，用来监控存储在虚拟机中的数组。这个类型会被擦除。  
    例如：  
    public static <T extends Comparable> T[] minmax(T[] a) {
    T[] mm = new T[2];  //Error
}


​    类型擦除会让这个方法永远构造Comparable[2]数组  

10. 类型擦除与多态的冲突  

11. 类型擦除后的冲突  
    无法创建引发冲突的条件 
    
    ```java
    public class Pair<T> {
    public boolean equals(T value) {
        return first.equals(value) && second.equals(value);
}
    ...
    }
    ```


​    擦除后，boolean equals(T)就是boolean equals(Object)，与Object.equals方法冲突

12. 泛型中参数化类型无法支持继承关系（但可以扩展或实现其他的泛型类）  
    ArrayList\<T\>类实现List\<T\>接口

- 泛型其他注意事项

1. 泛型类的静态上下文中类型变量无效

    ```java
    public class Singleton<T> {
        private static T singleInstance;  //Error
        //Error
        public static T getSingleInstance() {
            if (singleInstance == null) {
                //constuct new instance of T
            }
            return singleInstance;
        }
    }
    ```

    禁止使用带有类型变量的静态域和方法

2. 可以消除对受查异常的检查

## 擦除的补偿

```java
public class Erased<T> {
    private final int SIZE = 100;
    public static void f(Object arg) {
        if (arg instanceof T) {}  //报错，泛型不能使用instanceof操作
        T var = new T();  //报错，泛型不能new操作
        T[] array = new T[SIZE];  //报错，不能创建泛型数组
        T[] array1 = (T) new Object[SIZE];  //警告，泛型可以通过引入类型标签来对擦除进行补偿
    }
}
```

对于new T()，泛型的实例化无法实现的原因，一部分因为擦除，另一部分是因为编译器不能验证T具有默认（无参）构造器。

**注意**：在C++中，泛型是可以进行实例化的，并且是安全的，也就是可以T var = new T()这个操作的。

- 如何创建类型实例

    传递一个工厂对象

    1. 使用显示的工厂对象
    2. 模板方法设计模式

## 泛型数组

- 如何创建一个泛型数组？

    1. 使用ArrayList（类型安全）

    2. 按照编译器喜欢的方式定义一个引用（可以编译但是不能运行）

        ```java
        class Generic<T> {}
        
        public class ArrayOfGenericReference {
            static Generic<Integer>[] gia;
        }
        ```

        通过类型强转，需要注意的是运行时仍然会产生异常

        ```java
        public class GenericArray<T> {
            private Object[] array;
        
            public GenericArray(int sz) {
                array = new Object[sz];
            }
        
            public void put(int index, T item) {
                array[index] = item;
            }
        
            @SuppressWarnings("unchecked")
            public T get(int index) {
                return (T) array[index];
            }
        
            @SuppressWarnings("unchecked")
            public T[] rep() {
                return (T[]) array;  //Waring: unchecked cast
            }
        
            public static void main(String[] args) {
                GenericArray<Integer> gai = new GenericArray<>(10);
                for (int i = 0; i < 10; i++) {
                    gai.put(i, i);
                }
                for (int i = 0; i < 10; i++) {
                    System.out.print(gai.get(i)+" ");
                }
                System.out.println();
                try {
                    Integer[] ia = gai.rep();
                } catch (Exception e) {
                    System.out.println(e);
                }
            }
        }
        /**
         * 结果
         * 0 1 2 3 4 5 6 7 8 9
         * java.lang.ClassCastException: [Ljava.lang.Object; cannot be cast to              [Ljava.lang.Integer;
         */
        ```

    3. 提供一个数组构造表达式

        ```java
        //构造器表达式指示一个函数，给定所需的长度，会构造一个指定长度的String数组
        String[] ss = ArrayAlg.minmax(String[]::new, "Tom", "Dark", "Hary");
        
        public static <T extends Comparable> T[] minmax(IntFunction<T[]> constr, T... a) {
            T[] mm = constr.apply(2);
        }
        ```

    4. 老式的方法：通过反射

        ```java
        public static <T extends Comparable> T[] minmax(T... a) {
            T[] mm = (T[]) Array.newInstance(a.getClass().getComponentType(), 2);
        }
        ```

## 通配符

- 上边界限定通配符：<? extends Fruit>
    先了解一些数组的一种特殊行为：可以向导出类型的数组赋予基类型的数组引用

    ```java
    class Fruit {}
    class Apple extends Fruit {}
    class Jonathan extends Apple {}
    class Orange extends Fruit {}
    
    public class CovariantArrays {
        public static void main(String[] args) {
            Fruit fruit = new Apple[10];
            fruit[0] = new Apple();  //OK
            fruit[2] = new Jonathan();  //OK
            //Runtime type is Apple[], not fruit[] or Orange[]:
            try {
                //Complier allows you to add Fruit
                fruit[0] = new Fruit[];  //ArrayStoreException
            } catch(Exception e) {
                System.out.println(e);
            }
            try {
                //Complier allows you to add Oranges
                fruit[0] = new Orange();  //ArrayStoreException
            } catch(Exception e) {
                System.out.println(e);
            }
        }
    }
    
    /**
     * 输出：
     * java.lang.ArrayStoreException: Fruit
     * java.lang.ArrayStoreException: Orange
     */
    ```

    mian()中创建了一个**Apple**数组，并将其赋值给一个**Fruit**数组引用，这是有意义的，因为**Apple**也是一种**Fruit**，因此**Apple**数组也应该是一种**Fruit**数组。
    编译器允许你将**Fruit**放置到这个**Apple**数组（放入Fruit和Orange，向上转型），因为对编译器来说时有意义的，但是运行时的数组机制知道处理的实际上时**Apple[]**（所以只能在其中放置Apple或Apple的子类），因此会在向数组中放置异构类型时抛出异常

    泛型的目的之一就是将这种错误检测移入到编译期，看看使用泛型容器会怎么样？

    ```java
    public class NonCovariantGenerics {
        List<Fruit> flist = new ArrayList<Apple>();
    }
    ```

    无法编译，实际上这不是向上转型——**Apple**的**List**不是**Fruit**的**List**，**Apple**的**List**持有Apple和Apple的子类，**Fruit**的**List**持有Fruit和Fruit的子类，尽管包含了Apple，但它终究不是**Apple**的**List**，它仍旧是**Fruit**的**List**。类型上两者并不等价，尽管Apple是Fruit的子类。

    **接下来进入正题，使用通配符**：

    ```java
    public class GenericsAndCovariance {
        public static void main(String[] args) {
            List<? entends Fruit> flist = new ArrayList<Apple>();
            // Compile Error: can’t add any type of object:
            // flist.add(new Apple());  //这种向上转型，会丢失向其中传递任何对象的能力（包括Object）
            // flist.add(new Fruit());
            // flist.add(new Object());
            flist.add(null); // Legal but uninteresting  没有意义
            // We know that it returns at least Fruit:
            Fruit f = flist.get(0);  //在这个 List 中，不管它实际的类型到底是什么，但肯定能转型为 Fruit，所以编译器允许返回 Fruit。
        }
    }
    ```

    flist现在的类型是List<? extends Fruit>，可以理解为继承Fruit的类型的列表（通俗点就是继承了Fruit的类的List），但实际上并不意味着这个List将持有任何类型的Fruit。通配符引用的是明确的类型，因此它意味着“某种flist引用没有指定的具体类型”。因此这个被赋值的List必须持有Fruit或Apple这样的某种指定类型。

    了解了通配符的作用和限制后，好像任何接受参数的方法我们都不能调用了。其实倒也不是，看下面的例子：

    ```java
    public class CompilerIntelligence {
        public static void main(String[] args) {
            List<? extends Fruit> flist =
            Arrays.asList(new Apple());
            Apple a = (Apple)flist.get(0); // No warning
            flist.contains(new Apple()); // Argument is ‘Object’
            flist.indexOf(new Apple()); // Argument is ‘Object’
            //flist.add(new Apple());   无法编译
        }
    }
    ```

    contains 和 indexOf 方法正常执行，add方法无法编译。通过ArrayList的源码发现，add()接收的是一个泛型类作为参数，contains 和 indexOf 方法接收的是一个Object类型的参数。因此当你指定一个ArrayList<? extends Fruit>时，add()的参数就变成了“? extends Fruit”，而contains 和 indexOf 方法两个的参数仍是Object类型，不涉及任何通配符。

    为了在类型中使用了通配符的情况下禁止这类的调用，需要在类型参数列表中使用类型参数

    ```java
    public class Holder<T> {
        private T value;
        public Holder() {}
        public Holder(T val) {
            value = val;
        }
        public void set(T val) {
            value = val;
        }
        public T get() {
            return value;
        }
        public boolean equals(Object obj) {
            return value.equals(obj);
        }
    
        public static void main(String[] args) {
            Holder<Apple> Apple = new Holder<Apple>(new Apple());
            Apple d = Apple.get();
            Apple.set(d);
            // Holder<Fruit> Fruit = Apple; // Cannot upcast 有了边界，无法向上转型
            Holder<? extends Fruit> fruit = Apple; // OK
            Fruit p = fruit.get();
            d = (Apple) fruit.get(); // Returns ‘Object’
            try {
                Orange c = (Orange) fruit.get(); // No warning
            } catch(Exception e) {
                System.out.println(e);
            }
            // fruit.set(new Apple()); // Cannot call set()  编译器无法验证其类型安全
            // fruit.set(new Fruit()); // Cannot call set()
            System.out.println(fruit.equals(d)); // OK  接受的参数时Object类型，编译器可以识别
        }
    }
    /** Output: (Sample)
     *java.lang.ClassCastException: Apple cannot be cast to Orange
     *true
     */
    ```

- 超类型通配符：<? super MyClass>
    使用超类通配符就可以安全的传递类型对象，声明通配符是由某个特定类的任何基类来界定

    ```java
    public class SuperTypeWildcards {
        static void writeTo(List<? super Apple> apples) {
            apples.add(new Apple());
            apples.add(new Jonathan());
            // apples.add(new Fruit()); // Error
        }
    }
    ```

    writeTo 方法的参数 apples 的类型是 List<? super Apple>，它表示某种类型的 List，这个类型是 Apple 的基类型。也就是说，我们不知道实际类型是什么，但是这个类型肯定是 Apple 的父类型。因此，我们可以知道向这个 List 添加一个 Apple 或者其子类型的对象是安全的，这些对象都可以向上转型为 Apple。Apple是下届，不能添加Fruit（违反静态类型安全）。

- 无界通配符：<?>
    意味着“任何事物”，很多情况可以认为是一种装饰。

**总结**：写入类型参数，使用超类型通配符，读取类型参数，使用上边界限定通配符

## 泛型和反射

- 泛型Class类
    现在Class类是泛型的，例如String.class实际上是一个Class\<String>类的对象（唯一的对象）

    ```java
    //返回一个无参数构造器构造的一个新实例。它的返回类型被声明为T，其类型与Class<T>描述的类相同，这样避免了类型转换
    T newInstance()
    
    //如果obj为null或有可能转换成类型T，则返回obj，否则抛出BadCastException异常
    T cast(Object obj)
    
    //如果T是枚举类型，则返回所有值组成的数组，否则返回null
    T[] getEnumConstants()
    
    //返回这个类的超类，如果T不是一个类或Object类，则返回null
    Class<? super T> getSuperClass()
    
    //获得公有的构造器或带有给定参数类型的构造器
    Constructor<T> getConstructor(Class... parameterTypes)
    Constructor<T> getDeclaredConstructor(Class... parameterTypes)
    
    //返回用指定类型构造的新实例
    T newInstance(Object... parameters)
    ```

- 虚拟机中的泛型类型信息
    擦除的类型仍然保留了一些泛型祖先的微弱记忆。
    `public static Comparable min(Comparable[] a)`是`public static <T extends Comparable<? super T>> T min(T[] a)`的泛型方法擦除后。但是可以使用反射API来知道以下信息：

    1. 这个泛型方法有一个叫做T的类型参数

    2. 这个类型参数有一个子类型限定，其自身又是一个泛型类型

    3. 这个限定类型有一个通配符参数

    4. 这个通配符参数有一个超类型限定

    5. 这个泛型方法有一个泛型数组参数

        换句话说，知道需要重新构造实现者声明的泛型类型以及方法中的所有内容，只是不知道对于特定的对象或方法调用，以及如何解释类型参数

## 面试题

1. 泛型原理，举例说明；

    ```java
    public class Holder<T> {
        private T t;
    
        public Holder(T t) {
            this.t = t;
        }
    
        public void set(T t) {
            this.t = t;
        }
    
        public T get() {
            return t;
        }
    
        public static void main(String[] args) {
            Holder<String> holder = new Holder<String>("abc");
            String str = holder.get();
        }
    }
    ```

    创建的时候指明Holder持有什么类型的对象，置于<>中，获取的时候会自动的就是正确的类型

2. 什么是泛型中的限定通配符和非限定通配符
    限定通配符有上界限定通配符<? entends T>和超类型限定通配符（下届限定通配符）<? super T>
    非限定通配符是指\<T>，表示可以用任意泛型类型来替代，可以在某种意义上来说是泛型向上转型的语法格式

3. 简单讲一下泛型、泛型有什么作用、运行期可以确定类型吗？
    Java 5开始，“泛型” 意味着编写的代码可以被不同类型的对象所重用。
    作用：类型的参数化，就是可以把类型像方法的参数那样传递，实现了参数化类型的概念，使代码可以应用与多种类型

    运行期可以确定类型，因为Java中的泛型只存在在源代码中，在编译后，类型都会强转为相应的类型

4. Java泛型了解吗，知道它的运行机制吗？
    通过类型擦除实现，Java中的泛型基本上都是在编译器这个层次来实现的，在生成的Java字节代码中是不包含泛型中的类型信息的。使用泛型的时候加上的类型参数，会被编译器在编译的时候去掉，这个过程就称为类型擦除。如在代码中定义的List’<Object>和List\<String>等类型，在编译之后都会变成List。JVM看到的只是List，而由泛型附加的类型信息对JVM来说是不可见的。Java编译器会在编译时尽可能的发现可能出错的地方，但是仍然无法避免在运行时刻出现类型转换异常的情况。

5. 泛型常用特点，List\<String>能否转为List\<Object>
    E - Element (在集合中使用，因为集合中存放的是元素)
    T - Type（Java 类）
    K - Key（键）
    V - Value（值）
    N - Number（数值类型）
    ？ - 表示不确定的java类型

    一个方法如果接收List\<Object>作为形式参数，那么如果尝试将一个List\<String>的对象作为实际参数传进去，却发现无法通过编译。虽然从直觉上来说，Object是String的父类，这种类型转换应该是合理的。但是实际上这会产生隐含的类型转换问题，因此编译器直接就禁止这样的行为。

6. 需要泛型之间的类型转换怎么做？通配符？与PECS
    （1）如果你是想遍历collection，并对每一项元素操作时，此时这个集合时生产者（生产元素），应该使用 Collection<? extends E>，因为相对于你泛型E，能放进E中的（把Collection中的元素放入E类型中） ，应该是E的子类型。

    （2）如果你是想添加元素到collection中去，那么此时集合时消费者（消费元素）应该使用Collection<? super E>，同理，你要将E类的元素放入Collection中去，那么Collection应该存放的是E的父类型。

7. 泛型的检测：不符合泛型T的检测
    在进行编译之前就对所有泛型进行检测，加入类型检测和转换的指令，比如返回泛型的结果实际上返回的是擦出后的类型，而虚拟机会多加一个类型转换的指令

8. Java 如何优雅的实现元组？
    创建一个二维元组：

    ```java
    public class TwoTuple<A, B> {
        public final A first;
        public final B second;
        //有次序
        public TwoTuple(A a, B b) {
            this.first = a;
            this.second = b;
        }
    
        public String toString () {
            return "(" + first + ", " + second + ")";
        }
    }
    ```

9. 为什么 Java 泛型要通过擦除来实现？擦除有什么坏处或者说代价？
    擦除主要的正当理由是从非泛化代码到泛化代码的转变过程，以及在不破坏现有类库的情况下，将泛型融入Java语言，为了兼容性考虑（废话一大堆，就是为了能使泛型封号的融入Java，不能影响以前的代码）

    （1）泛型不能用于显示地引用运行时类型的操作之中，例如转型、instanceof操作、new表达式
    （2）擦除和兼容性导致了使用泛型并不是强制的（如 List\<String> list = new ArrayList(); 等写法）
    （3）导致编码谨慎，考虑边界操作

10. 请比较深入的谈谈你对 Java 泛型擦除的理解和带来的问题认识？
    参考[java 泛型基础问题汇总](https://www.cnblogs.com/huansky/p/8043149.html)
    类型擦除的基本过程也比较简单，在编译成字节码时，首先是进行类型检查，然后找到用来替换类型参数的具体类。这个具体类一般是Object。如果指定了类型参数的上界的话，则使用这个上界。把代码中的类型参数都替换成具体的类。同时去掉出现的类型声明，即去掉\<>的内容。比如T get()方法声明就变成了Object get()；List\<String>就变成了List。接下来就可能需要生成一些桥接方法（bridge method）。这是由于擦除了类型之后的类可能缺少某些必须的方法。
    比如：

    ```java
    public class MyString implements Comparable<string> {
        public int compareTo(String str) {
            return 0;
        }
    }
    ```

    当类型信息被擦除之后，上述类的声明变成了class MyString implements Comparable。但是这样的话，类MyString就会有编译错误，因为没有实现接口Comparable声明的int compareTo(Object)方法。这个时候就由编译器来动态生成这个方法

    问题：
    （1）泛型类并没有自己独有的Class类对象。

    ```java
    比如并不存在List\<String\>.class或是List\<Integer\>.class，而只有List.class。
    ```

    （2）静态变量是被泛型类的所有实例所共享的。

    ```java
    对于声明为MyClass\<T\>的类，访问其中的静态变量的方法仍然是 MyClass.myStaticVar。不管是通过new MyClass\<String\>还是new MyClass\<Intege\r>创建的对象，都是共享一个静态变量。  
    ```

    （3）泛型的类型参数不能用在Java异常处理的catch语句中。

    ```java
    因为异常处理是由JVM在运行时刻来进行的。由于类型信息被擦除，JVM是无法区分两个异常类型MyException\<String\>和MyException\<Integer\>的。对于JVM来说，它们都是 MyException类型的。也就无法执行与异常对应的catch语句。  
    ```

    （4）泛型不能用于显示地引用运行时类型的操作之中，例如转型、instanceof操作、new表达式
    （5）不能用基本类型实例化类型参数
    （6）运行时类型查询只适用于原始类型
    （7）不能创建参数化类型的数组
    （8）不能实例化类型变量
    （9）不能构造泛型数组
    （10）类型擦除与多态的冲突
    （11）类型擦除后的冲突
    （12）泛型中参数化类型无法支持继承关系
    （13）为兼容性带来了问题

11. 为什么 Java 的泛型数组不能采用具体的泛型类型进行初始化？
    因为泛型只存在源代码中，编译后，泛型擦除，就会被强转为Object[]，数组会记住它的元素类型，试图存储其他类型就会抛出ArraySotreException异常

    ```java
    List<String>[] lsa = new List<String>[10]; // Not really allowed.
    Object o = lsa;
    Object[] oa = (Object[]) o;
    List<Integer> li = new ArrayList<Integer>();
    li.add(new Integer(3));
    oa[1] = li; // Unsound, but passes run time store check
    String s = lsa[1].get(0); // Run-time error: ClassCastException.
    ```

    由于 JVM 泛型的擦除机制，所以上面代码可以给 oa[1] 赋值为 ArrayList 也不会出现异常，但是在取出数据的时候却要做一次类型转换，所以就会出现 ClassCastException，如果可以进行泛型数组的声明则上面说的这种情况在编译期不会出现任何警告和错误，只有在运行时才会出错，但是泛型的出现就是为了消灭 ClassCastException，所以如果 Java 支持泛型数组初始化操作就是搬起石头砸自己的脚。

12. 如何正确的初始化泛型数组实例？
    （1）通过声明通配类型的数组，然后进行类型转换（不是很安全）

    ```java
    GenricArray<String>[] array = (GenericArray<String>[]) new GenericArray<?>[10];
    ```

    （2）通过ArrayList来初始化，这是最安全的方法

    ```java
    private List<T> list = new ArrayList<T>();
    ```

    （3）通过反射

    ```java
    public static <T extends Comparable> T[] minmax(T... a) {
        T[] mm = (T[]) Array.newInstance(a.getClass().getComponentType(), 2);
    }
    ```

    （4）提供一个数组构造器表达式

    ```java
    public static <T extends Comparable> T[] minmax(IntFunction<T[]> constr, T... a) {
        T[] mm = constr.apply(2);
    }
    ```

13. Java 泛型对象能实例化 T t = new T() 吗，为什么？
    不能，无法编译，一部分因为擦除，另一部分是因为编译器不能验证T具有默认（无参）构造器。擦除的原因是编译器在编译的时候需要获得T类型，但是泛型只存在源代码中，编译时就会被擦除，而此时编译器又无法将之转换为一个确定的类型，所以无法编译。

    Java 8之后，最好的方法是让调用者提供一个构造器表达式

    ```java
    Pair<String> p = Pair.makePair(String::new);
    
    //makePair方法接收一个Supplier<T>，这是一个函数式接口
    public static <T> Pair<T> makePair(Supplier<T> constr) {
        return new Pair<>(constr.get(), constr.get());
    }
    ```

    或者传统的方法通过反射调用

14. 泛型中extends和super的区别
    \<? extends T>上边界限定通配符，一般用于读取类型参数
    \<? super T>超类型通配符，一般用于写入类型参数

参考[Java 泛型总结（三）：通配符的使用](https://segmentfault.com/a/1190000005337789)
[Java 泛型总结（二）：泛型与数组](https://segmentfault.com/a/1190000005179147)
[Java 泛型总结（一）：基本用法与类型擦除](https://segmentfault.com/a/1190000005179142)
[java泛型（二）、泛型的内部原理：类型擦除以及类型擦除带来的问题](https://blog.csdn.net/lonelyroamer/article/details/7868820)