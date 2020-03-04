---
title: Java反射、注解总结
tag: Java
category: Java

---

<meta name="referrer" content="no-referrer" />



[TOC]

# Java反射、注解

## 反射

在运行时，通过反射可以获取类的所有信息

### 什么是Class对象

Class对象包含了与类有关的信息，事实上，它就是用来创建类的所有的“常规”的对象。
每一个类都有一个Class对象（编译后被保存在一个同名的.class文件中），生成这个的过程是由JVM使用“类加载器”的子系统实现。

- **特别注意**
    RTTI和反射的区别只有一个，RTTI是编译器在编译时打开和检查.class文件，反射是编译器在运行时打开和检查.class文件（因为对于反射来说，编译时不能获取.class文件）

### 使用

- 获取Class对象

    ```java
    String className = "";  //全限定名（包含包名）
    Class stuClass = Class.forName(className);  //根据全限定名
    
    Class stuClass = Student.class;  //根据类名获取，这种更高效安全（会受到检查）
    
    Student student = new Student();
    Class stuClass = student.getClass();  //根据类型对象，直接获取
    
    Class<Student> stuClass = Student.class;  //可以范化，后面实例化对象的时候就可以不用强制转换了
    ```

- 实例化对象

    ```java
    //根据获取构造器实例化
    try {
        //获取所有的构造器
        Constructor[] constructors = stuClass.getConstructors();
        //获取对应的构造器，传入构造器的参数类型
        Constructor constructor = stuClass.getConstructor(String.class, int.class);
        //根据构造器实例化
        Student student = constructor.newInstance("constructor student", 2345);
    } catch (NoSuchMethodException | IllegalAccessException | InstantiationException | InvocationTargetException e) {
        e.printStackTrace();
    }
    
    //通过newInstance获取实例，默认构造器
    try {
        //类型转换，方便后面调用
        Student student = (Student) stuClass.newInstance();
    } catch (InstantiationException | IllegalAccessException e) {
        e.printStackTrace();
    }
    ```

- 获取方法

    ```java
    //获取所有的方法
    //获取方法，包括父类（Object也是）的，public的方法
    Method[] methods = stuClass.getMethods();
    //只获取本类的方法，不问权限
    Method[] methods1 = stuClass.getDeclaredMethods();
    //获取指定方法
    try {
        //public的，参数方法名
        Method method = stuClass.getMethod("toString");
        //不管权限，不包括父类的方法
        Method method1 = stuClass.getDeclaredMethod("toString");
    } catch (NoSuchMethodException e) {
        e.printStackTrace();
    }
    ```

    访问方法

    ```java
    //如果声明Class时没有泛化，就要强制转换，因为默认返回的是Object
    Student student = (Student) stuClass.newInstance();
    //调用私有方法
    try {
        Method methodStudy = stuClass.getDeclaredMethod("study", String.class);
        methodStudy.setAccessible(true);
        methodStudy.invoke(student, "通过class访问方法");
    } catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException e) {
        e.printStackTrace();
    }
    ```

- 获取接口

    ```java
    //获取接口，进而可以获取接口中的方法、成员变量
    Class<?>[] interfaces = stuClass.getInterfaces();
    ```

- 获取父类

    ```java
    //获取父类，进而可以获取父类相关的信息
    Class<?> parent = stuClass.getSuperclass();
    ```

