---
title: Java序列化总结
tag: Java
category: Java

---

<meta name="referrer" content="no-referrer" />



[TOC]

# Java序列化

对象的序列化可以实现轻量级持久性

## Serializable

简单序列化到文件

```java
public class Login implements Serializable {
    private String userName;
    private String passwd;

    public Login(String userName, String passwd) {
        this.userName = "no transient " + userName;
        this.passwd = "transient " + passwd;
    }

    @Override
    public String toString() {
        return "login info:\nusername:" + userName + "\npasswd:" + passwd;
    }

    public static void main(String[] args) throws IOException, InterruptedException, ClassNotFoundException {
        Login login = new Login("Tom", "123456");
        System.out.println(login);
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("login.out"));
        oos.writeObject(login);
        oos.close();

        System.out.println("Serializable");
        TimeUnit.SECONDS.sleep(1);

        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("login.out"));
        Login login1 = (Login) ois.readObject();
        System.out.println(login1);
    }
}
```

## Externalizable

Externalizable是用来实现序列化的控制

- 简单用法

    ```java
    public class Login implements Externalizable {
        private String userName;
        private String passwd;
    
        public Login(String userName, String passwd) {
            this.userName = "no transient " + userName;
            this.passwd = "transient " + passwd;
        }
    
        public Login() {
            //readExternal会自动调用默认的构造方法（所有的默认构造方法都会被调用）
        }
    
        @Override
        public String toString() {
            return "login info:\nusername:" + userName + "\npasswd:" + passwd;
        }
    
        @Override
        public void writeExternal(ObjectOutput out) throws IOException {
            out.writeObject(passwd);
            out.writeObject(userName);
        }
    
        @Override
        public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
            userName = (String) in.readObject();
            passwd = (String) in.readObject();
        }
    
        public static void main(String[] args) throws IOException, InterruptedException, ClassNotFoundException {
            Login login = new Login("Tom", "123456");
            System.out.println(login);
            ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("login.out"));
            oos.writeObject(login);
            oos.close();
    
            System.out.println("Externalizable");
            TimeUnit.SECONDS.sleep(1);
    
            ObjectInputStream ois = new ObjectInputStream(new FileInputStream("login.out"));
            Login login1 = (Login) ois.readObject();
            System.out.println(login1);
        }
    }
    /**
     * 结果：
     * login info:
     * username:no transient Tom
     * passwd:transient 123456
     * Externalizable
     * login info:
     * username:transient 123456
     * passwd:no transient Tom
     */
    ```

    从结果发现一点，username和passwd的值，序列化前后的值好像并不一样，看代码，发现，我们在writeExternal的时候passwd在前，而readExternal的时候又在后，说明手动实现的时候还得注意顺序，特别是同类型的。

- Externalizable的替代方法
    实现Serialiable接口，**添加（不是覆盖或实现）**writeObject()和readObject()方法，这样一旦对象被序列化，就会自动地调用这两个方法（不会使用默认地方法）

    ```java
    //必须具有准确地方法特征签名
    private void writeObject(ObjectOutputStream stream) throws IOException;
    
    private void readObject(ObjectInputStream stream) throws IOException, ClassNotFoundException;
    ```

    实现：

    ```java
    public class Login implements Serializable {
        private String userName;
        private transient String passwd;
    
        public Login(String userName, String passwd) {
            this.userName = "no transient " + userName;
            this.passwd = "transient " + passwd;
        }
    
        private void readObject(ObjectInputStream stream) throws IOException, ClassNotFoundException {
            //调用默认的readObject
            stream.defaultReadObject();
            passwd = (String) stream.readObject();
        }
    
        private void writeObject(ObjectOutputStream stream) throws IOException {
            //调用默认地writeObject
            stream.defaultWriteObject();
            //手动实现transient修饰的类的序列化
            stream.writeObject(passwd);
        }
    
        @Override
        public String toString() {
            return "login info:\nusername:" + userName + "\npasswd:" + passwd;
        }
    
        public static void main(String[] args) throws IOException, InterruptedException, ClassNotFoundException {
            Login login = new Login("Tom", "123456");
            System.out.println(login);
            ByteArrayOutputStream buf = new ByteArrayOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(buf);
            oos.writeObject(login);
    
            ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream  (buf.toByteArray()));
            Login login1 = (Login) ois.readObject();
            System.out.println(login1);
        }
    }
    
    /**
     * 结果：
     * login info:
     * username:no transient Tom
     * passwd:transient 123456
     * login info:
     * username:no transient Tom
     * passwd:transient 123456
     */
    ```

## transient

用于控制Serializable对象的序列化操作。transient可以逐个字段的关闭序列化

