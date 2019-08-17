---
title: Java集合框架整理(七)——TreeSet、HashSet、LinkedHashSet源码分析
tag: Java

---

<meta name="referrer" content="no-referrer" />



[TOC]

# TreeSet、HashSet、LinkedHashSet源码分析

TreeSet是基于TreeMap实现的有序的集合，HashSet是基于HashMap实现的，而LinkedHashSet更是继承自HashSet，所以他们的源码都不会太复杂，所以在看这篇文章之前需要先看[**TreeMap源码分析**](http://blog.penghesheng.cn/2019/04/25/Java集合框架整理(七)——TreeSet、HashSet、LinkedHashSet源码分析/)、[**HashMap的源码分析**](http://blog.penghesheng.cn/2019/04/25/Java集合框架整理(七)——TreeSet、HashSet、LinkedHashSet源码分析/)

## TreeSet

> 有序的Set集合
>
> 基于TreeMap实现

TreeSet的继承关系

```java
public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable
{
    private transient NavigableMap<E,Object> m;
    private static final Object PRESENT = new Object();
    TreeSet(NavigableMap<E,Object> m) {
        this.m = m;
    }

    public TreeSet() {
        this(new TreeMap<E,Object>());
    }

    public TreeSet(Comparator<? super E> comparator) {
        this(new TreeMap<>(comparator));
    }

    public TreeSet(Collection<? extends E> c) {
        this();
        addAll(c);
    }

    public TreeSet(SortedSet<E> s) {
        this(s.comparator());
        addAll(s);
    }
    ...
}
```

TreeSet继承自AbstractSet，这是之前[**Java集合框架整理**](http://blog.penghesheng.cn/2019/04/25/Java集合框架整理(七)——TreeSet、HashSet、LinkedHashSet源码分析/)看过了。然后还实现了NavigableSet、Cloneable、Serializable接口，重点看看NavigableSet（另外两个就没啥说的了）

在看NavigableSet接口之前，先看看TreeSet中的全局变量和构造器吧。
m是一个NavigableMap，也是一个接口，具体实现是TreeMap（详见[TreeMap源码分析](http://blog.penghesheng.cn/2019/04/25/Java集合框架整理(七)——TreeSet、HashSet、LinkedHashSet源码分析/)），TreeSet很多方法最后都是调用TreeMap来实现的，TreeSet就是一个壳子。PRESENT就是一个假想值。通过观看几个重载的构造方法，最终是以一个TreeMap来初始化的，所以才说TreeSet是基于TreeMap实现的；Set是继承Collection的，所以也可以通过它来初始化，Collection衍生的类（如ArrayList、LinkedList等都有类似的构造方法）；TreeSet还可以通过SortSet来构造，SortSet也是一个接口，它是下一步要看的NavigableSet的父接口

接着看NavigableSet接口吧

```java
public interface NavigableSet<E> extends SortedSet<E> {
    //返回Set中小于e的最大元素
    E lower(E e);
    //返回Set中小于等于e的最大元素
    E floor(E e);
    //返回Set中大于等于e的最小元素
    E ceiling(E e);
    //返回Set中大于e的最小元素
    E higher(E e);
    //检索并移除Set中最小的元素（第一个）
    E pollFirst();
    //检索并移除Set中最大的元素（最后一个)
    E pollLast();
    //迭代器
    Iterator<E> iterator();
    //逆序的Set结合
    NavigableSet<E> descendingSet();
    //逆序的迭代器
    Iterator<E> descendingIterator();
    //批量处理，并发执行
    NavigableSet<E> subSet(E fromElement, boolean fromInclusive,
                           E toElement,   boolean toInclusive);
    //返回小于toElement的Set结合
    NavigableSet<E> headSet(E toElement, boolean inclusive);
    //返回大于fromElement的Set集合
    NavigableSet<E> tailSet(E fromElement, boolean inclusive);
    SortedSet<E> subSet(E fromElement, E toElement);
    SortedSet<E> headSet(E toElement);
    SortedSet<E> tailSet(E fromElement);
}
```

NavigableSet又是继承自SortedSet，NavigableSet中定义了一些操作方法，具体实现则在TreeSet中

接着看看SortedSet

```java
public interface SortedSet<E> extends Set<E> {
    //排序方式
    Comparator<? super E> comparator();

    SortedSet<E> subSet(E fromElement, E toElement);

    SortedSet<E> headSet(E toElement);

    SortedSet<E> tailSet(E fromElement);
    //获取排序后的Set中的第一个
    E first();
    //获取排序后的Set中的最后一个
    E last();

    @Override
    default Spliterator<E> spliterator() {
        return new Spliterators.IteratorSpliterator<E>(
                this, Spliterator.DISTINCT | Spliterator.SORTED | Spliterator.ORDERED) {
            @Override
            public Comparator<? super E> getComparator() {
                return SortedSet.this.comparator();
            }
        };
    }
}
```

SortedSet名如其意，就是排序的Set，是直接继承的Set接口，前面我们能知道Set基本跟Collection差不多，就是多了默认的分割迭代器。在SortedSet中添加了几个方法，便于一些常用的操作，如first()、last()分别得到当前Set中的第一个和最后一个元素；同时SortedSet中重写了spliterator()，修改了默认分割迭代器，得到的是可比较的分割迭代器，这个迭代器又是根据comparator()生成的，所以comparator()就需要看具体实现了；subSet()和subList()是一样的，集合操作，返回两个集合的交；headSet()返回小于toElement集合的所有元素组成的集合；同理，tailSet()返回大于等于fromElement；comparator()是定义排序方式的,这个接口中也没多少方法，提供了一些排序后获取元素的方法

所以不管是NavigableSet还是SortedSet，代码不是很多，都是定义了一些有序的操作方法，接着继续看TreeSet中的具体实现吧

```java
public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable
{
    ...
    //迭代器，返回的就是TreeMap的
    public Iterator<E> iterator() {
        return m.navigableKeySet().iterator();
    }
    //逆序迭代器
    public Iterator<E> descendingIterator() {
        return m.descendingKeySet().iterator();
    }

    public NavigableSet<E> descendingSet() {
        return new TreeSet<>(m.descendingMap());
    }

    public int size() {
        return m.size();
    }

    public boolean isEmpty() {
        return m.isEmpty();
    }

    public boolean contains(Object o) {
        return m.containsKey(o);
    }

    public boolean add(E e) {
        return m.put(e, PRESENT)==null;
    }

    public boolean remove(Object o) {
        return m.remove(o)==PRESENT;
    }

    public void clear() {
        m.clear();
    }

    public  boolean addAll(Collection<? extends E> c) {
        // Use linear-time version if applicable
        if (m.size()==0 && c.size() > 0 &&
            c instanceof SortedSet &&
            m instanceof TreeMap) {
            SortedSet<? extends E> set = (SortedSet<? extends E>) c;
            TreeMap<E,Object> map = (TreeMap<E, Object>) m;
            Comparator<?> cc = set.comparator();
            Comparator<? super E> mc = map.comparator();
            if (cc==mc || (cc != null && cc.equals(mc))) {
                map.addAllForTreeSet(set, PRESENT);
                return true;
            }
        }
        return super.addAll(c);
    }

    public NavigableSet<E> subSet(E fromElement, boolean fromInclusive,
                                  E toElement,   boolean toInclusive) {
        return new TreeSet<>(m.subMap(fromElement, fromInclusive,
                                       toElement,   toInclusive));
    }

    public NavigableSet<E> headSet(E toElement, boolean inclusive) {
        return new TreeSet<>(m.headMap(toElement, inclusive));
    }

    public NavigableSet<E> tailSet(E fromElement, boolean inclusive) {
        return new TreeSet<>(m.tailMap(fromElement, inclusive));
    }

    public SortedSet<E> subSet(E fromElement, E toElement) {
        return subSet(fromElement, true, toElement, false);
    }

    public SortedSet<E> headSet(E toElement) {
        return headSet(toElement, false);
    }

    public SortedSet<E> tailSet(E fromElement) {
        return tailSet(fromElement, true);
    }

    public Comparator<? super E> comparator() {
        return m.comparator();
    }

    public E first() {
        return m.firstKey();
    }

    public E last() {
        return m.lastKey();
    }

    // NavigableSet API methods
    public E lower(E e) {
        return m.lowerKey(e);
    }

    public E floor(E e) {
        return m.floorKey(e);
    }

    public E ceiling(E e) {
        return m.ceilingKey(e);
    }

    public E higher(E e) {
        return m.higherKey(e);
    }

    public E pollFirst() {
        Map.Entry<E,?> e = m.pollFirstEntry();
        return (e == null) ? null : e.getKey();
    }

    public E pollLast() {
        Map.Entry<E,?> e = m.pollLastEntry();
        return (e == null) ? null : e.getKey();
    }

    @SuppressWarnings("unchecked")
    public Object clone() {
        TreeSet<E> clone;
        try {
            clone = (TreeSet<E>) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new InternalError(e);
        }

        clone.m = new TreeMap<>(m);
        return clone;
    }

    //返回TreeMap的分割迭代器
    public Spliterator<E> spliterator() {
        return TreeMap.keySpliteratorFor(m);
    }
}
```

看着代码很多，其实仔细看看，很多方法都是基于TreeMap的，根本就没啥实际性的逻辑，所以一定要看完TreeMap的源码

## HashSet

HashSet是基于HashMap实现的，所以也不用多说什么，直接看看源码吧

```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
{
    static final long serialVersionUID = -5024744406713321676L;
    private transient HashMap<E,Object> map;
    private static final Object PRESENT = new Object();

    public HashSet() {
        map = new HashMap<>();
    }

    public HashSet(Collection<? extends E> c) {
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
    }

    public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }

    public HashSet(int initialCapacity) {
        map = new HashMap<>(initialCapacity);
    }

    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }

    public Iterator<E> iterator() {
        return map.keySet().iterator();
    }

    public int size() {
        return map.size();
    }

    public boolean isEmpty() {
        return map.isEmpty();
    }

    public boolean contains(Object o) {
        return map.containsKey(o);
    }

    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }

    public boolean remove(Object o) {
        return map.remove(o)==PRESENT;
    }

    public void clear() {
        map.clear();
    }

    @SuppressWarnings("unchecked")
    public Object clone() {
        try {
            HashSet<E> newSet = (HashSet<E>) super.clone();
            newSet.map = (HashMap<E, Object>) map.clone();
            return newSet;
        } catch (CloneNotSupportedException e) {
            throw new InternalError(e);
        }
    }
    ...
    public Spliterator<E> spliterator() {
        return new HashMap.KeySpliterator<E,Object>(map, 0, -1, 0, 0);
    }
}
```

通过源码可以知道，大多数api的调用都是交给了内部的HashMap去处理了，所以原理需要看看[**HashMap源码分析**](http://blog.penghesheng.cn/2019/04/25/Java集合框架整理(七)——TreeSet、HashSet、LinkedHashSet源码分析/)

## LinkedHashSet

LinkedHashSet是继承自HashSet的，而它本身也没有很多方法，所以依然是内部使用HashMap

```java
public class LinkedHashSet<E>
    extends HashSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {

    private static final long serialVersionUID = -2851667679971038690L;
    public LinkedHashSet(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor, true);
    }

    public LinkedHashSet(int initialCapacity) {
        super(initialCapacity, .75f, true);
    }

    public LinkedHashSet() {
        super(16, .75f, true);
    }

    public LinkedHashSet(Collection<? extends E> c) {
        super(Math.max(2*c.size(), 11), .75f, true);
        addAll(c);
    }

    @Override
    public Spliterator<E> spliterator() {
        return Spliterators.spliterator(this, Spliterator.DISTINCT | Spliterator.ORDERED);
    }
}
```

通过源码可以看到，没有多少方法，基本上就是HashSet，不同的就是实例化的时候，参数不同得到了LinkedHashSet，准确来说，LinkedHashSet就是HashSet的一种特殊情况

默认容量是16，加载因子是0.75，依然要满足HashMap的2次幂的要求

## 总结

TreeSet、HashSet分别基于TreeMap、HashMap实现，而LinkedHashMap是继承自HashSet，算是HashSet的一种特殊情况。看完源码，其实他们之间没有什么大的区别，最大的区别就是基于不同的map来实现的，他们本事实现了不同的接口也是因为特性提供了不同的api，更通俗一点，他们就是一个空壳，所以要知道原理还得看[**TreeMap源码分析**](http://blog.penghesheng.cn/2019/04/25/Java集合框架整理(七)——TreeSet、HashSet、LinkedHashSet源码分析/)、[**HashMap源码分析**](http://blog.penghesheng.cn/2019/04/25/Java集合框架整理(七)——TreeSet、HashSet、LinkedHashSet源码分析/)