- 获取成员变量

    ```java
    //获取public成员变量，所有的（包括父类，接口)
    Field[] fields = stuClass.getFields();
    //只获取本类的成员变量，不问访问权限
    Field[] fields1 = stuClass.getDeclaredFields();
    try {
        //获取指定public成员变量
        Field field = stuClass.getField("age");
        //根据字段只获取Student类的，不问权限
        Field field1 = stuClass.getDeclaredField("stuNum");
    } catch (NoSuchFieldException e) {
        e.printStackTrace();
    }
    ```

    修改成员变量

    ```java
    //如果声明Class时没有泛化，就要强制转换，因为默认返回的是Object
    Student student = (Student) stuClass.newInstance();
    try {
        //根据名称获取，不问权限，只能获取本类
        Field fieldStuNum = stuClass.getDeclaredField("stuNum");
        //获得访问权限，即使是私有变量
        fieldStuNum.setAccessible(true);
        //修改值，前面一个参数是引用对象，后面是值
        fieldStuNum.set(student, 6666);
    } catch (NoSuchFieldException | IllegalAccessException e) {
        e.printStackTrace();
    }
    ```

    修改final常量

    ```java
    //只能是这两种方式初始化的final常量才可以修改，直接初始化的可以修改，但调用的时候还是没有被修改，所以没有意义
    //声明一个变量，在构造器中初始化
    private final String FINAL;
    //或者利用三目表达式初始化
    private final String FINAL = null == null ? "FINAL" : null;
    
    //获取私有常量
    Field fieldFinal = stuClass.getDeclaredField("FINAL");
    fieldFinal.setAccessible(true);
    fieldFinal.set(student1, "change final");
    ```

    [![别人博客中的图片](https://user-gold-cdn.xitu.io/2017/8/12/7a376b8628ac1343bba0715b211a531b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)](https://user-gold-cdn.xitu.io/2017/8/12/7a376b8628ac1343bba0715b211a531b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)别人博客中的图片

- 获取类加载器
    `stuClass.getClassLoader()`

- 操作数组

    ```java
    String[] strArray = new String[]{"5","7","暑期","美女","女生","女神"};
    Array.set(strArray,0,"帅哥");
    Class clazz = strArray.getClass();
    if (clazz.isArray()) {
        int length = Array.getLength(strArray);
        for (int i = 0; i < length; i++) {
            Object object = Array.get(strArray, i);
            String className=object.getClass().getName();
            System.out.println("----> object=" + object+",className="+className);
        }
    }
    ```

- 获得泛型类型

    ```java
    try {
        Method method =TestHelper.class.getDeclaredMethod("getGenericHelper",HashMap.class);
        Type[] genericParameterTypes = method.getGenericParameterTypes();
        // 检验是否为空
        if (null == genericParameterTypes || genericParameterTypes.length < 1) {
            return ;
        }
        // 取 getGenericHelper 方法的第一个参数
        ParameterizedType parameterizedType=(ParameterizedType)genericParameterTypes[0];
        Type rawType = parameterizedType.getRawType();
        System.out.println("----> rawType=" + rawType);
        Type[] actualTypeArguments = parameterizedType.getActualTypeArguments();
        if (actualTypeArguments==genericParameterTypes || actualTypeArguments.length<1) {
            return ;
        }
        //  打印出每一个类型
        for (int i = 0; i < actualTypeArguments.length; i++) {
            Type type = actualTypeArguments[i];
            System.out.println("----> type=" + type);
        }
    } catch (Exception e) {
    
    }
    ```

- 例子

    ```java
    public abstract class Person {
        private String name;
        public int age;
    
        public Person(String name) {
            this.name = name;
        }
    
        public String getName() {
            return name;
        }
    
        abstract void speak(String content);
    
        @Override
        public String toString() {
            return name;
        }
    
        private void personPrivate() {
    
        }
    }
    
    public interface StudentFunI {
    
        String TAG = "StudentFunI";
    
        void read(String book);
    
        String study(String className);
    }
    
    public class Student extends Person implements StudentFunI {
        private int stuNum;
        private String book;
    
        public Student(String name, int stuNum) {
            super(name);
            this.stuNum = stuNum;
            this.book = null;
        }
    
        public Student() {
            super("默认构造");
        }
    
        @Override
        void speak(String content) {
            System.out.println(super.getName() + " speak " + content);
        }
    
        @Override
        public void read(String book) {
            this.book = book;
        }
    
        @Override
        public String study(String className) {
            return "name:" + super.getName() + "stuNum:" + stuNum + "\tclassName:" + className;
        }
    
        @Override
        public String toString() {
            return super.toString() + "  stuNum:" + stuNum + "  book:" + book + "  FINAL: " + FINAL;
        }
    
        private void studentPrivate() {
    
        }
    }
    
    public class Test {
    
        public static void main(String[] args) {
            //必须使用全限定名（含包名），获取Class对象引用的一种方式，
            //如果Student类没有被加载，调用它会触发Student类的static子句（静态初始化块）类构造器的初始化
            Class stuClass = null;
            try {
                stuClass = Class.forName("reflection.Student");
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
            //通过字面常量获取（类名）。不会触发类构造器的初始化
            Class stuClass1 = Student.class;
    
            //通过实例化后，获取Class
            Student student = new Student("new student", 1234);
            Class stuClass2 = student.getClass();
            //泛化的class引用，
            Class<Integer> intClass = int.class;
    
            example(stuClass);
            newStudent(stuClass);
    
        }
    
        /**
        * 获取实例
        * @param stuClass
        */
        private static void newStudent(Class stuClass) {
            //获取构造器，构造实例
            try {
                //获取所有的构造器
                Constructor[] constructors = stuClass.getConstructors();
                for (Constructor constructor : constructors) {
                    System.out.println("构造方法"+constructor);
                }
                Constructor constructor = stuClass.getConstructor(String.class, int.class);
                Object stuConstructor = constructor.newInstance("constructor student", 2345);
                System.out.println(stuConstructor.toString());
            } catch (NoSuchMethodException | IllegalAccessException | InstantiationException | InvocationTargetException e) {
                e.printStackTrace();
            }
    
            System.out.println();
            //通过newInstance获取实例，默认构造器
            Student student1 = null;
            try {
                //类型转换，方便后面调用
                student1 = (Student) stuClass.newInstance();
            } catch (InstantiationException | IllegalAccessException e) {
                e.printStackTrace();
            }
            System.out.println(student1 != null ? student1.study("Java") : null);
            System.out.println(student1.toString());
            //修改私有变量
            try {
                Field fieldStuNum = stuClass.getDeclaredField("stuNum");
                fieldStuNum.setAccessible(true);
                fieldStuNum.set(student1, 6666);
                Field fieldBook  = stuClass.getDeclaredField("book");;
                fieldBook.setAccessible(true);
                fieldBook.set(student1, "修改book");
                Field fieldFinal = stuClass.getDeclaredField("FINAL");
                fieldFinal.setAccessible(true);
                fieldFinal.set(student1, "change final");
            } catch (NoSuchFieldException | IllegalAccessException e) {
                e.printStackTrace();
            }
    
            System.out.println("修改后");
            System.out.println(student1.toString());
            //调用私有方法
            try {
                Method methodStudy = stuClass.getDeclaredMethod("study", String.class);
                methodStudy.setAccessible(true);
                System.out.println(methodStudy.invoke(student1, "通过class访问方法"));
            } catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException e) {
                e.printStackTrace();
            }
            System.out.println();
    
        }
    
        /**
        * class应用示例
        * @param stuClass
        */
        private static void example(Class stuClass) {
            System.out.println("getName:" + stuClass.getName());
            System.out.println("getSimpleName:" + stuClass.getSimpleName());
            System.out.println("getCanonicalName:" + stuClass.getCanonicalName());
            System.out.println();
    
            //获取public成员变量，所有的（包括父类，接口)
            Field[] fields = stuClass.getFields();
            for (Field field : fields) {
                System.out.println("包括父类 field: " + field.getName() + "\t" + field.getType());
            }
            //只获取Student类的成员变量，不问访问权限
            Field[] fields1 = stuClass.getDeclaredFields();
            for (Field field : fields1) {
                System.out.println("field: " + field.getName() + "\t" + field.getType());
            }
            try {
                //获取指定public成员变量
                Field field = stuClass.getField("age");
                //根据字段只获取Student类的，不问权限
                Field field1 = stuClass.getDeclaredField("stuNum");
            } catch (NoSuchFieldException e) {
                e.printStackTrace();
            }
    
            System.out.println();
            //获取方法，包括父类（Object也是）的
            Method[] methods = stuClass.getMethods();
            for (Method method : methods) {
                System.out.println("包括父类 method: " + method.getName());
            }
            //只获取Student的方法
            Method[] methods1 = stuClass.getDeclaredMethods();
            for (Method method : methods1) {
                System.out.println("method: " + method.getName());
            }
            //获取指定方法
            Method method = null;
            try {
                method = stuClass.getMethod("toString");
            } catch (NoSuchMethodException e) {
                e.printStackTrace();
            }
            System.out.println(method != null ? method.getName() : null);
            //获取指定的方法，不包括父类
    //        Method method1 = stuClass.getDeclaredMethod("toString");
    
            System.out.println();
    
            //获取接口
            Class<?>[] interfaces = stuClass.getInterfaces();
            for (int i = 0; i < interfaces.length; i++) {
                Class<?> inter = interfaces[i];
                Method[] methods2 = inter.getMethods();
                for (Method method1 : methods2) {
                    System.out.println(inter.getName() + "接口中的方法" + method1.getName());
                }
            }
            //获取父类
            Class<?> parent = stuClass.getSuperclass();
            Method[] parentMethods = parent.getDeclaredMethods();
            for (Method method1 : parentMethods) {
                System.out.println(parent + "父类  方法" + method1.getName());
            }
    
            //获取类加载器
            System.out.println("类加载器 " + stuClass.getClassLoader());
        }
    }
    ```

    结果：

    > getName:reflection.Student
    > getSimpleName:Student
    > getCanonicalName:reflection.Student
    >
    > 包括父类 field: TAG class java.lang.String
    > 包括父类 field: age int
    > field: FINAL class java.lang.String
    > field: stuNum int
    > field: book class java.lang.String
    >
    > 包括父类 method: toString
    > 包括父类 method: read
    > 包括父类 method: study
    > 包括父类 method: getName
    > 包括父类 method: wait
    > 包括父类 method: wait
    > 包括父类 method: wait
    > 包括父类 method: equals
    > 包括父类 method: hashCode
    > 包括父类 method: getClass
    > 包括父类 method: notify
    > 包括父类 method: notifyAll
    > method: toString
    > method: read
    > method: study
    > method: studentPrivate
    > method: speak
    > toString
    >
    > reflection.StudentFunI接口中的方法read
    > reflection.StudentFunI接口中的方法study
    > class reflection.Person父类 方法toString
    > class reflection.Person父类 方法getName
    > class reflection.Person父类 方法personPrivate
    > class reflection.Person父类 方法speak
    > 类加载器 sun.misc.Launcher$AppClassLoader@18b4aac2
    > 构造方法public reflection.Student(java.lang.String,int)
    > 构造方法public reflection.Student()
    > constructor student stuNum:2345 book:null FINAL: final tag
    >
    > name:默认构造stuNum:0 className:Java
    > 默认构造 stuNum:0 book:null FINAL: final tag
    > 修改后
    > 默认构造 stuNum:6666 book:修改book FINAL: change final
    > name:默认构造stuNum:6666 className:通过class访问方法

### 缺陷与作用

- 反射的性能
    反射机制给予Java开发很大的灵活性，但反射机制本身也有缺点，代表性的缺陷就是反射的性能，一般来说，通过反射调用方法的效率比直接调用的效率要至少慢一倍以上。
- 作用
    反射的一个很重要的作用，就是在设计模式中的应用，包括在工厂模式和代理模式中的应用

## 注解

注解：也称为元数据，为我们在代码中添加信息提供了一种形式化的方法，注解在一定程度上是在把元数据与源代码文件结合在一起。

- 作用：
    1. 能够以编译器来测试和验证的的格式，存储有关程序的额外信息
    2. 用来生成描述符文件，或新的类定义
    3. 有助于减轻编写“样板”代码的负担
    4. 将元数据保存在Java源代码中，利用annotation API为自己的注解构造处理工具

### 三种标准注解

- @Override
    表示当前的方法定义将覆盖超类中的方法，仅保留在Java源文件中
- @Desperated
    用于告知编辑器，某以程序元素（方法、成员变量）不建议使用（过时了），如果程序员使用了注解为它的元素，编译器会发出警告
- @Suppress Warnings
    关闭不当的编译器警告信息
    | 参数 | 含义 |
    | :-: | :-: |
    | deprecation | 使用了过时的类或方法时的警告 |
    | unchecked | 执行了未检查的转换时的警告 |
    | fallthrough | 当switch程序块进入下一个case而没有break的警告 |
    | path | 在类路径、源文件路径等有不存在路径时的警告 |
    | serial | 当可序列化的类缺少serialVersionUID定义时的警告 |
    | finall | 任意finally子句不能正常完成时的警告 |
    | all | 以上所有情况的警告 |

### 元注解

- @Target
    表示该注解用于什么地方
    | 参数类型 | 参数说明 |
    | :-: | :-: |
    | CONSTRUCTOP | 构造器的声明 |
    | FIEFLE | 域声明（包括enum实例）|
    | LOCAL_VARIABLE | 局部变量声明 |
    | METHOD | 方法声明 |
    | PACKAEGE | 包声明 |
    | PARAMETER | 参数声明 |
    | TYPE | 类、接口或enum声明 |
    | TYPE_PARAMETER | 注解可以用于类型参数声明（1.8新加入） |
    | TYPE_USE | 类型使用声明（1.8新加入) |

- @Retention
    表示需要在什么级别保存该注解信息
    | 参数类型 | 参数说明 |
    | :-: | :-: |
    | SOURCE | 注解将被编译器丢弃，仅在java源文件 |
    | CLASS | 注解在class文件中可用，但会被VM丢弃 |
    | RUNTIME | VM在运行期也保留注解，因此可以通过反射机制获取信息 |

- @Documented
    将注解包含在javadoc中

- @Inherited
    允许子类继承父类中的注解

- @Repeatable
    1.8新加入，表示同一个位置重复相同的注解

    ```java
    //Java8前无法这样使用
    @FilterPath("/web/update")
    @FilterPath("/web/add")
    public class A {}
    ```

### 基本使用

当一个接口直接继承java.lang.annotation.Annotation接口时，仍是接口，而并非注解。要想自定义注解类型，只能通过@interface关键字的方式，其实通过该方式会隐含地继承.Annotation接口。

```java
//自定义注解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface UseCase {
    int id();
    //使用了默认值，如果没有给出description值，就会使用这个默认值
    String description() default "no description";
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface UseCase1 {
    float price() default 0;
    //嵌套注解，注解可以作为元素
    UseCase usecase() default @UseCase(id = -1);
}


public class PasswdUtils {
    //使用
    @UseCase(id = 47, description = "至少包含一个数字")
    public boolean validatePasswd(String passwd) {
        return passwd.matches("\\w*\\d\\w*");
    }

    @UseCase(id = 48)
    public String encryptPasswd(String passwd) {
        return new StringBuilder(passwd).reverse().toString();
    }

    @UseCase(id = 49, description = "新密码不能跟旧密码相同")
    public boolean checkForNewPasswd(List<String> prevPasswd, String passwd) {
        return !prevPasswd.contains(passwd);
    }
}
//测试，通过查找@UseCase标记
public class UserCaseTracker {

    public static void trackUseCase(List<Integer> useCase, Class<?> cl) {
        //获取cl的所有方法，不问权限
        for (Method method : cl.getDeclaredMethods()) {
            //获取方法的注解，此处获取指定的UseCase标记
            UseCase uc = method.getAnnotation(UseCase.class);
            if (uc != null) {
                System.out.println("Found UseCase:" + uc.id() + " " + uc.description());
                //移除已经找到的id
                useCase.remove(new Integer(uc.id()));
            }
        }
        for (int i : useCase) {
            System.out.println("Warning Missing UseCase" + i);
        }
    }

    public static void main(String[] args) {
        List<Integer> useCases = new ArrayList<>();
        //提供一组id值
        Collections.addAll(useCases, 47, 48, 49, 50, 51);
        trackUseCase(useCases, PasswdUtils.class);
    }
}
```

输出结果：

> Found UseCase:48 no description
> Found UseCase:47 至少包含一个数字
> Found UseCase:49 新密码不能跟旧密码相同
> Warning Missing UseCase50
> Warning Missing UseCase51

- 注解元素
    注解元素可有的类型有如下：

    1. 所有基本类型（int、float等）
    2. String
    3. enum
    4. Annotation
    5. 以上类型的数组
        除此外，不可用其他类型注解元素。注解也可以作为元素的类型（注解可以嵌套）
        例如上述UseCase.java中，包含int元素id，String元素description

- 默认值
    元素不能有不确定的值，要么有默认值，要么在使用注解时提供元素的值
    默认值不能用null

- 快捷方式
    在注解中定义了value的元素，且该元素是唯一要赋值的一个元素，就可以不用采用名-值对的语法
    例如

    ```java
    @Target(ElementType.METHOD)
    @Retention(RetentionPolicy.RUNTIME)
    public @interface UseCase {
        int value() default -1;
    }
    ```

    在使用时就不用`@UseCase(value = 47)`，可以直接使用`@UseCase(47)`

- 注解不支持继承

- 使用apt处理注解
    注解处理工具apt，与javadoc一样被设计为操作Java源文件，而不是编译后的类。

### 1.8后的一些改进

- @Target
    新增两个参数

- 重复注解

    ```java
    //Java8前无法这样使用
    @FilterPath("/web/update")
    @FilterPath("/web/add")
    public class A {}
    ```

- 类型注解

    1. 创建类实例`new @Interned MyObject();`
    2. 类型映射`myString = (@NonNull String) str;`
    3. implements语句中`class UnmodifiableList<T> implements @Readonly List<@Readonly T> { ... }`
    4. throw execption声明`void monitorTemperature() throws @Critical TemperatureException { ... }`

## 相关面试题

1. 说说你对Java反射的理解
    在运行时需要一些类的信息，可以通过反射来动态获取并且修改，通过反射可以做很多事情，重点还是获取Class对象，然后就可以获取实例、访问方法、修改变量等等

2. 说说你对依赖注入的理解
    依赖注入是实现程序解耦的一种方式，降低依赖，按需绑定

    设置注入：设置注入是通过setter方法注入被调用者的实例。
    构造注入：利用构造方法来设置依赖注入的方式称为构造注入。

3. java注解
    用于标记，比javadoc更好一点的东西

4. 简单说说 Java 反射各种获取 Field 方法的区别
    getFields()：获取public成员变量，所有的（包括父类，接口)
    getDeclaredFields()：只获取本类的成员变量，不问访问权限
    getField(String name)：获取指定public成员变量
    getDeclaredField(String name)：获取指定的本类中的成员变量，不问权限

