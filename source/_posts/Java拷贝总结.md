---
title: Java拷贝总结
tag: Java
---

[TOC]

# Java拷贝相关知识

clone顾名思义就是复制， 在Java语言中， clone方法被对象调用，所以会复制对象。所谓的复制对象，首先要分配一个和源对象同样大小的空间，在这个空间中创建一个新的对象

## clone方法

clone是Object这个超类的一个Protect方法。
`protected native Object clone() throws CloneNotSupportedException;`

通过源码，我们可以知道以下几点：

1. clone是一个受保护的native方法（native方法是非Java语言实现的代码，仅供Java程序调用），因为受保护，所以不能直接从类外进行访问
2. x.clone() != x 必须为真，克隆对象将有单独的内存分配
3. x.clone().getClass() == x.getClass()，原始和克隆的对象应该具有同样的类型，但不是强制性的
4. x.clone().equals(x) 应该为true，所比较的对象内容应该相同，但不强制性
5. 被复制的类需要实现Clonenable接口
6. 覆盖clone()方法，访问修饰符设为public。方法中调用super.clone()方法得到需要的复制对象

## 浅拷贝(Shallow Copy)

引例：

```java
/**
 * @author 14512 on 2018/7/27.
 */
public class Person implements Cloneable {
    private String name;
    private int age;
    private Address address;

    public void setAddress(Address address) {
        this.address = address;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public void setName(String name) {
        this.name = name;
    }

    public void setAddress(String addressStr, int addressCode) {
        this.address.setAddress(addressStr);
        this.address.setAddressCode(addressCode);
    }

    public Address getAddress() {
        return address;
    }

    public int getAge() {
        return age;
    }

    public String getName() {
        return name;
    }

    @Override
    public String toString() {
        return "name:" + name + "\tage:" + age + "\taddress:"+address.getAddress()+address.getAddressCode();
    }

    @Override
    public Object clone() {
        try {
            return super.clone();
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
            return null;
        }
    }
}

/**
 * @author 14512 on 2018/7/27.
 * Person的成员属性
 */
public class Address {
    private String address;
    private int addressCode;

    public Address(String address, int addressCode) {
        this.address = address;
        this.addressCode = addressCode;
    }

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }

    public int getAddressCode() {
        return addressCode;
    }

    public void setAddressCode(int addressCode) {
        this.addressCode = addressCode;
    }
}

/**
 * @author 14512 on 2018/7/25.
 * 测试类
 */
public class Test {

    public static void main(String[] args) {
       Person person = new Person();
       person.setName("person");
       person.setAge(20);
       person.setAddress(new Address("person address", 100));
       System.out.println("person " + person.toString());
       Person person1 = (Person) person.clone();
       System.out.println("clone person " + person1.toString());
       System.out.println(person.getName() == person1.getName());
       System.out.println("修改clone person的属性值");
       person1.setAge(21);
       person1.setName("clone person");
       person1.setAddress("clone address", 200);
       System.out.println(person.getName() == person1.getName());
       System.out.println("person " + person.toString());
       System.out.println("clone person" + person1.toString());
    }
}

/**
 * 结果：
 * person name:person   age:20  address:person address100
 * clone person name:person age:20  address:person address100
 * true
 * 修改clone person的属性值
 * false
 * person name:person   age:20  address:clone address200
 * clone personname:clone person    age:21  address:clone address200
 */
```

从结果中可以看出，基本类型的属性age，在clone后person1修改为21，并不会修改person的age；但是当你修改person1的address时，person的address也被修改了，这就意味着，clone后，基本类型的属性时真正clone了，但是引用类的address属性并没有真正的clone，它们指向的地址还是一个地址（尽管引用也变了，只是进行了引用的传递，而没有真实的创建一个新的对象）。这个就是浅拷贝和深拷贝的区别所在了。浅拷贝只能拷贝基本类型（基本变量引用和地址都变），无法真正拷贝引用属性（因为引用变量clone了，但是指向的地址并没有改变）

