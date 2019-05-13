---
title: Java集合框架整理(二)——HashMap源码分析
tag: Java
date: 2019-05-06
---

# HashMap源码分析（1.8）

## 简介

> 1. **HashMap**是一个键值对的集合容器，继承自Map
> 2. 是线程不安全的（线程安全的是ConcurrentHashMap），允许key为null，value为null
> 3. 是无序集合，遍历时是无序的
> 4. 是一个关联数组、哈希表，内部的数据结构是一个数组，称之为哈希桶，每个桶里放的是一个链表，链表的节点就是最后存放的元素（1.8 当链表的节点数大于8，链表转为红黑树，节点数小于6，红黑树转为链表）
> 5. 开放地址法，解决哈希冲突，具体体现在上一点的数据结构实现上
> 6. 默认初始桶长16，默认加载因子0.75，最大容量2^30，在put的时候才会去初始化桶
> 7. 每次扩容是以前的2倍，通过new一个新的数组，然后拷贝值
> 8. 通过**扰动函数**来减少哈希碰撞（1.8 2次扰动，1.7 9次扰动）

下面就是HashMap的数据结构

[![HashMap](https://img-blog.csdnimg.cn/20190506232217775.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWR1XzM2OTU5ODg2,size_16,color_FFFFFF,t_70)](https://img-blog.csdnimg.cn/20190506232217775.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWR1XzM2OTU5ODg2,size_16,color_FFFFFF,t_70)HashMap

接着从源码角度来看看HashMap吧

首先看看HashMap继承结构

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    ...
}
```

HashMap是继承自AbstractMap的，AbstractMap是实现了Map的（具体可以看看[Java集合框架整理(一)](http://blog.penghesheng.cn/2019/05/06/Java集合框架整理(二)——HashMap源码分析/)），同时自己也实现了Map、Cloneable（可复制）、Serializable（可序列化）等接口

接着看看一些内部属性吧

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    //默认桶初始化容量 16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
	//最大桶容量 2^31-1
    static final int MAXIMUM_CAPACITY = 1 << 30;
	//默认加载因子 0.75
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
	//当链表节点数大于8转为红黑树的阈值
    static final int TREEIFY_THRESHOLD = 8;
	//当红黑树节点数小于6转为链表的阈值
    static final int UNTREEIFY_THRESHOLD = 6;
	//最小树化容量阈值，即 当哈希表中的容量 > 该值时，才允许树形化链表 （即 将链表 转换成红黑树）；否则，若桶内元素太多时，则直接扩容，而不是树形化。 
	//为了避免进行扩容、树形化选择的冲突，这个值不能小于 4 * TREEIFY_THRESHOLD
    static final int MIN_TREEIFY_CAPACITY = 64;
    ...
    //桶
    transient Node<K,V>[] table;
	//用来遍历的 keySet values
    transient Set<Map.Entry<K,V>> entrySet;
	//存储元素个数
    transient int size;
	//记录修改次数的
    transient int modCount;
	//实际哈希表内元素数量的扩容阈值，当哈希表内元素数量超过阈值时，会发生扩容resize()
    int threshold;
	//实际加载因子，用于计算哈希表元素数量的阈值。  threshold = 哈希桶.length * loadFactor
    final float loadFactor;
    ...
}
```

具体属性的解释都备注好了，也没啥难懂的

然后我们看看具体存储的节点Node

```java
static class Node<K,V> implements Map.Entry<K,V> {
       final int hash;
       final K key;
       V value;
       Node<K,V> next;

       Node(int hash, K key, V value, Node<K,V> next) {
           this.hash = hash;
           this.key = key;
           this.value = value;
           this.next = next;
       }

       public final K getKey()        { return key; }
       public final V getValue()      { return value; }
       public final String toString() { return key + "=" + value; }

       public final int hashCode() {
           return Objects.hashCode(key) ^ Objects.hashCode(value);
       }

       public final V setValue(V newValue) {
           V oldValue = value;
           value = newValue;
           return oldValue;
       }

       public final boolean equals(Object o) {
           if (o == this)
               return true;
           if (o instanceof Map.Entry) {
               Map.Entry<?,?> e = (Map.Entry<?,?>)o;
               if (Objects.equals(key, e.getKey()) &&
                   Objects.equals(value, e.getValue()))
                   return true;
           }
           return false;
       }
   }
```

Node中记录了key、value、hash值以及下一个节点

然后主要注意一些东西，key和hash值都是final的，也就是初始化后不能轻易改变的，而value却不是，这就是为啥通常只能根据key来改变value，而没有根据value改变key；Node的hashCode是通过key和value相与得到的；equals不仅比较Node，也可以是key和value相同

接着看看构造函数

```java
public HashMap(int initialCapacity, float loadFactor) {
       if (initialCapacity < 0)
           throw new IllegalArgumentException("Illegal initial capacity: " +
                                              initialCapacity);
       if (initialCapacity > MAXIMUM_CAPACITY)
           initialCapacity = MAXIMUM_CAPACITY;
       if (loadFactor <= 0 || Float.isNaN(loadFactor))
           throw new IllegalArgumentException("Illegal load factor: " +
                                              loadFactor);
       this.loadFactor = loadFactor;
       this.threshold = tableSizeFor(initialCapacity);
   }

   public HashMap(int initialCapacity) {
       this(initialCapacity, DEFAULT_LOAD_FACTOR);
   }

   public HashMap() {
       this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
   }

   public HashMap(Map<? extends K, ? extends V> m) {
       this.loadFactor = DEFAULT_LOAD_FACTOR;
       putMapEntries(m, false);
   }
```

构造函数就只有4个，通常我们通过`new HashMap<>()`，这个时候就是无参构造，只会默认指定一个默认的加载因子，其他的都是再put时使用默认值

指定桶的初始容量，也会使用默认的加载因子，最后重载到根据容量、加载因子两个参数的构造器；在这个构造器中，会先后检验初始化的容量是否不合理（<0 或者超出最大值），同样会检验加载因子。然后将加载因子赋值给变量loadFactor，通过`tableSizeFor(initialCapacity)`方法来计算threshold阈值（这还不是确定的，只是尽进行了一个初始化）

先看看是怎么计算的吧

```java
//计算大于cap值的最小2的幂
static final int tableSizeFor(int cap) {
       int n = cap - 1;
       n |= n >>> 1;
       n |= n >>> 2;
       n |= n >>> 4;
       n |= n >>> 8;
       n |= n >>> 16;
       return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
   }
```

将传入的容量大小转化为：大于传入容量大小的最小的2的幂

好了，这里转换就是通过与运算和位运算来的，回到之前的构造方法，我们还有一个构造方法没看

还有一个构造方法是根据一个Map来初始化的，同样使用了默认的加载因子，然后通过`putMapEntries(m, false)`方法将Map中的元素添加到HashMap中，看看这个方法是什么吧

```java
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
       int s = m.size();
       if (s > 0) {
           if (table == null) { // pre-size
               float ft = ((float)s / loadFactor) + 1.0F;
               int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                        (int)ft : MAXIMUM_CAPACITY);
               if (t > threshold)
                   //这里计算出来就是有用的了
                   threshold = tableSizeFor(t);
           }
           else if (s > threshold)
               resize();
           for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
               K key = e.getKey();
               V value = e.getValue();
               //添加到哈希表中
               putVal(hash(key), key, value, false, evict);
           }
       }
   }
```

putMapEntries方法也很简单，如果table还没有初始化的话（为null），就先计算阈值；接着如果map的size大于了阈值，那么就要进行扩容了；如果都没问题的话，通过遍历Map，将key、value存入HashMap中

好了，构造方法和初始化就先看到这里吧，具体怎么扩容、怎么添加后面会讲到

到这里，我们发现，实例化的时候，我们都没有真正上的进行初始化桶（table数组），都只是进行一些值得确定（如加载因子、容量等）

## put

通常，我们都是通过put添加到HashMap中

```java
public V put(K key, V value) {
       return putVal(hash(key), key, value, false, true);
   }
```

put方法中，根据key值计算了hash值（就是我们存储的位置hash值），然后通过putVal方法添加到HashMap中

先看看这个计算key得hash吧

```java
static final int hash(Object key) {
       int h;
       return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
   }
```

从这个计算来看，可以知道一下几点

> 1. 当key为null时，hash值为0，说明key是可以为null的（对比HashTable，HashTable对key直接hashCode（），若key为null时，会抛出异常，所以HashTable的key不可为null）
> 2. 当key不为null时，先计算出key的哈希码h，然后进行扰动处理（按位异或哈希码自身右移16位），所以是两次扰动（一次异或，一次右移16位）

### putVal

通过key得到hash后，传入putVal方法

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                  boolean evict) {
       Node<K,V>[] tab; Node<K,V> p; int n, i;
       //table为null或者length为0，通过resize()扩容进行初始化，所以第一次初始化是在第一次put时候（以Map实例化除外，但它也会在resize中进行初始化）
       if ((tab = table) == null || (n = tab.length) == 0)
           //记录table的length
           n = (tab = resize()).length;
       //通过(length - 1) & hash计算下标，没有哈希冲突则直接添加进去
       if ((p = tab[i = (n - 1) & hash]) == null)
           tab[i] = newNode(hash, key, value, null);
       else {
           //产生哈希冲突
           Node<K,V> e; K k;
           //key值存在（处于桶内的头节点），更新value
           if (p.hash == hash &&
               ((k = p.key) == key || (key != null && key.equals(k))))
               e = p;
           else if (p instanceof TreeNode)
               //如果是红黑树节点，则以红黑树的方式添加
               e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
           else {
               //否则是链表，遍历链表到链尾，插入
               for (int binCount = 0; ; ++binCount) {
                   if ((e = p.next) == null) {
                       p.next = newNode(hash, key, value, null);
                       //如果到达了树化（链表转红黑树）的阈值（8），则进行树化
                       if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                           treeifyBin(tab, hash);
                       break;
                   }
                   if (e.hash == hash &&
                       ((k = e.key) == key || (key != null && key.equals(k))))
                       break;
                   p = e;
               }
           }
           //红黑树或链表中的存在key，更新value就可以了
           if (e != null) { // existing mapping for key
               V oldValue = e.value;
               if (!onlyIfAbsent || oldValue == null)
                   e.value = value;
               afterNodeAccess(e);
               return oldValue;
           }
       }
       //操作修改+1
       ++modCount;
       //是否需要扩容
       if (++size > threshold)
           resize();
       //LinkedHashMap用的api
       afterNodeInsertion(evict);
       return null;
   }
