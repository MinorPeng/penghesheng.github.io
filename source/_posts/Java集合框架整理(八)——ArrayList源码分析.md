---
title: Java集合框架整理(八)——ArrayList源码分析
tag: Java
date: 
---

# ArrayList源码分析

ArrayList（动态数组）：

> 以数组实现。节约空间，但数组有容量限制。超出限制时会增加50%容量，用System.arraycopy()复制到新的数组，因此最好能给出数组大小的预估值。
>
> 默认第一次插入元素时创建大小为10的数组。
>
> 按数组下标访问元素—get(i)/set(i,e) 的性能很高，这是数组的基本优势。
>
> 直接在数组末尾加入元素—add(e)的性能也高，但如果按下标插入、删除元素 —add(i,e), remove(i), remove(e)，则要用System.arraycopy()来移动部分受影响的元素，性能就变差了，这是基本劣势。

先看看它的类继承关系吧

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    private static final long serialVersionUID = 8683452581122892189L;
    //默认容量大小为10
    private static final int DEFAULT_CAPACITY = 10;
    //给空实例的共享的空数组实例
    private static final Object[] EMPTY_ELEMENTDATA = {};
    //用于区分上边的EMPTY_ELEMENTDATA，有默认大小的空实例
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    //不可序列化的元素数组，就是存放数据的地方
    transient Object[] elementData;
    //元素个数
    private int size;
    //指定大小初始化数组，一般建议使用这个构造器
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            //为0的时候使用空数组
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
    //没有指定容量的时候，elementData指定为DEFAULTCAPACITY_EMPTY_ELEMENTDATA
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
    //根据Collection进行初始化
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            //必须为数组类型
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
    ...
}
```

ArrayList是通过继承AbstractList并实现List、RandomAccess、Cloneable、Serializable等接口，关于AbstractList和List可以看总纲[**Java集合框架整理**](http://blog.penghesheng.cn/2019/05/06/Java集合框架整理(八)——ArrayList源码分析/)，RandomAccess这个是表示支持随机访问，优化性能的

ArrayList中定义了几个全局变量，DEFAULT_CAPACITY是默认的容量大小，EMPTY_ELEMENTDATA、DEFAULTCAPACITY_EMPTY_ELEMENTDATA分别表示一个空数组实例和有默认容量的空数组实例，然后还有真正的数据存放的引用elementData和元素个数size。

通过几种构造器，可以知道，如果在没有指定容量的情况下，默认是一个由默认容量的空数组实例，指定为0的时候就是一个空数组实例，指定容量大小时就实例化一个相应大小的数组。所以不管怎样，只要实例化ArrayList后elementData就不是一个null，而是一个实例化过的对象。当根据Collection进行初始化的时候，一定要保证能够得到是Object[]，不是的话也会处理成Object[]。

接着看看常用的操作吧

## add

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    ...
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // 检查容量是否足够，在保证足够的情况下进行插入
        elementData[size++] = e;
        return true;
    }

    public void add(int index, E element) {
        rangeCheckForAdd(index);  //检查是否越界

        ensureCapacityInternal(size + 1);  // 检查容量是否足够
        //直接通过System.arraycopy函数进行数据的移动
        System.arraycopy(elementData, index, elementData, index + 1, size - index);
        elementData[index] = element;
        size++;
    }

    //确保容量足够，不够就进行扩容，先进行第一步检查
    private void ensureCapacityInternal(int minCapacity) {
        //如果之前还是DEFAULTCAPACITY_EMPTY_ELEMENTDATA的空数组实例，最小需求容量就是默认值10
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        ensureExplicitCapacity(minCapacity);
    }
    //接着判断是否需要进行扩容
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;  //修改次数，在AbstractList中定义的
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;  //数组最大容量
    //真正进行扩容的地方
    private void grow(int minCapacity) {
        int oldCapacity = elementData.length;
        //在原来的基础上进行一半扩容
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        //然后判断是否满足最小需求容量
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        //检查是否超出最大容量，以最大值进行扩容
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
}
```