例子：假设某个Login对象保存某个特定的登陆会话信息。

```java
public class Login implements Serializable {
    private String userName;
    private transient String passwd;

    public Login(String userName, String passwd) {
        this.userName = userName;
        this.passwd = passwd;
    }

    @Override
    public String toString() {
        return "login info:\nusername:" + userName + "\npasswd:" + passwd;
    }

    public static void main(String[] args) throws IOException, InterruptedException, ClassNotFoundException {
        Login login = new Login("Tom", "123456");
        System.out.println(login);
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("login.out"));
        oos.writeObject(login);
        oos.close();

        System.out.println("Serializable");
        TimeUnit.SECONDS.sleep(1);

        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("login.out"));
        Login login1 = (Login) ois.readObject();
        System.out.println(login1);
    }
}
```

> 结果：
> login info:
> username:Tom
> passwd:123456
> Serializable
> login info:
> username:Tom
> passwd:null

## 深入理解序列化

[Java 序列化的高级认识](https://www.ibm.com/developerworks/cn/java/j-lo-serial/index.html)
[深入学习Java序列化](http://beautyboss.farbox.com/post/study/shen-ru-xue-xi-javaxu-lie-hua)

## 相关面试题

1. transient关键字
    用于控制Serializable对象的序列化操作。transient可以逐个字段的关闭序列化

2. 说说你对 Java 的 transient 关键字理解
    对于不需要被序列化的属性就可以通过加上 transient 关键字来处理。一旦属性被 transient 修饰就不再是对象持久化的一部分(一个静态变量不管是否被transient修饰，均不能被序列化)，该属性的内容在序列化后将无法获得访问，transient 关键字只能修饰属性变量成员而不能修饰方法和类（注意局部变量是不能被 transient 关键字修饰的），属性成员如果是引用类型也需要保证实现 Serializable 接口；此外在 Java 中对象的序列化可以通过实现两种接口来实现，若实现的是 Serializable 接口则所有的序列化将会自动进行，若实现的是 Externalizable 接口则没有任何东西可以自动序列化，需要在 writeExternal 方法中进行手工指定所要序列化的变量，这与是否被 transient 修饰无关。

3. 如何将一个Java对象序列化到文件里
    通过Serializable自动序列化或通过Externalizable手动序列化

4. Java 序列化中 writeReplace() 方法有什么作用？

5. Java 序列化中 readResolve() 方法有什么作用？

6. Java 序列化存储传输为什么不安全？怎么解决？
    [【码农每日一题】Java 自定义序列化(Part 2)相关问题](https://mp.weixin.qq.com/s/LaASpNkXl_1Gljc1wl5fIA)

7. 简单说说 Externalizable 与 Serializable 有什么区别？
    使用 transient 还是用 writeObject() 和 readObject() 方法都是基于 Serializable 接口的序列化；JDK 提供的另一个序列化接口 Externalizable 继承自 Serializable，使用该接口后基于 Serializable 接口的序列化机制就会失效（包括 transient，因为 Externalizable 不会主动序列化），当使用该接口时序列化的细节需要由我们自己去实现，另外使用 Externalizable 主动进行序列化时当读取对象时会调用被序列化类的无参构方法去创建一个新的对象，然后再将被保存对象的字段值分别填充到新对象中，所以实现 Externalizable 接口的类必须提供一个无参 public 的构造方法。

8. 对于transient修饰的属性如何在不删除修饰符的情况下让其可以序列化？

9. Serializable 序列化中自定义 readObjectNoData() 方法有什么作用？
    这个方法主要用来保证通过继承扩容后对老版本的兼容性，适用场景如下：比如类 Persion 被序列化到硬盘后存储为文件 old.txt，接着 Person 被修改继承自 Animal，为了保证用新的 Person 反序列化老版本 old.txt 文件且 Animal 类的成员有默认值则可以在 Animal 类中定义 readObjectNoData 方法返回成员的默认值，具体可以参见 ObjectInputStream 类中的 readSerialData 方法判断

    参考[Java 自定义序列化(Part 1)相关问题](https://mp.weixin.qq.com/s/dOTws5ZeFDDW_mOBVPpB4g)

10. 父类实现了 Serializable 接口后如何不修改父类的情况下让子类不可序列化

11. 子类实现序列化接口与父类实现序列化接口有什么区别吗

12. 父类实现了 Serializable 接口后如何不修改父类的情况下让子类不可序列化
    参考[Java 序列化与继承相关面试题](https://mp.weixin.qq.com/s/MxW4JlbUZwG7YIsmlMIKvQ)

## 相关参考

### Java 编程思想