```

总的分为如下几个部分

1. table是否为null或者length为0：是通过`resize()`进行初始化
2. 通过桶的length，(length - 1) & hash计算在桶中的下标，为null则没有产生冲突，插入，否则进入下一步
3. 存在哈希冲突，首先判断是不是和桶内的头结点拥有同一个key，是则更新value，不是则进行桶内链表或者红黑树的比较
4. 如果是红黑树，则通过红黑树的插入方式进行插入
5. 不是红黑树，是链表，则遍历链表，如果链表中有相同的key，跳出遍历循环，更新value；如果没有，遍历到链尾，进行插入，插入成功后判断是否到达树化的阈值（8），如果到达，将链表进行树化（转为红黑树）
6. 记录操作数，size+1，判断是否到达阈值，是否需要进行扩容

### 扩容

我们看看`resize()`方法，如何进行扩容和初始化

```java
final Node<K,V>[] resize() {
       Node<K,V>[] oldTab = table;
       int oldCap = (oldTab == null) ? 0 : oldTab.length;
       int oldThr = threshold;
       int newCap, newThr = 0;
       if (oldCap > 0) {
           //检查是否到达最大值，已经到达最大值，不再进行扩容
           if (oldCap >= MAXIMUM_CAPACITY) {
               threshold = Integer.MAX_VALUE;
               return oldTab;
           }
           else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
               //新桶的容量newCap是旧桶容量的2倍，并且保证小于最大容量，旧容量大于默认容量16
               //新的加载因子是旧的2倍 一般是0.75变为1.5
               newThr = oldThr << 1; // double threshold
       }
       else if (oldThr > 0) // initial capacity was placed in threshold
           newCap = oldThr;
       else {               // zero initial threshold signifies using defaults
           //初始化为默认的容量 一般table==null且阈值为0或者不存在的情况下
           newCap = DEFAULT_INITIAL_CAPACITY;
           //计算新的阈值
           newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
       }
       if (newThr == 0) {
           float ft = (float)newCap * loadFactor;
           newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                     (int)ft : Integer.MAX_VALUE);
       }
       threshold = newThr;
       //真正初始化桶的位置
       @SuppressWarnings({"rawtypes","unchecked"})
           Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
       table = newTab;
       //扩容完成后，进行值的复制
       if (oldTab != null) {
           for (int j = 0; j < oldCap; ++j) {
               Node<K,V> e;
               if ((e = oldTab[j]) != null) {
                   oldTab[j] = null;
                   //只有头结点的情况
                   if (e.next == null)
                       newTab[e.hash & (newCap - 1)] = e;
                   else if (e instanceof TreeNode)
                       //是红黑树，则需要进行拆分，重新计算下标
                       ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                   else { // preserve order
                       //链表基本上不会有什么位置的变化，因为2的幂的缘故，计算出新的下标还是以前的下标，桶的容量始终保持2的幂也是为了方便链表的复制更高效一点
                       //低位头结点
                       Node<K,V> loHead = null, loTail = null;
                       //高位头结点
                       Node<K,V> hiHead = null, hiTail = null;
                       Node<K,V> next;
                       //遍历链表，尾插法
                       do {
                           next = e.next;
                           if ((e.hash & oldCap) == 0) {
                               if (loTail == null)
                                   loHead = e;
                               else
                                   loTail.next = e;
                               loTail = e;
                           }
                           else {
                               if (hiTail == null)
                                   hiHead = e;
                               else
                                   hiTail.next = e;
                               hiTail = e;
                           }
                       } while ((e = next) != null);
                       //将低位的链表放置到原index处
                       if (loTail != null) {
                           loTail.next = null;
                           newTab[j] = loHead;
                       }
                       //将高位链表放在新的index处
                       if (hiTail != null) {
                           hiTail.next = null;
                           newTab[j + oldCap] = hiHead;
                       }
                   }
               }
           }
       }
       return newTab;
   }