在add的时候，必须确保容量是足够的，所以add之前都要先进行判断，不够就会进行扩容，每次扩容都会增加目前元素个数的**一半**（如果当前是10，扩容后就是10+10/2），如果到达的了最大容量Integer.MAX_VALUE - 8就不会再增加，以这个为最小需求容量进行数组移动（如果真到了这个地步也不应该采用ArrayList）

**addAll:**

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    ...
    //添加一个集合的数据
    public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }

    public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index);  //检查是否越界

        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount
        //判断插入的位置是否已经有元素了还是在后面的空白部分插入
        int numMoved = size - index;  
        if (numMoved > 0)
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);

        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }
}
```

## remove

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    ...
    //删除指定位置的元素并返回
    public E remove(int index) {
        rangeCheck(index);
        modCount++;
        E oldValue = elementData(index);
        int numMoved = size - index - 1;  //是否是删除的最后一个元素，不是则需要进行数据前移
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
        return oldValue;
    }
    //删除某个元素（集合中所有出现的）
    public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            //通过遍历整个数组进行删除
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
    //删除一个集合c中的所有元素，即保留不在集合c的元素
    public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, false);
    }
    //保留包含在集合c中的元素
    public boolean retainAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, true);
    }
    //直接覆盖掉
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
    //批量删除
    private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;
        boolean modified = false;
        try {
            //根据标志量complement来决定是保留集合c中的元素还是保留不在c中的元素
            for (; r < size; r++)
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            if (r != size) {
                //剩下数据的移动
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                w += size - r;
            }
            if (w != size) {
                // 清楚后面重复或无用的空间 clear to let GC do its work
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
        return modified;
    }
}
```

remove中removeAll()、retainAll()两个函数比较难一点，分别表示保留不在/在集合c中的元素

## get

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    ...
    public E get(int index) {
        rangeCheck(index);
        return elementData(index);
    }
}
```

get方法很简单，检查一下索引，直接获取

## set

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    ...
    public E set(int index, E element) {
        rangeCheck(index);
        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
}
```

set也没啥看的，跟get对应

## 其他方法

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    ...
    //去掉多余的空间（没有数据占用的空间）
    public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
    }
    //是否包含某个元素
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }
    //得到第一次出现索引
    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
    //最后依次出现的索引
    public int lastIndexOf(Object o) {
        if (o == null) {
            for (int i = size-1; i >= 0; i--)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = size-1; i >= 0; i--)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }

    public Object clone() {
        try {
            ArrayList<?> v = (ArrayList<?>) super.clone();
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError(e);
        }
    }
    //清空数组
    public void clear() {
        modCount++;
        for (int i = 0; i < size; i++)
            elementData[i] = null;
        size = 0;
    }

    //根据operator进行替换操作
    @Override
    @SuppressWarnings("unchecked")
    public void replaceAll(UnaryOperator<E> operator) {
        Objects.requireNonNull(operator);
        final int expectedModCount = modCount;
        final int size = this.size;
        for (int i=0; modCount == expectedModCount && i < size; i++) {
            elementData[i] = operator.apply((E) elementData[i]);
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
        modCount++;
    }

    //通过Arrays进行排序
    @Override
    @SuppressWarnings("unchecked")
    public void sort(Comparator<? super E> c) {
        final int expectedModCount = modCount;
        Arrays.sort((E[]) elementData, 0, size, c);
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
        modCount++;
    }
}
```

然后关于ArrayList的迭代器其实跟AbstrList中的差不多，跟LinkedList也差不多，大致思想都是一样的，需要的可以自己去看看

## 总结

ArrayList通过增、删、改、查可以看出，增和删的时候很容易要移动元素，性能消耗比较大，改、查就十分简单，根据索引就可以了，但是不知道索引还是需要遍历整个数组才行。相对于LinkedList改、查比较方便，但LinkedList增、删都由于ArrayList。

ArrayList有默认的容量为10，但并没有在构造器中进行初始化（除非指定容量的时候会初始化数组）。每次扩容的时候，是在原来size的基础上增加1/2，然后去跟最小需求容量比较