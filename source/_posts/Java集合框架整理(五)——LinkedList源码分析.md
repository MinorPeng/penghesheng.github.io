---
title: Java集合框架整理(五)——LinkedList源码分析
tag: Java
---

# LinkedList源码分析

LinkedList：链表实现的集合

先看看类的继承关系

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    //链表长度
    transient int size = 0;
    //头结点
    transient Node<E> first;
    //尾节点
    transient Node<E> last;

    public LinkedList() {
    }

    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }
    ...
    //结点
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
}
```

LinkedList是通过AbstractSequentialList来实现的，并且同时实现了List、Deque、Cloneable、Serialzable等接口。List前面[**Java集合框架整理**](http://blog.penghesheng.cn/2019/04/25/Java集合框架整理(五)——LinkedList源码分析/)已经看过了，Deque则在[**Queue**中有讲到](http://blog.penghesheng.cn/2019/04/25/Java集合框架整理(五)——LinkedList源码分析/)，Cloneable和Serialzable表示可克隆和序列化，也没啥说的

接着可以看到有三个trasient修饰的全局变量表示不可序列化，然后是构造器。默认实现的是一个空链表，或者根据Collection生成的链表。然后我们看到链表结点定义的是一个私有静态内部类，只有一个构造方法，参数分别是元素，下一个结点和前一个结点（维护了双链表，既可前向遍历也可后向遍历），结点的引用可以让我们直接进行添加、删除操作而不用向数组一样需要进行其他元素的移动，但遍历就相对麻烦一些

接着看看常见的操作方法吧

------

## 添加add

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    ...
    //使用尾插法添加
    public boolean add(E e) {
        linkLast(e);
        return true;
    }
    //添加到指定位置
    public void add(int index, E element) {
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);  //添加到链尾
        else
            linkBefore(element, node(index));  //添加到某个结点前，通过node()获取改位置的结点
    }

    public boolean addAll(Collection<? extends E> c) {
        return addAll(size, c);
    }
    //将集合中的元素依次插入到指定位置
    public boolean addAll(int index, Collection<? extends E> c) {
        checkPositionIndex(index);
        //转为元素数组
        Object[] a = c.toArray();
        int numNew = a.length;
        if (numNew == 0)
            return false;

        Node<E> pred, succ;
        //整个集合插到链尾
        if (index == size) {
            succ = null;
            pred = last;
        } else {
            //指定结点后面
            succ = node(index);
            pred = succ.prev;
        }
        //遍历插入
        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            Node<E> newNode = new Node<>(pred, e, null);
            if (pred == null)
                first = newNode;
            else
                pred.next = newNode;
            pred = newNode;
        }

        if (succ == null) {
            last = pred;
        } else {
            pred.next = succ;
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }

    public void addFirst(E e) {
        linkFirst(e);
    }

    public void addLast(E e) {
        linkLast(e);
    }

    //头插法
    private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;
        if (f == null)
            last = newNode;
        else
            f.prev = newNode;
        size++;
        modCount++;
    }
    //尾插法
    //获取尾结点，生成一个新结点，如果没有尾结点，链表为空链的情况，那就成为头结点；如果有，则调整链尾，修改结点个数
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
    //插入，插入在某个结点前；通过节点的前指针pred来进行插入操作
    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }
    //通过比较size的一半，进行遍历查找，前向还是后向
    Node<E> node(int index) {
        //减小遍历次数
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }

    //来自Deque接口的方法
    public boolean offer(E e) {
        return add(e);
    }

    public boolean offerFirst(E e) {
        addFirst(e);
        return true;
    }

    public void push(E e) {
        addFirst(e);
    }
}
```

常用的add、addAll方法，通过链表的特性进行添加，由于也实现了Deque接口，所以一些方法逻辑就是基本一致

------