```

首先看一下是真正确定容量和阈值的

如果oldCap的容量是大于0的，则会检验旧容量是否达到了最大容量（达到最大容量不再进行扩容），没有到达最大容量，则进行2倍扩容（扩容后也要保证小于等于最大容量），同时阈值也扩大2倍；

如果当前table是空的，但是已经有了阈值，那么新容量就等于旧容量；

如果table是空的，且没有设置阈值，那么旧才用默认的值，默认容量16，默认阈值12（默认加载因子 * 默认容量）

接着更新阈值，然后构建新的哈希桶（如果之前没有初始化桶，那么这次就会初始化了）

接着又是判断旧桶存不存在，旧桶不为null，那么就需要将旧桶的数据复制到新桶中：

```java
1. 遍历旧桶
 2. 如果旧桶中有元素，则复制给e，将原哈希桶置空（以便GC）
 3. 如果旧桶中的链表就一个头结点（只有e），则没有哈希冲突，直接通过hash & (newCap - 1)（与运算）计算新的位置，然后存入
 4. 如果旧桶里放的是红黑树，则通过红黑树的方式，进行拆分，重新计算（一般红黑树都会重新转为链表）
 5. 如果旧桶里的链表不只一个结点，则通过遍历链表，从新计算节点的index下标，低位的仍然还在低位（以前的index下标），高位则放在高位（通过高低位头结点来记录）