**注意**：例子中同样使用了String的引用，可以看到，在setName之前，两个name的地址是一样的，但是setName之后，两个name的地址就不一样了，所以操作person1的name并没有影响person的name。这并不是说String这个引用例外了，而是String是不可变的（final类），所以赋值后会分配一个新的内存来存储，新的引用就指向了新的地址，所以才会出现这种情况。

## 深拷贝(Deep Copy)

引例：在上面的例子中，修改一下Person的clone（自定义clone）

```java
@Override
public Object clone() {
    Person person = new Person();
    person.setName(name);
    person.setAge(age);
    person.setAddress(new Address(address.getAddress(), address.getAddressCode()));
    return person;
}

/**
 * 结果：
 * person name:person	age:20	address:person address100
 * clone person name:person	age:20	address:person address100
 * true
 * 修改clone person的属性值
 * false
 * person name:person	age:20	address:person address100
 * clone personname:clone person	age:21	address:clone address200
 */
```

从结果中可以看到，不管是age还是address，person1的修改并没有影响person的值，这就意味着person1是person的克隆（两者完全独立）

**注意**：在clone后，person1.setName之前，我们比较了一次两个name的地址，为什么还是指向同一个呢？再看一下我们自定义的clone方法，我们并没有操作String的clone，并且String这个类并没有复写clone这个方法。setNeme之后，地址就不一样了还是String的不可变性

- 利用成员类实现Clonenable，来实现深拷贝

```java
/**
 * @author 14512 on 2018/7/27.
 */
public class Person implements Cloneable {
    ...

    @Override
    public Object clone() {
        try {
            Person person = (Person) super.clone();
            person.address = (Address) address.clone();
            return person;
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
            return null;
        }
    }
}

/**
 * @author 14512 on 2018/7/27.
 */
public class Address implements Cloneable {
    ...

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}

/**
 * 结果：
 *person name:person	age:20	address:person address100
 * clone person name:person	age:20	address:person address100
 * true
 * 修改clone person的属性值
 * false
 * person name:person	age:20	address:person address100
 * clone personname:clone person	age:21	address:clone address200
 */
```

利用Address实现Clonenable接口，然后在Person中调用Address的clone方法，从而实现深拷贝。结果同上面一个一样。其实这也算自定义clone。

- 通过序列化来实现深拷贝

```java
public class Person implements Serializable {
    ...
}

public class Address implements Serializable {
   ...
}

/**
 * @author 14512 on 2018/7/25.
 */
public class Test {

    public static void main(String[] args) throws IOException {
        ObjectOutputStream oos = null;
        ObjectInputStream ois = null;

        try {
            Person person = new Person();
            person.setName("person");
            person.setAge(20);
            person.setAddress(new Address("person address", 100));
            System.out.println("person " + person.toString());

            Person person1 = null;

            //通过序列化实现深拷贝
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            oos = new ObjectOutputStream(bos);
            // 序列化以及传递这个对象 
            oos.writeObject(person);
            oos.flush();

            ByteArrayInputStream bin = new ByteArrayInputStream(bos.toByteArray());
            ois = new ObjectInputStream(bin);
            // 返回新的对象
            person1 = (Person) ois.readObject();

            System.out.println("clone person " + person1.toString());
            System.out.println(person.getName() == person1.getName());
            System.out.println("修改clone person的属性值");
            person1.setAge(21);
            person1.setName("clone person");
            person1.setAddress("clone address", 200);
            System.out.println(person.getName() == person1.getName());
            System.out.println("person " + person.toString());
            System.out.println("clone person" + person1.toString());
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        } finally {
            if (oos != null) {
                oos.close();
            }
            if (ois != null) {
                ois.close();
            }
        }

    }
}

/**
 * 结果：
 *person name:person	age:20	address:person address100
 * clone person name:person	age:20	address:person address100
 * false
 * 修改clone person的属性值
 * false
 * person name:person	age:20	address:person address100
 * clone personname:clone person	age:21	address:clone address200
 */
```

先都实现Serializable（这是序列化的前提），然后通过序列化，传递后获取。结果跟之前差不多。唯一的差别就是前后name的比较，通过序列化的深拷贝，name并没有像之前的那样，先是指向了同一个地址，而是一开始就是两个地址了。

## 延迟拷贝(Lazy Copy)。