## 删除remove

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    ...
    //删除指定元素
    public boolean remove(Object o) {
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            //通过遍历查找删除
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }
    //删除指定位置的元素
    public E remove(int index) {
        checkElementIndex(index);
        return unlink(node(index));
    }
    //默认移除头结点
    public E remove() {
        return removeFirst();
    }
    //删除第一次出现的某个元素
    public boolean removeFirstOccurrence(Object o) {
        return remove(o);
    }
    //删除最后一次出现的元素，通过反响遍历
    public boolean removeLastOccurrence(Object o) {
        if (o == null) {
            for (Node<E> x = last; x != null; x = x.prev) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = last; x != null; x = x.prev) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }

    //删除头结点
    public E removeFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return unlinkFirst(f);
    }
    //删除尾结点
    public E removeLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return unlinkLast(l);
    }

    //删除第一个结点
    private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC
        first = next;
        if (next == null)
            last = null;
        else
            next.prev = null;
        size--;
        modCount++;
        return element;
    }
    //删除最后一个结点
    private E unlinkLast(Node<E> l) {
        // assert l == last && l != null;
        final E element = l.item;
        final Node<E> prev = l.prev;
        l.item = null;
        l.prev = null; // help GC
        last = prev;
        if (prev == null)
            first = null;
        else
            prev.next = null;
        size--;
        modCount++;
        return element;
    }

    public E poll() {
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
    }

    public E pollFirst() {
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
    }

    public E pollLast() {
        final Node<E> l = last;
        return (l == null) ? null : unlinkLast(l);
    }

    public E pop() {
        return removeFirst();
    }
}
```

remove也没啥难的，跟add基本对应类似

------

## get

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    ...
    public E get(int index) {
        checkElementIndex(index);  //检查下标
        return node(index).item;
    }

    //得到头结点
    public E getFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }
    //得到尾节点
    public E getLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return l.item;
    }

    public E peek() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
    }

    public E element() {
        return getFirst();
    }

    public E peekFirst() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
     }

     public E peekLast() {
        final Node<E> l = last;
        return (l == null) ? null : l.item;
    }
}
```

------

## set

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    ...
    //修改指定位置的值，返回旧值
    public E set(int index, E element) {
        checkElementIndex(index);
        Node<E> x = node(index);
        E oldVal = x.item;
        x.item = element;
        return oldVal;
    }
}
```

------

## 其他方法

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    ...
    //获取某个元素的位置。后向遍历
    public int indexOf(Object o) {
        int index = 0;
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null)
                    return index;
                index++;
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
    }
    //最后一次出现的位置
    public int lastIndexOf(Object o) {
        int index = size;
        if (o == null) {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (x.item == null)
                    return index;
            }
        } else {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (o.equals(x.item))
                    return index;
            }
        }
        return -1;
    }
    //通过比较是否下标判断是否存在某个元素
    public boolean contains(Object o) {
        return indexOf(o) != -1;
    }

    public int size() {
        return size;
    }
    //清空链表，遍历置null
    public void clear() {
        for (Node<E> x = first; x != null; ) {
            Node<E> next = x.next;
            x.item = null;
            x.next = null;
            x.prev = null;
            x = next;
        }
        first = last = null;
        size = 0;
        modCount++;
    }
    //拷贝。浅拷贝
    public Object clone() {
        LinkedList<E> clone = superClone();
        clone.first = clone.last = null;
        clone.size = 0;
        clone.modCount = 0;

        // Initialize clone with our elements
        for (Node<E> x = first; x != null; x = x.next)
            clone.add(x.item);
        return clone;
    }
    //转为元素数组
    public Object[] toArray() {
        Object[] result = new Object[size];
        int i = 0;
        for (Node<E> x = first; x != null; x = x.next)
            result[i++] = x.item;
        return result;
    }
}
```

然后看看迭代器

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    ...
    public ListIterator<E> listIterator(int index) {
        checkPositionIndex(index);
        return new ListItr(index);
    }

    public Iterator<E> descendingIterator() {
        return new DescendingIterator();
    }

    @Override
    public Spliterator<E> spliterator() {
        return new LLSpliterator<E>(this, -1, 0);
    }

}
```

通过index获取普通的迭代器，分割迭代器也是重写了。descendingIterator()提供的是一个逆序的迭代器，看源码可以知道就是以size()来生成的ListItr，next是当前结点的前一个结点

先看ListItr

```java
private class ListItr implements ListIterator<E> {
    private Node<E> lastReturned;
    private Node<E> next;
    private int nextIndex;
    private int expectedModCount = modCount;

    ListItr(int index) {
        // assert isPositionIndex(index);
        next = (index == size) ? null : node(index);
        nextIndex = index;
    }

    public boolean hasNext() {
        return nextIndex < size;
    }

    public E next() {
        checkForComodification();
        if (!hasNext())
            throw new NoSuchElementException();

        lastReturned = next;
        next = next.next;
        nextIndex++;
        return lastReturned.item;
    }

    public boolean hasPrevious() {
        return nextIndex > 0;
    }

    public E previous() {
        checkForComodification();
        if (!hasPrevious())
            throw new NoSuchElementException();

        lastReturned = next = (next == null) ? last : next.prev;
        nextIndex--;
        return lastReturned.item;
    }