```

好了，hash扰动和扩容基本就看得差不多了，接着看看其他几个添加api吧

```java
 public void putAll(Map<? extends K, ? extends V> m) {
        //这个方法前面我们也看过了
       putMapEntries(m, true);
   }
//若key对应的value之前存在，不会覆盖 1.8新增
public V putIfAbsent(K key, V value) {
       return putVal(hash(key), key, value, true, true);
   }
```

总之，不管怎样，put最后都会通过`putVal()`方法来添加

## get

接着看看从HashMap中获取元素吧

```java
public V get(Object key) {
       Node<K,V> e;
       return (e = getNode(hash(key), key)) == null ? null : e.value;
   }
```

首先根据key计算hash值，通过getNode来获取对应key的节点，为null则返回null，否则返回节点的value

```java
final Node<K,V> getNode(int hash, Object key) {
       Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
       if ((tab = table) != null && (n = tab.length) > 0 &&
           (first = tab[(n - 1) & hash]) != null) {
           //检查头结点
           if (first.hash == hash && // always check first node
               ((k = first.key) == key || (key != null && key.equals(k))))
               return first;
           if ((e = first.next) != null) {
               //红黑树的查找
               if (first instanceof TreeNode)
                   return ((TreeNode<K,V>)first).getTreeNode(hash, key);
               //链表内的查找
               do {
                   if (e.hash == hash &&
                       ((k = e.key) == key || (key != null && key.equals(k))))
                       return e;
               } while ((e = e.next) != null);
           }
       }
       return null;
   }
