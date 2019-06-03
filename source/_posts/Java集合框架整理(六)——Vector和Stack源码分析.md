---
title: Java集合框架整理(六)——Vector和Stack源码分析
tag: Java
---

[TOC]

# Vector和Stack源码分析

## Vector源码

> 实现动态数组
>
> 支持同步访问
>
> 事先不知道数组大小，类似于ArrayList

先看看类的继承关系吧

```java
public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    //数据数组
    protected Object[] elementData;
    //元素个数
    protected int elementCount;
    //容量增量因子
    protected int capacityIncrement;
    private static final long serialVersionUID = -2767605614048989439L;

    public Vector(int initialCapacity, int capacityIncrement) {
        super();
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        this.elementData = new Object[initialCapacity];
        this.capacityIncrement = capacityIncrement;
    }

    public Vector(int initialCapacity) {
        this(initialCapacity, 0);
    }

    public Vector() {
        this(10);
    }

    public Vector(Collection<? extends E> c) {
        elementData = c.toArray();
        elementCount = elementData.length;
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, elementCount, Object[].class);
    }
    ...
}
```

Vector是通过继承AbstractList实现的，同时实现了List、RandomAccess、Cloneable、Serializable接口，AbstractList和List都在[**Java集合框架**](http://blog.penghesheng.cn/2019/04/25/Java集合框架整理(六)——Vector和Stack源码分析/)中讲到了，RandomAccess、Cloneable、Serializable分别表示支持随机访问、可拷贝、可序列化等。

Vector有三个全局变量elementData、elementCount、capacityIncrement，分别是数组、元素个数以及容量增量因子。通过Vector的几个构造器可以知道，Vector容量默认是10，增量因子为0，有了容量，最终会直接进行实例化。如果通过Collection来实例化一个Vector，需要注意的是要转为Object[]类型

接着看看几个操作方法吧

### add

```java
public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    ...
    //在末尾插入
    public synchronized boolean add(E e) {
        modCount++;
        ensureCapacityHelper(elementCount + 1);  //保证容量足够
        elementData[elementCount++] = e;
        return true;
    }
    //直接插入，在末尾
    public synchronized void addElement(E obj) {
        modCount++;
        ensureCapacityHelper(elementCount + 1);
        elementData[elementCount++] = obj;
    }
    //在指定位置插入
    public synchronized void insertElementAt(E obj, int index) {
        modCount++;
        if (index > elementCount) {
            throw new ArrayIndexOutOfBoundsException(index
                                                     + " > " + elementCount);
        }
        ensureCapacityHelper(elementCount + 1);
        System.arraycopy(elementData, index, elementData, index + 1, elementCount - index);
        elementData[index] = obj;
        elementCount++;
    }

    //保证有足够的容量，当最小需求容量大于当前数据个数的时候，需要进行扩容
    private void ensureCapacityHelper(int minCapacity) {
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;  //最大数组容量
    //真正进行扩容的地方
    private void grow(int minCapacity) {
        int oldCapacity = elementData.length;
        //根据是否有增量因子进行扩容，没有的话增量因子为当前数据个数
        int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                         capacityIncrement : oldCapacity);
        //增量扩容后是否满足最小需求容量，不满足则直接使用最小容量为新的容量
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        //检查是否超出最大数组容量
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    //当超出最大数组容量后使用Integer.MAX_VALUE作为容量
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
}
```

addElement方法是synchronized修饰的，是线程安全的，其实Vector的很多方法都是synchronized修饰的，所以Vector是一个线程安全的集合
add相关的方法跟ArrayList的差不多；在扩容的是否会检查是否设置了增量因子的，有增量因子就直接使用，没有则用当前数据个数作为增量因子进行扩容，然后得到的容量是当前数据个数+增量因子

addAll等方法的原理都是一样的，这里就不再一一看源码了

### remove

```java
public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    ...
    //删除元素
    public synchronized boolean removeElement(Object obj) {
        modCount++;
        int i = indexOf(obj);
        if (i >= 0) {
            removeElementAt(i);
            return true;
        }
        return false;
    }
    //删除某个位置的元素
    public synchronized void removeElementAt(int index) {
        modCount++;
        if (index >= elementCount) {
            throw new ArrayIndexOutOfBoundsException(index + " >= " + elementCount);
        }
        else if (index < 0) {
            throw new ArrayIndexOutOfBoundsException(index);
        }
        int j = elementCount - index - 1;
        if (j > 0) {
            System.arraycopy(elementData, index + 1, elementData, index, j);
        }
        elementCount--;
        elementData[elementCount] = null; /* to let gc do its work */
    }
    //移除所有的元素
    public synchronized void removeAllElements() {
        modCount++;
        // Let gc do its work
        for (int i = 0; i < elementCount; i++)
            elementData[i] = null;

        elementCount = 0;
    }
    //移除两个下标之前的元素
    protected synchronized void removeRange(int fromIndex, int toIndex) {
        modCount++;
        int numMoved = elementCount - toIndex;
        System.arraycopy(elementData, toIndex, elementData, fromIndex, numMoved);
        //设为null方便GC
        int newElementCount = elementCount - (toIndex-fromIndex);
        while (elementCount != newElementCount)
            elementData[--elementCount] = null;
    }
}
```

### get

```java
public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    ...
    //获取指定位置的元素
    public synchronized E elementAt(int index) {
        if (index >= elementCount) {
            throw new ArrayIndexOutOfBoundsException(index + " >= " + elementCount);
        }

        return elementData(index);
    }
    //获取第一个元素
    public synchronized E firstElement() {
        if (elementCount == 0) {
            throw new NoSuchElementException();
        }
        return elementData(0);
    }
    //获取最后一个元素
    public synchronized E lastElement() {
        if (elementCount == 0) {
            throw new NoSuchElementException();
        }
        return elementData(elementCount - 1);
    }
    //获取指定下标的元素
    public synchronized E get(int index) {
        if (index >= elementCount)
            throw new ArrayIndexOutOfBoundsException(index);

        return elementData(index);
    }
}
```

### set

```java
public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    ...
    //既可以是插入也可以是更新
    public synchronized void setElementAt(E obj, int index) {
        if (index >= elementCount) {
            throw new ArrayIndexOutOfBoundsException(index + " >= " + elementCount);
        }
        elementData[index] = obj;
    }
    //更新指定位置的元素
    public synchronized E set(int index, E element) {
        if (index >= elementCount)
            throw new ArrayIndexOutOfBoundsException(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
}
```

### 其他方法

```java
public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    ...
    //复制到anArray中
    public synchronized void copyInto(Object[] anArray) {
        System.arraycopy(elementData, 0, anArray, 0, elementCount);
    }

    //手动进行扩容或缩容
    public synchronized void setSize(int newSize) {
        modCount++;
        if (newSize > elementCount) {
            ensureCapacityHelper(newSize);
        } else {
            for (int i = newSize ; i < elementCount ; i++) {
                elementData[i] = null;
            }
        }
        elementCount = newSize;
    }
    //得到当前容量
    public synchronized int capacity() {
        return elementData.length;
    }
    //得到当前元素个数
    public synchronized int size() {
        return elementCount;
    }
    //返回一个枚举类型的Vector，Vector中所有的数组变成枚举类型
    public Enumeration<E> elements() {
        return new Enumeration<E>() {
            int count = 0;

            public boolean hasMoreElements() {
                return count < elementCount;
            }

            public E nextElement() {
                synchronized (Vector.this) {
                    if (count < elementCount) {
                        return elementData(count++);
                    }
                }
                throw new NoSuchElementException("Vector Enumeration");
            }
        };
    }
    //是否包含集合c的元素，利用AbstractCollection的方法进行判断，removeAll、retainAll方法都是用的父类的
    public synchronized boolean containsAll(Collection<?> c) {
        return super.containsAll(c);
    }
    //批量分割集合
    public synchronized List<E> subList(int fromIndex, int toIndex) {
        return Collections.synchronizedList(super.subList(fromIndex, toIndex),
                                            this);
    }
    ...
}
```

其他的这些方法除了一些个别的，大多数都和ArrayList中的一样，不同的是Vector的方法是用synchronized修饰的，是线程安全的。Vector的迭代器原理跟ArrayList的都差不多，准确的说跟AbstractList中的差不多

## Stack源码

```java
public class Stack<E> extends Vector<E> {
    public Stack() {
    }

    public E push(E item) {
        addElement(item);
        return item;
    }

    public synchronized E pop() {
        E       obj;
        int     len = size();
        obj = peek();
        removeElementAt(len - 1);
        return obj;
    }

    public synchronized E peek() {
        int     len = size();
        if (len == 0)
            throw new EmptyStackException();
        return elementAt(len - 1);
    }

    public boolean empty() {
        return size() == 0;
    }

    public synchronized int search(Object o) {
        int i = lastIndexOf(o);
        if (i >= 0) {
            return size() - i;
        }
        return -1;
    }

    private static final long serialVersionUID = 1224463164541339165L;
}
```

Stack的源码就很简单了，继承自Vector，使用的就是Vector中的数组，只是改了一下符合栈的一些方法的名字，逻辑操作做了适当的处理，同时Stack也是线程安全的

## 总结

Vector并没有啥新奇的东西，看来ArrayList源码的就知道，两者差不了太多，两者最大的区别就是Vector是线程安全的，ArrayList是不安全的，而Stack则是完全基于Vector实现的，所以也没啥特别的
需要注意的是Vector和ArrayList的扩容方法虽然都差不多（都是基于数组实现），但扩容的大小还是不一样的