5. 简单说说 Java 反射各种获取 Method 方法的区别
    getMethods()：获取所有的方法，包括父类（Object也是）的，public的方法
    getDeclaredMethods()：只获取本类的方法，不问权限
    getMethod(String name, Class\<?>… parameterTypes)：public的，参数方法名，参数类型 getDeclaredMethod(String name, Class\<?>… parameterTypes)：不管权限，不包括父类的方法，方法名、参数类型

6. 简单说说 Java 反射各种获取类 Constructor 的方法区别
    getConstructors()：获取所有的构造器
    getDeclaredConstructors()：获取本类的所有构造器，不问权限
    getConstructor(Class<?>… parameterTypes)：获取对应的构造器，传入构造器的参数类型
    getDeclaredConstructor(Class<?>… parameterTypes)：获取指定的构造器，本类，不管权限

7. 如何防止反射、序列化攻击单例？
    修改构造器，加锁，让单例在被要求创建第二个实例化的时候抛出异常
    利用枚举类型，编写一个包含单个元素的枚举类型

    针对序列化：实现Serializable，重写readResolve方法

    针对该clone：实现Cloneable接口，重写克隆方法

8. 简单说说 Java Class.newInstance() 与Constructor.newInstance() 方法区别
    Class.newInstance() 只能够调用无参的构造函数，即默认的构造函数；
    Constructor.newInstance() 可以根据传入的参数，调用任意构造构造函数。

    Class.newInstance() 抛出所有由被调用构造函数抛出的异常。

    Class.newInstance() 要求被调用的构造函数是可见的，也即必须是public类型的;
    Constructor.newInstance() 在特定的情况下，可以调用私有的构造函数。

## 相关参考

Java 编程思想
[Java 反射由浅入深 | 进阶必备](https://juejin.im/post/598ea9116fb9a03c335a99a4)
[java反射详解](https://www.cnblogs.com/rollenholt/archive/2011/09/02/2163758.html)
[Java 反射机制详解](https://blog.csdn.net/gdutxiaoxu/article/details/68947735)
[深入理解Java注解类型(@Annotation)](https://blog.csdn.net/javazejian/article/details/71860633)
[Java注解深入理解](https://www.jianshu.com/p/4068da3c8d3d)
[单例与序列化的那些事儿](http://www.hollischuang.com/archives/1144)