```

首先是对table桶的判断和元素的判断，有才会进行查找

如果刚好是头节点，返回

如果不是头节点，判断是不是红黑树节点，是通过红黑树查找；否则是链表，遍历链表查找

几个类似方法

```java
//有默认值的get方法 1.8新增
public V getOrDefault(Object key, V defaultValue) {
       Node<K,V> e;
       return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
   }
```

## replace

```java
public boolean replace(K key, V oldValue, V newValue) {
       Node<K,V> e; V v;
       if ((e = getNode(hash(key), key)) != null &&
           ((v = e.value) == oldValue || (v != null && v.equals(oldValue)))) {
           e.value = newValue;
           afterNodeAccess(e);
           return true;
       }
       return false;
   }

public V replace(K key, V value) {
       Node<K,V> e;
       if ((e = getNode(hash(key), key)) != null) {
           V oldValue = e.value;
           e.value = value;
           afterNodeAccess(e);
           return oldValue;
       }
       return null;
   }
```

replace方法都是通过getNode来获取到对应的节点，然后替换旧值

## remove

通过remove删除一个元素

```java
public V remove(Object key) {
      Node<K,V> e;
      return (e = removeNode(hash(key), key, null, false, true)) == null ?
          null : e.value;
  }
```

跟get方法类似，根据key获取hash值，然后通过removeNode方法进行具体的删除

```java
final Node<K,V> removeNode(int hash, Object key, Object value,
                              boolean matchValue, boolean movable) {
       Node<K,V>[] tab; Node<K,V> p; int n, index;
       if ((tab = table) != null && (n = tab.length) > 0 &&
           (p = tab[index = (n - 1) & hash]) != null) {
           Node<K,V> node = null, e; K k; V v;
           //头节点的检查
           if (p.hash == hash &&
               ((k = p.key) == key || (key != null && key.equals(k))))
               //待删除节点赋值给node
               node = p;
           else if ((e = p.next) != null) {
               //红黑树的获取节点
               if (p instanceof TreeNode)
                   node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
               else {
                   //链表获取key的节点
                   do {
                       if (e.hash == hash &&
                           ((k = e.key) == key ||
                            (key != null && key.equals(k)))) {
                           node = e;
                           break;
                       }
                       p = e;
                   } while ((e = e.next) != null);
               }
           }
           if (node != null && (!matchValue || (v = node.value) == value ||
                                (value != null && value.equals(v)))) {
               //删除红黑树的key的节点
               if (node instanceof TreeNode)
                   ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
               else if (node == p)
                   //如果是头结点直接通过next覆盖
                   tab[index] = node.next;
               else
                   //链表中间，通过链表直接覆盖
                   p.next = node.next;
               //记录操作次数
               ++modCount;
               //size-1
               --size;
               //LinkedHashMap使用的api
               afterNodeRemoval(node);
               return node;
           }
       }
       return null;
   }
```

整个删除逻辑也不复杂，根据key计算hash值，得到对应的下标，然后依次判断是头结点、红黑树中的节点、链表中的节点，然后进行删除就可以了

```java
public boolean remove(Object key, Object value) {
       return removeNode(hash(key), key, value, true, true) != null;
   }
```

key、value为条件删除，最终也是嗲用removeNode方法

## 其他api

### 清空

```java
//清空操作，置为null
public void clear() {
       Node<K,V>[] tab;
       modCount++;
       if ((tab = table) != null && size > 0) {
           size = 0;
           for (int i = 0; i < tab.length; ++i)
               tab[i] = null;
       }
   }
```

### 遍历

```java
public Set<Map.Entry<K,V>> entrySet() {
       Set<Map.Entry<K,V>> es;
       //entrySet是缓存
       return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
   }