    public int nextIndex() {
        return nextIndex;
    }

    public int previousIndex() {
        return nextIndex - 1;
    }

    public void remove() {
        checkForComodification();
        if (lastReturned == null)
            throw new IllegalStateException();

        Node<E> lastNext = lastReturned.next;
        unlink(lastReturned);
        if (next == lastReturned)
            next = lastNext;
        else
            nextIndex--;
        lastReturned = null;
        expectedModCount++;
    }

    public void set(E e) {
        ...
    }

    public void add(E e) {
        ...
    }

    public void forEachRemaining(Consumer<? super E> action) {
        ...
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

其实ListItr跟AbstractList中的差不多，核心思想都是间接来操作集合，这里就是间接操作链表了

再看LLSpliterator

```java
static final class LLSpliterator<E> implements Spliterator<E> {
    static final int BATCH_UNIT = 1 << 10;  // 分批每一批的size的增量
    static final int MAX_BATCH = 1 << 25;  // 每一批最大size
    final LinkedList<E> list;
    Node<E> current;      // 当前的结点
    int est;              // size估计值
    int expectedModCount; // est修改次数
    int batch;            // 分割每一批的数量size

    LLSpliterator(LinkedList<E> list, int est, int expectedModCount) {
        this.list = list;
        this.est = est;
        this.expectedModCount = expectedModCount;
    }

    final int getEst() {
        int s; // force initialization
        final LinkedList<E> lst;
        if ((s = est) < 0) {
            if ((lst = list) == null)
                s = est = 0;
            else {
                expectedModCount = lst.modCount;
                current = lst.first;
                s = est = lst.size;
            }
        }
        return s;
    }
    //估计当前Spliterator实例中将要迭代的元素的数量，如果数量是无限的、未知的或者计算数量的花销太大，则返回Long.MAX_VALUE
    public long estimateSize() { return (long) getEst(); }
    //分割迭代器，没调用一次，将原来的迭代器等分为两份，并返回索引靠前的那一个子迭代器
    public Spliterator<E> trySplit() {
        Node<E> p;
        int s = getEst();
        if (s > 1 && (p = current) != null) {
            int n = batch + BATCH_UNIT;
            if (n > s)
                n = s;
            if (n > MAX_BATCH)
                n = MAX_BATCH;
            Object[] a = new Object[n];
            int j = 0;
            do { a[j++] = p.item; } while ((p = p.next) != null && j < n);
            current = p;
            batch = j;
            est = s - j;
            return Spliterators.spliterator(a, 0, j, Spliterator.ORDERED);
        }
        return null;
    }
    //通过action批量消费所有的未迭代的数据。
    public void forEachRemaining(Consumer<? super E> action) {
        Node<E> p; int n;
        if (action == null) throw new NullPointerException();
        if ((n = getEst()) > 0 && (p = current) != null) {
            current = null;
            est = 0;
            do {
                E e = p.item;
                p = p.next;
                action.accept(e);
            } while (p != null && --n > 0);
        }
        if (list.modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
    //单个对元素执行给定的动作，如果有剩下元素未处理返回true，否则返回false
    public boolean tryAdvance(Consumer<? super E> action) {
        Node<E> p;
        if (action == null) throw new NullPointerException();
        if (getEst() > 0 && (p = current) != null) {
            --est;
            E e = p.item;
            current = p.next;
            action.accept(e);
            if (list.modCount != expectedModCount)
                throw new ConcurrentModificationException();
            return true;
        }
        return false;
    }
    //返回当前对象有哪些特征值
    public int characteristics() {
        return Spliterator.ORDERED | Spliterator.SIZED | Spliterator.SUBSIZED;
    }
}
```

这里就简单看看Spliterator，具体的请自行查看有关Spliterator的知识
[java源码阅读之Spliterator](https://blog.csdn.net/jiangmingzhi23/article/details/78927552)
[JDK8源码之Spliterator并行遍历迭代器](https://blog.csdn.net/lh513828570/article/details/56673804)

## 总结

LinkedList就是充分利用链表这个数据结构的特定，实现的集合，在1.8加入了分割迭代器用于并发处理和遍历，优化性能。同时在查找某个结点的时候，也是优化算法为O(n/2)的时间复杂度；链表的空间复杂度并不复杂，唯一的缺点就是遍历的问题比较麻烦；LinkedList采用的是双链表的形式，在遍历上比单链表效率又要高许多。还有点需要注意的是LinkedList并没有实现List接口中的replaceAll()和sort()方法，而ArrayList实现了