延迟拷贝是浅拷贝和深拷贝的一个组合，实际上很少会使用。 当最开始拷贝一个对象时，会使用速度较快的浅拷贝，还会使用一个计数器来记录有多少对象共享这个数据。当程序想要修改原始的对象时，它会决定数据是否被共享（通过检查计数器）并根据需要进行深拷贝。

延迟拷贝从外面看起来就是深拷贝，但是只要有可能它就会利用浅拷贝的速度。当原始对象中的引用不经常改变的时候可以使用延迟拷贝。由于存在计数器，效率下降很高，但只是常量级的开销。而且, 在某些情况下, 循环引用会导致一些问题。

## 总结

1. 克隆方法用于创建对象的拷贝，为了使用clone方法，类必须实现java.lang.Cloneable接口重写protected方法clone，如果没有实现Clonebale接口会抛出CloneNotSupportedException.
2. 在克隆java对象的时候不会调用构造器
3. java提供一种叫浅拷贝（shallow copy）的默认方式实现clone，创建好对象的副本后然后通过赋值拷贝内容，意味着如果你的类包含引用类型，那么原始对象和克隆都将指向相同的引用内容，这是很危险的，因为发生在可变的字段上任何改变将反应到他们所引用的共同内容上。为了避免这种情况，需要对引用的内容进行深度克隆。
4. 按照约定，实例的克隆应该通过调用super.clone()获取，这样有助克隆对象的不变性。如：clone!=original和clone.getClass()==original.getClass()，尽管这些不是必须的

**浅拷贝**
[![网上盗的图，浅拷贝](https://images2015.cnblogs.com/blog/690102/201607/690102-20160727132640216-1387063948.png)](https://images2015.cnblogs.com/blog/690102/201607/690102-20160727132640216-1387063948.png)网上盗的图，浅拷贝
**深拷贝**
[![网上盗的图，深拷贝](https://images2015.cnblogs.com/blog/690102/201607/690102-20160727132838528-120069275.png)](https://images2015.cnblogs.com/blog/690102/201607/690102-20160727132838528-120069275.png)网上盗的图，深拷贝

## 相关面试题

1. String 克隆的特殊性在哪里？StringBuffer 和 StringBuilder 呢？
    String克隆仍然是复制引用，并不会修改地址，但是String在内存中是不可修改的对象（被声明为final类，String是不可变的），所以修改值后，新的引用会指向一个新的内存。
    StringBuffer和StringBuilder的内部实现方式与String不同，StringBuffer和StringBuilder都是在自身上修改的，所以它们的克隆也只是复制引用，但是引用不会指向新的地址，仍是原来的地址（针对浅拷贝）
2. 什么是深拷贝和浅拷贝
    深拷贝就是一个拷贝出来的对象，不管是基本类型还是引用类型的属性，都与原来的对象没有关联，是独立的内存空间
    浅拷贝不同的地方就是，基本类型的属性是独立的空间，而引用类型的属性仅仅是引用拷贝了，但都还是指向同一个地址的
3. 为什么需要克隆
    主要是为了在新的上下文环境中复用对象的部分或全部数据。避免操作修改了原数据
4. 浅度克隆（浅拷贝）和深度克隆（深拷贝）的区别是什么？
    克隆的程度，浅拷贝无法真正实现引用型的克隆，深度拷贝就必须要实现这个
5. 实现对象克隆常见的方式有哪些，具体怎么做
    浅拷贝：实现Clonenable，复写clone方法，不做多余的操作
    深拷贝：引用类型的属性、本身都要实现Clonenable接口，引用类复写原生的clone方法（可以什么都不用改变），本身类实现clone的时候可以根据自己的逻辑实现，也可以调用引用类的clone方法从而使引用类型的属性实现克隆。还可以通过序列化来实现深拷贝

## 参考相关链接

[Java深拷贝和浅拷贝](https://lrh1993.gitbooks.io/android_interview_guide/content/java/basis/copy.html)
[Java提高篇——对象克隆（复制）](https://www.cnblogs.com/Qian123/p/5710533.html#_labelTop)
[详解Java中的clone方法 – 原型模式](https://blog.csdn.net/zhangjg_blog/article/details/18369201)