```

看看EntrySet

```java
final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
    public final int size()                 { return size; }
       public final void clear()               { HashMap.this.clear(); }
       public final Iterator<Map.Entry<K,V>> iterator() {
           return new EntryIterator();
       }
       //通过getNode方法
       public final boolean contains(Object o) {
           if (!(o instanceof Map.Entry))
               return false;
           Map.Entry<?,?> e = (Map.Entry<?,?>) o;
           Object key = e.getKey();
           Node<K,V> candidate = getNode(hash(key), key);
           return candidate != null && candidate.equals(e);
       }
       //通过removeNode方法
       public final boolean remove(Object o) {
           if (o instanceof Map.Entry) {
               Map.Entry<?,?> e = (Map.Entry<?,?>) o;
               Object key = e.getKey();
               Object value = e.getValue();
               return removeNode(hash(key), key, value, true, true) != null;
           }
           return false;
       }
       //获取迭代器
       public final Spliterator<Map.Entry<K,V>> spliterator() {
           return new EntrySpliterator<>(HashMap.this, 0, -1, 0, 0);
       }
       //forEach遍历
       public final void forEach(Consumer<? super Map.Entry<K,V>> action) {
           Node<K,V>[] tab;
           if (action == null)
               throw new NullPointerException();
           if (size > 0 && (tab = table) != null) {
               int mc = modCount;
               for (int i = 0; i < tab.length; ++i) {
                   for (Node<K,V> e = tab[i]; e != null; e = e.next)
                       action.accept(e);
               }
               if (modCount != mc)
                   throw new ConcurrentModificationException();
           }
       }
   }
```

## 总结

> 1. HashMap是一个键值对的Map集合，是无序的且遍历无序
> 2. 是线程非安全的，key和value都可以为null
> 3. 开放地址法，俗称拉链法来解决哈希冲突；其底层采用数组作为哈希桶table，桶内存放链表（1.8 链表大于8转为红黑树，节点数小于6转为链表）
> 4. 默认容量是16（保证2的幂），默认加载因子0.75，根据`默认容量 * 默认加载因子`计算阈值，当容量达到阈值会进行扩容操作；table桶的初始化是在第一次put（通过putVal方法）存入的时候才进行（构造器中的初始化只是一些值的初始化如容量、加载因子、阈值等）
> 5. 扩容当table桶没有初始化或者达到容量时，会进行扩容；扩容前，会再次确定容量、阈值，然后创建一个新的table桶，如果存在旧桶，则会将旧桶的值复制到新桶中；复制过程，会根据新桶的容量计算新的下标，然后会根据头结点、红黑树节点（红黑树会将以前的红黑树进行拆分，重新计算哈希冲突，最后可能是红黑树，也可能是链表）、链表来进行不同的复制（链表复制过程中，会有低位链表——下标是没有变的，高位链表——新的下标；这是由于桶的容量始终是2的幂产生的一种高效率的方式）
> 6. 插入操作前，会检查是否扩容；然后计算根据`(length - 1) & hash(key)`（通过位与运算来代替求余运算）来计算桶中的下标，然后依次检查头节点、红黑树节点、链表；头结点如果没有值或者有key，则插入或更新；红黑树节点，则通过红黑树添加；链表则是通过遍历链表，有相同的key则替换value，没有则在链尾插入新的节点，如果节点数大于8则进行树化；插入完成会去检验是否需要扩容
> 7. get、remove、replace等方法，都十分简单，通过key获取hash值进而得到下标，然后还是头节点、红黑树、链表几种不同的获取或删除节点

### 1.7和1.8的区别

1. 1.7的一些运算赋值在1.8中变为了位运算，一些判断也从Math变为了if else
2. JDK 1.7 做了9次扰动处理 = 4次位运算 + 5次异或运算。1.8 简化了扰动函数 = 只做了2次扰动 = 1次位运算 + 1次异或运算
3. 1.7扩容函数是在确定要扩容进行，1.8的扩容函数可能是初始化；1.7更新阈值在扩容后，1.8在扩容前，很多判断、阈值更新都是在扩容前
4. 1.8扩容后进行复制时，会先进行链表的一个判断（当前链表只有一个元素时），1.7没有；1.8有链表高低位的处理，是将整个旧链表复制到新的中，低位链表的索引跟原来的一样，高位的则放在新的索引处（尾插法），1.7则是遍历旧链表，头插法插入新链表（可能导致逆序）
5. 1.8中加入了红黑树，当链表长度到8时链表转化为红黑树，扩容时，重新计算存储位置，红黑树内数量<6又会转化为链表
6. key和value都允许为null（key只能有一个为null，而value则可以有多个为null）

## 特别鸣谢

[面试必备：HashMap源码解析（JDK8）](https://blog.csdn.net/zxt0601/article/details/77413921)

[Java源码分析：关于 HashMap 1.8 的重大更新](https://blog.csdn.net/carson_ho/article/details/79373134)