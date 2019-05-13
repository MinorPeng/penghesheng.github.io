---
title: Java集合框架整理(一)
tag: Java
---

# Java Map、List、Vector源码分析

[脑图](http://naotu.baidu.com/file/195a5d4c7010b778a85631f5f61ab92c?token=f5c0ec8bbaa0f24e)

[![Java集合框架脑图](https://upload-images.jianshu.io/upload_images/4061843-77301c3b14d67330.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://upload-images.jianshu.io/upload_images/4061843-77301c3b14d67330.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)Java集合框架脑图

Java集合大致分为Set、List、Queue、Map四类。

> Set：无序、不可重复的集合
> List：有序、可重复的集合
> Queue：队列集合
> Map：有映射关系的集合

而Java集合最根本的接口主要是**Collection**和**Map**，前三个都是通过Collection接口实现

## Collection

[![Collection](https://img-blog.csdnimg.cn/20190116164907870.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWR1XzM2OTU5ODg2,size_16,color_FFFFFF,t_70)](https://img-blog.csdnimg.cn/20190116164907870.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWR1XzM2OTU5ODg2,size_16,color_FFFFFF,t_70)Collection

```java
public interface Collection<E> extends Iterable<E> {
    //元素数
    int size();
    boolean isEmpty();
    //是否包含某个元素
    boolean contains(Object o);
    //该Collection的迭代器
    Iterator<E> iterator();
    Object[] toArray();
    <T> T[] toArray(T[] a);
    boolean add(E e);
    boolean remove(Object o);
    //是否包含某个元素集合Collection
    boolean containsAll(Collection<?> c);
    boolean addAll(Collection<? extends E> c);
    boolean removeAll(Collection<?> c);
    default boolean removeIf(Predicate<? super E> filter) {
        Objects.requireNonNull(filter);
        boolean removed = false;
        final Iterator<E> each = iterator();
        while (each.hasNext()) {
            if (filter.test(each.next())) {
                each.remove();
                removed = true;
            }
        }
        return removed;
    }
    //保留当前Collection和c两个的交（共有的元素）
    boolean retainAll(Collection<?> c);
    void clear();
    boolean equals(Object o);
    int hashCode();
    default Spliterator<E> spliterator() {
        return Spliterators.spliterator(this, 0);
    }

    default Stream<E> stream() {
        return StreamSupport.stream(spliterator(), false);
    }

    default Stream<E> parallelStream() {
        return StreamSupport.stream(spliterator(), true);
    }
}
```

Collection首先继承了Iterable，看Iterable的源码中注释:这个接口是提供了”for-each loop”类似的迭代功能，得到迭代器（Iterator）实现迭代遍历，而Iterator是一种统一遍历集合元素的方式，所以所有实现了Collection的都会默认有一个迭代器，Iterable是为了得到相应的迭代器（每个集合最后迭代的方式可能不一样），接着，Collection中定义了一些常用的方法

```java
public interface Iterable<T> {
    //返回该Collection上的Iterator
    Iterator<T> iterator();

    //保证通过指定的action（应该是迭代方式）进行遍历完全
    default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }

    //分割迭代器，用于并行处理的
    default Spliterator<T> spliterator() {
        return Spliterators.spliteratorUnknownSize(iterator(), 0);
    }
}

//这是Iterator的代码，相当简单了
public interface Iterator<E> {
    boolean hasNext();

    E next();

    default void remove() {
        throw new UnsupportedOperationException("remove");
    }

    default void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        while (hasNext())
            action.accept(next());
    }
}
```

Collection的代码看完了，接着看看它的实现和相关的接口吧。
从脑图中可以看到，继承于Collection的接口有List、Set、Queue，这三个是直接继承的，然后还有一个抽象类AbstractCollection实现了Collection。

```java
public abstract class AbstractCollection<E> implements Collection<E> {
    protected AbstractCollection() {
    }

    public abstract Iterator<E> iterator();
    public abstract int size();
    public boolean isEmpty() {
        return size() == 0;
    }

    //通过迭代器，遍历判断是否包含
    public boolean contains(Object o) {
        Iterator<E> it = iterator();
        if (o==null) {
            while (it.hasNext())
                if (it.next()==null)
                    return true;
        } else {
            while (it.hasNext())
                if (o.equals(it.next()))
                    return true;
        }
        return false;
    }

    public Object[] toArray() {
        Object[] r = new Object[size()];
        Iterator<E> it = iterator();
        for (int i = 0; i < r.length; i++) {
            if (! it.hasNext()) // fewer elements than expected
                return Arrays.copyOf(r, i);
            r[i] = it.next();
        }
        return it.hasNext() ? finishToArray(r, it) : r;
    }

    @SuppressWarnings("unchecked")
    public <T> T[] toArray(T[] a) {
        int size = size();
        T[] r = a.length >= size ? a :
                  (T[])java.lang.reflect.Array
                  .newInstance(a.getClass().getComponentType(), size);
        Iterator<E> it = iterator();

        for (int i = 0; i < r.length; i++) {
            if (! it.hasNext()) { // fewer elements than expected
                if (a == r) {
                    r[i] = null; // null-terminate
                } else if (a.length < i) {
                    return Arrays.copyOf(r, i);
                } else {
                    System.arraycopy(r, 0, a, 0, i);
                    if (a.length > i) {
                        a[i] = null;
                    }
                }
                return a;
            }
            r[i] = (T)it.next();
        }
        // more elements than expected
        return it.hasNext() ? finishToArray(r, it) : r;
    }

    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    @SuppressWarnings("unchecked")
    private static <T> T[] finishToArray(T[] r, Iterator<?> it) {
        ...
    }

    public boolean add(E e) {
        throw new UnsupportedOperationException();
    }

    public boolean remove(Object o) {
        Iterator<E> it = iterator();
        if (o==null) {
            while (it.hasNext()) {
                if (it.next()==null) {
                    it.remove();
                    return true;
                }
            }
        } else {
            while (it.hasNext()) {
                if (o.equals(it.next())) {
                    it.remove();
                    return true;
                }
            }
        }
        return false;
    }

    public boolean containsAll(Collection<?> c) {
        ...
    }

    public boolean addAll(Collection<? extends E> c) {
        ...
    }

    public boolean removeAll(Collection<?> c) {
        ...
    }

    public boolean retainAll(Collection<?> c) {
        ...
    }

    public void clear() {
        Iterator<E> it = iterator();
        while (it.hasNext()) {
            it.next();
            it.remove();
        }
    }

    public String toString() {
        Iterator<E> it = iterator();
        if (! it.hasNext())
            return "[]";

        StringBuilder sb = new StringBuilder();
        sb.append('[');
        for (;;) {
            E e = it.next();
            sb.append(e == this ? "(this Collection)" : e);
            if (! it.hasNext())
                return sb.append(']').toString();
            sb.append(',').append(' ');
        }
    }
}
```

在这个抽象类中，对之前定义的方法进行了处理和实现，添加了逻辑处理。
通过迭代器遍历，判断是否包含某个元素（contains）；转换为array（toArray）和指定的数组（toArray(T[] a)），定义了最大容量为Integer.MAX_VALUE - 8；删除处理（remove）；清空集合处理（clear）；toString重写。

AbstractCollection这个抽象类的子类又根据实现不同的接口（List、Set、Queue）得到AbstractList、AbstractSet、AbstractQueue这几个分别对应的抽象类，然后常用的ArrayList、Vector、HashSet、PriorityQueue、DelayQueue等都是直接或间接继承这几个对应的抽象类来实现的。

### List

```java
public interface List<E> extends Collection<E> {
    int size();
    boolean isEmpty();
    boolean contains(Object o);
    Iterator<E> iterator();
    Object[] toArray();
    <T> T[] toArray(T[] a);
    boolean add(E e);
    boolean remove(Object o);
    boolean containsAll(Collection<?> c);
    boolean addAll(Collection<? extends E> c);
    boolean addAll(int index, Collection<? extends E> c);
    boolean removeAll(Collection<?> c);
    boolean retainAll(Collection<?> c);
    default void replaceAll(UnaryOperator<E> operator) {
        Objects.requireNonNull(operator);
        final ListIterator<E> li = this.listIterator();
        while (li.hasNext()) {
            li.set(operator.apply(li.next()));
        }
    }

    @SuppressWarnings({"unchecked", "rawtypes"})
    default void sort(Comparator<? super E> c) {
        Object[] a = this.toArray();
        Arrays.sort(a, (Comparator) c);
        ListIterator<E> i = this.listIterator();
        for (Object e : a) {
            i.next();
            i.set((E) e);
        }
    }
    void clear();
    boolean equals(Object o);
    int hashCode();
    E get(int index);
    E set(int index, E element);
    void add(int index, E element);
    E remove(int index);
    int indexOf(Object o);
    int lastIndexOf(Object o);
    ListIterator<E> listIterator();
    ListIterator<E> listIterator(int index);
    List<E> subList(int fromIndex, int toIndex);
    @Override
    default Spliterator<E> spliterator() {
        return Spliterators.spliterator(this, Spliterator.ORDERED);
    }
}
```

List直接继承自Collection，然后定义了很多方法，其中有一部分跟父接口Collecion中是一样的；然后List有定义了一些适合自己的方法（如get、set等），还添加了一个Sort方法用于排序，比Collection多了一种为List定制的ListIterator

总之，List的源码也没啥看的，也很简单；看了List的源码，后面看Set、Queue的源码都差不多，因为都是继承自Collection，然后根据自己的特性，添加了属于自己的方法和迭代器。这些特性呢，其实就跟学的数据结构一个道理，只不过Java封装的更好、更完善

看看AbstractList吧

```java
public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E> {
    protected AbstractList() {
    }

    public boolean add(E e) {
        add(size(), e);
        return true;
    }

    abstract public E get(int index);

    //总是会抛出这个异常
    public E set(int index, E element) {
        throw new UnsupportedOperationException();
    }

    public void add(int index, E element) {
        throw new UnsupportedOperationException();
    }

    public E remove(int index) {
        throw new UnsupportedOperationException();
    }

    //通过迭代遍历，找到下标，从0开始
    public int indexOf(Object o) {
        ListIterator<E> it = listIterator();
        if (o==null) {
            while (it.hasNext())
                if (it.next()==null)
                    return it.previousIndex();
        } else {
            while (it.hasNext())
                if (o.equals(it.next()))
                    return it.previousIndex();
        }
        return -1;
    }

    //最后一次出现的下标，从最后一个开始，向前遍历查找
    public int lastIndexOf(Object o) {
        ...
    }

    public void clear() {
        //通过迭代器remove
        ...
    }

    public boolean addAll(int index, Collection<? extends E> c) {
        ...
    }

    public Iterator<E> iterator() {
        return new Itr();
    }

    //ListIterator，默认0开始迭代
    public ListIterator<E> listIterator() {
        return listIterator(0);
    }

    //从某个下标开始的迭代
    public ListIterator<E> listIterator(final int index) {
        rangeCheckForAdd(index);
        return new ListItr(index);
    }

    //迭代器实现类
    private class Itr implements Iterator<E> {
        ...
    }

    //ListIterator实现类，nextIndex就是当前cursor，previousIndex，就是cursor-1
    private class ListItr extends Itr implements ListIterator<E> {
        ListItr(int index) {
            cursor = index;
        }
        ...
    }

    //取集合中的某一部分子集来进行操作，是直接在原数据上进行操作
    public List<E> subList(int fromIndex, int toIndex) {
        ...
    }

    //两个AbstractListd的比较，同时迭代
    public boolean equals(Object o) {
        ...
        ListIterator<E> e1 = listIterator();
        ListIterator<?> e2 = ((List<?>) o).listIterator();
        while (e1.hasNext() && e2.hasNext()) {
            E o1 = e1.next();
            Object o2 = e2.next();
            if (!(o1==null ? o2==null : o1.equals(o2)))
                return false;
        }
        return !(e1.hasNext() || e2.hasNext());
    }

    public int hashCode() {
        int hashCode = 1;
        for (E e : this)
            hashCode = 31*hashCode + (e==null ? 0 : e.hashCode());
        return hashCode;
    }
    ...
}

//subList实现类
class SubList<E> extends AbstractList<E> {
    private final AbstractList<E> l;
    private final int offset;
    private int size;

    SubList(AbstractList<E> list, int fromIndex, int toIndex) {
        ...
    }

    ...
}

//随机访问，通过它可以间接操控父类
class RandomAccessSubList<E> extends SubList<E> implements RandomAccess {
    ...
}
```

代码好像有点多，怎么说呢，AbstractList基本囊括了所有的List操作，基本上平时用到的大多数都在这个源码里面，当然get方法还待实现，后面的ArrayList、Vector、Stack就是根据相应的进行特殊的操作了

下面看看常用的List吧，都继承于AbstractList这个抽象类：

- LinkedList
    LinkedList是继承自AbstractSequenceList这个抽象类，而AbstractSequenceList又是继承自AbstractList的
- ArrayList
    1. 动态数组，线程非安全，允许null
    2. 可以手动调用扩容
    3. 默认是空数组构造，默认容量是10
    4. 扩容默认扩容1.5倍
    5. add通过移动数组，remove也是移动数组进行覆盖，数组的拷贝是通过native方法（System.arraycopy）
    6. clear操作就是置为null
- SparseArray
    1. int型的key，Object型的value，HashMap则是任意类型
    2. 一个数组存放key，一个数组存放value；key是有序的
    3. 不再使用像HashMap的Entry
    4. 只实现了Cloneable接口，默认容量是10
    5. 通过二分查找法来查找插入的下标index，如果返回的是负值表明该位置没有被占用，可以直接取反获取下标
    6. 删除数据后不会直接移动数据进行覆盖，而是将删除的位置标记为DELETE，标明当前位置没有数据，那么在put时，有原值直接覆盖且不会返回，HashMap会返回原值，发现要插入的位置是DELETE，就直接插入，但是如果发现该位置已经有值了（key值不一样），那么触发gc，调整其他位置的DELETE并删除无用的key，重新计算下标再进行插入；如果容量已经满了，则扩容后插入
    7. 扩容时，当前容量小于等于4，则扩容后容量为8.否则为当前容量的两倍
- Vector
    1. 默认容量10，增长系数0，扩容翻倍
    2. 通过synchronized 来保证线程安全
    3. 允许元素为null
- Stack
    继承自Vector

### Set

```java
public interface Set<E> extends Collection<E> {
    int size();
    boolean isEmpty();
    boolean contains(Object o);
    Iterator<E> iterator();
    Object[] toArray();
    <T> T[] toArray(T[] a);
    boolean add(E e);
    boolean remove(Object o);
    boolean containsAll(Collection<?> c);
    boolean addAll(Collection<? extends E> c);
    boolean retainAll(Collection<?> c);
    boolean removeAll(Collection<?> c);
    void clear();
    boolean equals(Object o);
    int hashCode();
    @Override
    default Spliterator<E> spliterator() {
        return Spliterators.spliterator(this, Spliterator.DISTINCT);
    }
}
```

咋一看，好像跟Collection没啥区别，就是多了默认的分割迭代器。因为特性的原因，方法比List少了一部分（如sort、get、set、listIterator、subList等等）

看看AbstractSet

```java
public abstract class AbstractSet<E> extends AbstractCollection<E> implements Set<E> {
    protected AbstractSet() {
    }

    public boolean equals(Object o) {
        if (o == this)
            return true;

        if (!(o instanceof Set))
            return false;
        Collection<?> c = (Collection<?>) o;
        if (c.size() != size())
            return false;
        try {
            return containsAll(c);
        } catch (ClassCastException unused)   {
            return false;
        } catch (NullPointerException unused) {
            return false;
        }
    }

    public int hashCode() {
        int h = 0;
        Iterator<E> i = iterator();
        while (i.hasNext()) {
            E obj = i.next();
            if (obj != null)
                h += obj.hashCode();
        }
        return h;
    }

    public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);
        boolean modified = false;
        //以少的为循环条件
        if (size() > c.size()) {
            for (Iterator<?> i = c.iterator(); i.hasNext(); )
                modified |= remove(i.next());
        } else {
            for (Iterator<?> i = iterator(); i.hasNext(); ) {
                if (c.contains(i.next())) {
                    i.remove();
                    modified = true;
                }
            }
        }
        return modified;
    }
}
```

这个代码好像也没啥，就对equals、hashCode进行实现，removeAll修改了一下，以元素少的为遍历条件进行删除

- HashSet
    1. 基于HashMap实现，底层保存数据使用HashMap
    2. 有内部排序，当重复的时候不会放入内部的HashMap
- LinkedHashSet
- TreeSet

### Queue

```java
public interface Queue<E> extends Collection<E> {
    boolean add(E e);
    boolean offer(E e);
    E remove();
    E poll();
    E element();
    E peek();
}
```

你没看错，Queue的代码就这么点；没有去再给自己一个迭代器，用的就是Collection的，多了offer、poll、element、peek等方法

就直接看看AbstractQueue这个抽象类吧

```java
public abstract class AbstractQueue<E>
    extends AbstractCollection<E>
    implements Queue<E> {

    protected AbstractQueue() {
    }

    public boolean add(E e) {
        if (offer(e))
            return true;
        else
            throw new IllegalStateException("Queue full");
    }

    public E remove() {
        E x = poll();
        if (x != null)
            return x;
        else
            throw new NoSuchElementException();
    }

    public E element() {
        E x = peek();
        if (x != null)
            return x;
        else
            throw new NoSuchElementException();
    }

    public void clear() {
        while (poll() != null)
            ;
    }

    public boolean addAll(Collection<? extends E> c) {
        if (c == null)
            throw new NullPointerException();
        if (c == this)
            throw new IllegalArgumentException();
        boolean modified = false;
        for (E e : c)
            if (add(e))
                modified = true;
        return modified;
    }
}
```

代码也是很少，add方法最终是通过offer来处理的，remove就是通过poll来处理的，element是通过peek方法来处理的，clear也是通过poll来处理的，这些方法都是需要后面实现的。

接着看看几种Queue和Deque常用类吧

- 线程安全的Queue
    ConcurrentLinkedDeque、ConcurrentLinkedQueue、LinkedBlockingQueue、LinkedBlockingDeque、PriorityBlockingQueue、ArrayBlockingQueue
- 线程非安全的Queue
    PriorityQueue、DelayQueue、LinkedTransferQueue

## Map

[![Map](https://img-blog.csdnimg.cn/20190116165243432.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWR1XzM2OTU5ODg2,size_16,color_FFFFFF,t_70)](https://img-blog.csdnimg.cn/20190116165243432.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWR1XzM2OTU5ODg2,size_16,color_FFFFFF,t_70)Map

> Map作为另一个大类的集合，我们用的也不算少
> Map是一种数据映射关系的集合，是以键值对的形式存储（key-value），其中key不可重复

首先看看Map这个根接口

```java
public interface Map<K,V> {
    int size();  //元素个数
    boolean isEmpty();  //是否为空集合
    boolean containsKey(Object key);  //是否包含某个键key
    boolean containsValue(Object value);  //是否包含某个值
    V get(Object key);  //获取
    V put(K key, V value);  //存入一个元素
    V remove(Object key);  //根据键删除某个元素
    void putAll(Map<? extends K, ? extends V> m);  //添加一个map集合
    void clear();  //清空
    Set<K> keySet();  //返回key的Set集合
    Collection<V> values();  //返回value的Collection集合
    Set<Map.Entry<K, V>> entrySet();  //返回Entry（具有键值对）的Set集合
    //Map内部的一个接口，同时存有key和value，用于entrySet返回
    interface Entry<K,V> {
        K getKey();
        V getValue();
        V setValue(V value);
        boolean equals(Object o);
        int hashCode();

        public static <K extends Comparable<? super K>, V> Comparator<Map.Entry<K,V>> comparingByKey() {
            return (Comparator<Map.Entry<K, V>> & Serializable)
                (c1, c2) -> c1.getKey().compareTo(c2.getKey());
        }

        public static <K, V extends Comparable<? super V>> Comparator<Map.Entry<K,V>> comparingByValue() {
            return (Comparator<Map.Entry<K, V>> & Serializable)
                (c1, c2) -> c1.getValue().compareTo(c2.getValue());
        }

        public static <K, V> Comparator<Map.Entry<K, V>> comparingByKey(Comparator<? super K> cmp) {
            Objects.requireNonNull(cmp);
            return (Comparator<Map.Entry<K, V>> & Serializable)
                (c1, c2) -> cmp.compare(c1.getKey(), c2.getKey());
        }

        public static <K, V> Comparator<Map.Entry<K, V>> comparingByValue(Comparator<? super V> cmp) {
            Objects.requireNonNull(cmp);
            return (Comparator<Map.Entry<K, V>> & Serializable)
                (c1, c2) -> cmp.compare(c1.getValue(), c2.getValue());
        }
    }

    boolean equals(Object o);
    int hashCode();  //hash值
    //根据key获取value默认要指定一个默认值，防止获取不存在的值（不存在的key）。貌似是后来加上的
    default V getOrDefault(Object key, V defaultValue) {
        V v;
        return (((v = get(key)) != null) || containsKey(key))
            ? v
            : defaultValue;
    }
    //根据action默认的foreach，key和value都会参与
    default void forEach(BiConsumer<? super K, ? super V> action) {
        Objects.requireNonNull(action);
        for (Map.Entry<K, V> entry : entrySet()) {
            K k;
            V v;
            try {
                k = entry.getKey();
                v = entry.getValue();
            } catch(IllegalStateException ise) {
                // this usually means the entry is no longer in the map.
                throw new ConcurrentModificationException(ise);
            }
            action.accept(k, v);
        }
    }
    //通过遍历，然后根据function来替换原有的值
    default void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) {
        Objects.requireNonNull(function);
        for (Map.Entry<K, V> entry : entrySet()) {
            K k;
            V v;
            try {
                k = entry.getKey();
                v = entry.getValue();
            } catch(IllegalStateException ise) {
                // this usually means the entry is no longer in the map.
                throw new ConcurrentModificationException(ise);
            }

            // ise thrown from function is not a cme.
            v = function.apply(k, v);

            try {
                entry.setValue(v);
            } catch(IllegalStateException ise) {
                // this usually means the entry is no longer in the map.
                throw new ConcurrentModificationException(ise);
            }
        }
    }
    //加入
    default V putIfAbsent(K key, V value) {
        V v = get(key);
        if (v == null) {
            v = put(key, value);
        }

        return v;
    }
    //根据键值对删除指定的元素
    default boolean remove(Object key, Object value) {
        Object curValue = get(key);
        if (!Objects.equals(curValue, value) ||
            (curValue == null && !containsKey(key))) {
            return false;
        }
        remove(key);
        return true;
    }
    //替换，最终是通过put实现
    default boolean replace(K key, V oldValue, V newValue) {
        Object curValue = get(key);
        if (!Objects.equals(curValue, oldValue) ||
            (curValue == null && !containsKey(key))) {
            return false;
        }
        put(key, newValue);
        return true;
    }

    default V replace(K key, V value) {
        V curValue;
        if (((curValue = get(key)) != null) || containsKey(key)) {
            curValue = put(key, value);
        }
        return curValue;
    }

    default V computeIfAbsent(K key,
            Function<? super K, ? extends V> mappingFunction) {
        Objects.requireNonNull(mappingFunction);
        V v;
        if ((v = get(key)) == null) {
            V newValue;
            if ((newValue = mappingFunction.apply(key)) != null) {
                put(key, newValue);
                return newValue;
            }
        }

        return v;
    }

    default V computeIfPresent(K key,
            BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
        Objects.requireNonNull(remappingFunction);
        V oldValue;
        if ((oldValue = get(key)) != null) {
            V newValue = remappingFunction.apply(key, oldValue);
            if (newValue != null) {
                put(key, newValue);
                return newValue;
            } else {
                remove(key);
                return null;
            }
        } else {
            return null;
        }
    }

    default V compute(K key,
            BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
        Objects.requireNonNull(remappingFunction);
        V oldValue = get(key);

        V newValue = remappingFunction.apply(key, oldValue);
        if (newValue == null) {
            // delete mapping
            if (oldValue != null || containsKey(key)) {
                // something to remove
                remove(key);
                return null;
            } else {
                // nothing to do. Leave things as they were.
                return null;
            }
        } else {
            // add or replace old mapping
            put(key, newValue);
            return newValue;
        }
    }

    default V merge(K key, V value,
            BiFunction<? super V, ? super V, ? extends V> remappingFunction) {
        Objects.requireNonNull(remappingFunction);
        Objects.requireNonNull(value);
        V oldValue = get(key);
        V newValue = (oldValue == null) ? value :
                   remappingFunction.apply(oldValue, value);
        if(newValue == null) {
            remove(key);
        } else {
            put(key, newValue);
        }
        return newValue;
    }
}
```

### HashMap

#### 1.7和1.8的区别

1. 1.7的一些运算赋值在1.8中变为了位运算，一些判断也从Math变为了if else
2. JDK 1.7 做了9次扰动处理 = 4次位运算 + 5次异或运算。1.8 简化了扰动函数 = 只做了2次扰动 = 1次位运算 + 1次异或运算
3. 1.7扩容函数是在确定要扩容进行，1.8的扩容函数可能是初始化；1.7更新阈值在扩容后，1.8在扩容前，很多判断、阈值更新都是在扩容前
4. 1.8扩容后进行复制时，会先进行链表的一个判断（当前链表只有一个元素时），1.7没有；1.8有链表高低位的处理，是将整个旧链表复制到新的中，低位链表的索引跟原来的一样，高位的则放在新的索引处（尾插法），1.7则是遍历旧链表，头插法插入新链表（可能导致逆序）
5. 1.8中加入了红黑树，当链表长度到8时链表转化为红黑树，扩容时，重新计算存储位置，红黑树内数量<6又会转化为链表
6. key和value都允许为null（key只能有一个为null，而value则可以有多个为null）

### ConcurrentHashMap

1. 1.6使用ReentrantLock保证并发安全，1.8使用CAS（Compare and Swipe）和synchronized 来保证并发安全
2. 1.8有红黑树的概念，使用红黑树，这些跟1.8的HashMap一样
3. 由于并发的问题，在扩容的时候是采用单线程初始化大小，多个线程共同进行值的复制
4. 1.6头插法，1.8逆序插入链表
5. 1.6删除操作要重新复制一次链表，所以1.7并不绝对并发安全（删除没有完全的时候旧链表存在）
6. 1.6next是final修饰，1.8不用

### LinkedHashMap

1. 数组+哈希表，非线程安全
2. key、value允许为null
3. 继承自HashMap，有一个双向链表
4. 相比HashMap，增加了accessOrder参数控制迭代时的结点顺序
5. 增删数据，都是调用的HashMap的函数，只是修改了一些链头链尾指向的结点
6. accessOrder=true的模式下,在afterNodeAccess()函数中，会将当前被访问到的节点e，移动至内部的双向链表的尾部。值得注意的是，afterNodeAccess()函数中，会修改modCount,因此当你正在accessOrder=true的模式下,迭代LinkedHashMap时，如果同时查询访问数据，也会导致fail-fast，因为迭代的顺序已经改变

### HashTable

1. 默认容量是11，不要求底层数组的容量一定要为2的整数次幂
2. 扩容后重新计算index 求余（%），扩容是2倍+1
3. clear操作仅仅是将table（数组）设为null
4. 每一个方法都是用synchronized 修饰来保证线程安全
5. Hashtable中key和value都不允许为null
6. 计算hash值，直接用key的hashCode()
7. 拉链法，hashSeed ^ k.hashCode()解决Hash冲突
8. 所谓快速失败就是在并发集合中，其进行迭代操作时，若有其他线程对其进行结构性的修改，这时迭代器会立马感知到，并且立即抛出ConcurrentModificationException 异常，而不是等到迭代完成之后才告诉你（你已经出错了）。

### TreeMap

1. TreeMap是根据key进行排序的，它的排序和定位需要依赖比较器或覆写Comparable接口
2. key不能为null
3. 就是红黑树实现
4. 非线程同步

### ArrayMap

1. 两个数组，int型存放hashcod，Object型保存键值对，int型的2倍容量
2. 扩容时只拷贝数组，不重建hash表
3. 删除时，如果集合剩余元素少于一定阈值，数组会收缩（容量减小）
4. 对key从小到大排序，二分法查询key的下标
5. 默认扩容size是4，默认容量为0
6. put操作出来计算hashcode外，其余跟SparseArray基本类似，也类似于ArrayList
7. 扩容规则：如果容量大于8，则扩容一半。
8. mHashes长度大于8，且 集合长度 小于当前空间的 1/3,则执行一个 shrunk，收缩操作