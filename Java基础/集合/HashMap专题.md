> 大部分来源是其他博客，小部分是自己的理解，站在巨人的肩上，会看得更远。因为java对HashMap做了较大的优化，故本笔记分为jdk1.7和1.8两个版本叙述。

## 一、HashMap源码分析(JDK1.7)
### HashMap结构

HashMap的主干是一个Entry数组。每一个Entry包含一个key-value键值对。
```java
//HashMap的主干数组，可以看到就是一个Entry数组，初始值为空数组{}，主干数组的长度一定是2的次幂，至于为什么这么做，后面会有详细分析。
transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;
```
Entry是HashMap中的一个静态内部类。代码如下：
```java
static class Entry<K,V> implements Map.Entry<K,V> {
    final K key;
    V value;
    Entry<K,V> next;//存储指向下一个Entry的引用，单链表结构
    int hash;//对key的hashcode值进行hash运算后得到的值，存储在Entry，避免重复计算

    Entry(int h, K k, V v, Entry<K,V> n) {
        value = v;
        next = n;
        key = k;
        hash = h;
    } 
```
其他几个重要字段：
```java
//实际存储的key-value键值对的个数
transient int size;
//阈值，当table == {}时，该值为初始容量（初始容量默认为16）；当table被填充了，也就是为table分配内存空间后，threshold一般为 capacity*loadFactory。HashMap在进行扩容时需要参考threshold，后面会详细谈到
int threshold;
//负载因子，代表了table的填充度有多少，默认是0.75
final float loadFactor;
//用于快速失败，由于HashMap非线程安全，在对HashMap进行迭代时，如果期间其他线程的参与导致HashMap的结构发生变化了（比如put，remove等操作），需要抛出异常ConcurrentModificationException
transient int modCount;
```
HashMap有4个构造器，其他构造器如果用户没有传入`initialCapacity `和`loadFactor`这两个参数，会使用默认值。`initialCapacity`默认为16，`loadFactory`默认为0.75。

我们看下其中一个构造方法
```java
public HashMap(int initialCapacity, float loadFactor) {
　//此处对传入的初始容量进行校验，最大不能超过MAXIMUM_CAPACITY = 1<<30(2^30)
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);

    this.loadFactor = loadFactor;
    threshold = initialCapacity;
　　　　　
    init();//init方法在HashMap中没有实际实现，不过在其子类如 linkedHashMap中就会有对应实现
}
```
从上面这段代码我们可以看出，在常规构造器中，没有为数组table分配内存空间（有一个入参为指定Map的构造器例外），而是在执行put操作的时候才真正构建table数组。

### put流程

```java
public V put(K key, V value) {
    //如果table数组为空数组{}，进行数组填充（为table分配实际内存空间），入参为threshold，此时threshold为initialCapacity 默认是1<<4(2^4=16)
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);
    }
   //如果key为null，存储位置为table[0]或table[0]的冲突链上
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key);//对key的hashcode进一步计算，确保散列均匀
    int i = indexFor(hash, table.length);//获取在table中的实际位置
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
    //如果该对应数据已存在，执行覆盖操作。用新value替换旧value，并返回旧value
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;//保证并发访问时，若HashMap内部结构发生变化，快速响应失败
    addEntry(hash, key, value, i);//新增一个entry
    return null;
}
```
put流程还是很清晰的，总结如下：

**step 1: 在第一个元素插入 HashMap 的时候调用`inflateTable`方法做一次数组的初始化**

```java
private void inflateTable(int toSize) {
    int capacity = roundUpToPowerOf2(toSize);//capacity一定是2的次幂
    threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);//此处为threshold赋值，取capacity*loadFactor和MAXIMUM_CAPACITY+1的最小值，capaticy一定不会超过MAXIMUM_CAPACITY，除非loadFactor大于1
    table = new Entry[capacity];
    initHashSeedAsNeeded(capacity);//ignore
}
```

**step 2: 允许key为null,固定存储在table[0]或者table[0]的冲突链上**

**step 3: 计算hash值，然后计算数组的下标i**

如果把我们覆盖的Object::hashcode方法的返回直接作用在indexFor方法上，可以发现的是：在做`&`运算的时候，仅仅是**后几位有效。**那如果我们key的哈希值高位变化很大，低位变化很小。直接拿过去做`&`运算，这就会导致计算出来的Hash值相同的很多。

而设计者**将key的哈希值的高位也做了运算(与高16位做异或运算，使得在做&运算时，此时的低位实际上是高位与低位的结合)，这就增加了随机性**，减少了碰撞冲突的可能性，同时不会有太大的开销 以下是hash函数代码：

```java
//这是一个神奇的函数，用了很多的异或，移位等运算，对key的hashcode进一步进行计算以及二进制位的调整等来保证最终获取的存储位置尽量分布均匀
final int hash(Object k) {
        int h = hashSeed;
        if (0 != h && k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }
        h ^= k.hashCode();
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }

```

看不懂不存在的，我们看看JDK1.8的实现，比较简洁：

```java
static final int hash(Object key) {  
     int h;
     // h = key.hashCode() 为第一步 取hashCode值
     // h ^ (h >>> 16)  为第二步 高位参与运算
     return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

![](./imgs/索引位置.png)

以上hash函数计算出的值，通过indexFor进一步处理来获取实际的存储位置

```java
static int indexFor(int h, int length) {
    return h & (length-1);
}
```

h&（length-1）保证获取的index一定在数组范围内，举个例子，默认容量16，length-1=15，h=18,转换成二进制计算为

```
    1  0  0  1  0
&   0  1  1  1  1
    __________________
    0  0  0  1  0    = 2
```

最终计算出的index=2。有些版本的对于此处的计算会使用取模运算，也能保证index一定在数组范围内，不过位运算对计算机来说，性能更高一些（HashMap中有大量位运算）。

所以最终存储位置的确定流程是这样的：
![image](https://raw.githubusercontent.com/herofishs/markdownPhoto/master/%E6%90%9C%E7%8B%97%E6%88%AA%E5%9B%BE20180829174702.png)

**step 4: 遍历table[i]和他的冲突链，如果找到key值相同的节点就覆盖这个节点下面的value，返回旧值，结束put流程。如果没有找到,进入step 5。**

**step 5: 调用addEntry()方法**

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    // 如果当前 HashMap 大小已经达到了阈值，并且新值要插入的数组位置已经有元素了，那么要扩容
    if ((size >= threshold) && (null != table[bucketIndex])) {
        // 扩容，后面会介绍一下
        resize(2 * table.length);
        // 扩容以后，重新计算 hash 值
        hash = (null != key) ? hash(key) : 0;
        // 重新计算扩容后的新的下标
        bucketIndex = indexFor(hash, table.length);
    }
    // 往下看
    createEntry(hash, key, value, bucketIndex);
}
// 这个很简单，其实就是将新值放到链表的表头，然后 size++
void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```

先判断是否需要执行扩容操作，注意扩容条件是**当发生哈希冲突并且size大于阈值。**然后再将这个新的数据插入到数组的相应位置处的链表的**表头**。

我们来继续看上面提到的resize方法
```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {//扩容前的数组大小如果已经达到最大(2^30)了
        threshold = Integer.MAX_VALUE;//修改阈值为int的最大值(2^31-1)，这样以后就不会扩容了
        return;
    }
	// 新的数组
    Entry[] newTable = new Entry[newCapacity];
    // 将原来数组中的值迁移到新的更大的数组中
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}
```
如果数组进行扩容，数组长度发生变化，而存储位置`index = h&(length-1)`也可能会发生变化，需要重新计算index，我们先来看看transfer这个方法:
```java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
　//for循环中的代码，逐个遍历链表，重新计算索引位置，将老数组数据复制到新数组中去（数组不存储实际数据，所以仅仅是拷贝引用而已）
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
　　　　　 //将当前entry的next链指向新的索引位置,newTable[i]有可能为空，有可能也是个entry链，如果是entry链，直接在链表头部插入。
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}
```
newTable[i]的引用赋给了e.next，也就是使用了单链表的头插入方式，同一位置上新元素总会被放在链表的头部位置；**这样先放在一个索引上的元素终会被放到Entry链的尾部(如果发生了hash冲突的话），这一点和Jdk1.8有区别。**另外，**在旧数组中同一条Entry链上的元素，通过重新计算索引位置后，有可能被放到了新数组的不同位置上。**

### get流程

```java
public V get(Object key) {
　　　　 //如果key为null,则直接去table[0]处去检索即可。
        if (key == null)
            return getForNullKey();
        Entry<K,V> entry = getEntry(key);
        return null == entry ? null : entry.getValue();
 }
```
get方法通过key值返回对应value，如果key为null，直接去table[0]处检索。我们再看一下getEntry这个方法
```java
final Entry<K,V> getEntry(Object key) {
       
    if (size == 0) {
        return null;
    }
    //通过key的hashcode值计算hash值
    int hash = (key == null) ? 0 : hash(key);
    //indexFor (hash&length-1) 获取最终数组索引，然后遍历链表，通过equals方法比对找出对应记录
    for (Entry<K,V> e = table[indexFor(hash, table.length)];e != null;e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
            return e;
    }
    return null;
} 
```
可以看出，get方法的实现相对简单，key(hashcode)-->hash-->indexFor-->最终索引位置，找到对应位置table[i]，再查看是否有链表，遍历链表，通过key的equals方法比对查找对应的记录。

要注意的是，有人觉得上面在定位到数组位置之后然后遍历链表的时候，e.hash == hash这个判断没必要，仅通过equals判断就可以。其实不然，试想一下，如果key类仅仅重写了equals方法却没有重写hashCode，导致两个key对象equals相等，而hashcode不相等。如果恰巧这两个对象映射到同一个位置，那么仅仅用equals判断可能是相等的，但其hashCode和当前对象不一致，这种情况，根据Object的hashCode的约定，不能返回当前对象，而应该返回null。

## 二、HashMap JDK1.8

虽然JDK1.8对于HashMap做了较大的性能优化，但是核心流程还是没有改变的。并且JDK1.8在源码上精简了不少，但是可阅读性也下降了，这里给出JDK1.8的put方法流程图，方便理解。有能力的同学可以查阅JDK1.8的源码。

![](./imgs/HashMap-put.png)

①.判断键值对数组table[i]是否为空或为null，否则执行resize()进行扩容；

②.根据键值key计算hash值得到插入的数组索引i，如果table[i]==null，直接新建节点添加，转向⑥，如果table[i]不为空，转向③；

③.判断table[i]的首个元素是否和key一样，如果相同直接覆盖value，否则转向④，这里的相同指的是hashCode以及equals；

④.判断table[i] 是否为treeNode，即table[i] 是否是红黑树，如果是红黑树，则直接在树中插入键值对，否则转向⑤；

⑤.遍历table[i]，判断链表长度是否大于8，大于8的话把链表转换为红黑树，在红黑树中执行插入操作，否则进行链表的插入操作；遍历过程中若发现key已经存在直接覆盖value即可；

⑥.插入成功后，判断实际存在的键值对数量size是否超多了最大容量threshold，如果超过，进行扩容。

## 三、HashMap JDK1.8与1.7的区别
所以这里给出两者的区别，读者可以自行体会，有能力的同学可以看看源码。

### **结构不同**

JDK1.8中使用一个Node数组来存储数据，但这个Node可能是链表结构，也可能是红黑树结构。

如果链表的节点超过了8个，那么会调用treeifyBin函数，将链表转换为红黑树。

那么即使hashcode完全相同，由于红黑树的特点，查找某个特定元素，也只需要O(log n)的开销，也就是说put/get的操作的时间复杂度最差只有O(log n)

听起来挺不错，但是真正想要利用JDK1.8的好处，有一个限制：**key的对象，必须正确的实现了Compare接口**。如果没有实现Compare接口，或者实现得不正确（比方说所有Compare方法都返回0）
那JDK1.8的HashMap其实还是慢于JDK1.7的。

**为什么要这么做？**

应该是为了避免Hash Collision DoS攻击，Java中String的hashcode函数的强度很弱，有心人可以很容易的构造出大量hashcode相同的String对象。如果向服务器一次提交数万个hashcode相同的字符串参数，那么可以很容易的卡死JDK1.7版本的服务器。但是String正确的实现了Compare接口，因此在JDK1.8版本的服务器上，Hash Collision DoS不会造成不可承受的开销。

注意，**并不是桶子上有8位元素的时候它就能变成红黑树，它得同时满足我们的散列表容量大于64才行的**。 

### **插入方式不同**

JDK1.7用的是头插法，而JDK1.8及之后使用的都是尾插法

**为什么要这样做？**

因为JDK1.7是用单链表进行的纵向延伸，当采用头插法时会容易出现逆序且环形链表死循环问题。但是在JDK1.8之后是因为加入了红黑树使用尾插法，能够避免出现逆序且链表死循环的问题。

### 扩容操作的时机不同

JDK1.7在put之前扩容，JDK1.8在put之后扩容。同时JDK1.7的扩容条件是**当发生哈希冲突并且size大于阈值**，而JDK1.8的扩容条件仅仅是size大于阈值

### **扩容后迁移数据时老数据在新数组的存储位置计算方式不同**

JDK1.7是重新计算的hash值，然后再计算存储位置，而JDK1.8是利用了扩容前已经计算了的hash值，做了一个简单的操作，并不需要重新计算。

由于是双倍扩容，迁移过程中，会将原来 table[i] 中的链表的所有节点，分拆到新的数组的 newTable[i] 和 newTable[i + oldLength] 位置上。

![](./imgs/2.png)

元素在重新计算hash之后，因为n变为2倍，那么n-1的二进制在高位多1bit(红色)，因此新的index就会发生这样的变化： 

![](./imgs/3.png)

因此，我们在扩充HashMap的时候，不需要像JDK1.7的实现那样重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”，可以看看下图为16扩充为32的resize示意图:

![](./imgs/4.png)

这个设计确实非常的巧妙，既省去了重新计算hash值的时间，而且同时，由于新增的1bit是0还是1可以认为是随机的，因此resize的过程，均匀的把之前的冲突的节点分散到新的bucket了。这一块就是JDK1.8新增的优化点。

**除此之外，JDK1.7中rehash的时候，旧链表迁移新链表的时候，如果在新表的数组索引位置相同，则链表元素会倒置，但是从上图可以看出，JDK1.8不会倒置。**

## 四、并发下的HashMap
* JDK1.7

HashMap在多线程的情况下，会引发安全问题。

解释的过程十分烧脑，并且面试让你说过程根本说不明白，就不放在这里了，百度一大堆，[我这里也给个相对比较好理解的链接](http://www.sohu.com/a/204365538_478315/)。

总结如下： 
1. Hashmap的Resize包含扩容和ReHash两个步骤，其中ReHash步骤中的transfer方法在并发的情况下可能会形成链表环，在后面的get操作时e = e.next操作无限循环，Infinite Loop出现。

2. 多线程put的时候可能导致元素丢失。
  ```
  void transfer(Entry[] newTable) {  
        Entry[] src = table;  
        int newCapacity = newTable.length;  
        for (int j = 0; j < src.length; j++) {  
            Entry e = src[j];  
            if (e != null) {  
                src[j] = null;  //1
                do {  
                    Entry next = e.next;  
                    int i = indexFor(e.hash, newCapacity);  
                    e.next = newTable[i];  
                    newTable[i] = e;  
                    e = next;  
                } while (e != null);  
            }  
        }  
  ```
   问题出在transfer处，当需要reSize时，如果线程一执行到了上述代码1中的位置（即把旧表置为null）,然后线程二执行get操作，这个时候访问的是旧表，而此时拿到的却是null。[详情](http://blog.51cto.com/hisundev/1093919)

 大佬认为，这个不算是bug，是正常情况，在并发环境下，推荐使用ConcurrentHashMap。

 * JDK1.8   

本来以为是正常设置，然后在jdk1.8不会被修复，没想到只是大佬嘴硬不承认时bug罢了。事实上，在jdk1.8版本，hashmap多线程put不会造成死循环。

如何修复的呢？非常简单，应发死循环的原因是因为reSize原来链表部分的entry会倒置，JDK8是用 head 和 tail 来保证链表的顺序和之前一样，这样就不会产生循环引用。

但是，jdk1.8虽然修复了死循环的bug,还是有数据丢失等问题，故依然建议在并发环境下，使用ConcurrentHashMap。

## 五、HashMap的常见面试题

### 1.为什么 HashMap 中 String、Integer 这样的包装类适合作为 key 键

* String、Integer是final类，具有不可变性，保证了Key的不可更改，不会出现put&get时哈希值不同的情况。
* 内部已经重写了equals和hashcode方法，严格遵守约定&Hash碰撞少。

### 2.为什么HashMap的数组长度一定是2的次幂？

* 扩容迁移数组时，不用重新计算hash值。（JDK1.8优化）
* 可以使用&代替取余计算，提升效率

### 3.为什么HashMap线程不安全？

* JDK1.7、JDK1.8都没有使用同步

* JDK1.7在并发下容器出现reSize()死循环

* fail-fast策略

  每修改一次HashMap的内容，modCount变量增一，判断modCount的值与预期值是否相同，如不相同，则表示有其他线程修改HashMap，那么就快速响应失败。

### 4.HashMap的缺点

- 不支持范围查找
- 无序的，即遍历的时候，先put的节点，不一定先get。（这应该算特点，而不是缺点）
- 空间浪费，HashMaP中的数组一定是2的幂。
- 线程不安全

### 四、ConcurrentHashMap

烧脑指数爆表，日后补充。
#### jdk1.7

整个 ConcurrentHashMap 由一个个 Segment 组成，Segment 代表”部分“或”一段“的意思，所以很多地方都会将其描述为分段锁。注意，行文中，我很多地方用了“槽”来代表一个 segment。

简单理解就是，ConcurrentHashMap 是一个 Segment 数组，Segment 通过继承 ReentrantLock 来进行加锁，所以每次需要加锁的操作锁住的是一个 segment，这样只要保证每个 Segment 是线程安全的，也就实现了全局的线程安全。

![image](https://raw.githubusercontent.com/herofishs/markdownPhoto/master/concurrentHashMap%E7%BB%93%E6%9E%84.png)

concurrencyLevel：并行级别、并发数、Segment 数，怎么翻译不重要，理解它。默认是 16，也就是说 ConcurrentHashMap 有 16 个 Segments，所以理论上，这个时候，最多可以同时支持 16 个线程并发写，只要它们的操作分别分布在不同的 Segment 上。这个值可以在初始化的时候设置为其他值，但是一旦初始化以后，它是不可以扩容的。

再具体到每个 Segment 内部，其实每个 Segment 很像之前介绍的 HashMap，不过它要保证线程安全，所以处理起来要麻烦些。

```java
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    // 计算并行级别 ssize，因为要保持并行级别是 2 的 n 次方
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    // 我们这里先不要那么烧脑，用默认值，concurrencyLevel 为 16，sshift 为 4
    // 那么计算出 segmentShift 为 28，segmentMask 为 15，后面会用到这两个值
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;
 
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
 
    // initialCapacity 是设置整个 map 初始的大小，
    // 这里根据 initialCapacity 计算 Segment 数组中每个位置可以分到的大小
    // 如 initialCapacity 为 64，那么每个 Segment 或称之为"槽"可以分到 4 个
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    // 默认 MIN_SEGMENT_TABLE_CAPACITY 是 2，这个值也是有讲究的，因为这样的话，对于具体的槽上，
    // 插入一个元素不至于扩容，插入第二个的时候才会扩容
    int cap = MIN_SEGMENT_TABLE_CAPACITY; 
    while (cap < c)
        cap <<= 1;
 
    // 创建 Segment 数组，
    // 并创建数组的第一个元素 segment[0]
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    // 往数组写入 segment[0]
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
```
初始化完成，我们得到了一个 Segment 数组。

我们就当是用 new ConcurrentHashMap() 无参构造函数进行初始化的，那么初始化完成后：

* Segment 数组长度为 16，不可以扩容
* Segment[i] 的默认大小为 2，负载因子是 0.75，得出初始阈值为 1.5，也就是以后插入第一个元素不会触发扩容，插入第二个会进行第一次扩容
* 这里初始化了 segment[0]，其他位置还是 null，至于为什么要初始化 segment[0]，后面的代码会介绍
* 当前 segmentShift 的值为 32 – 4 = 28，segmentMask 为 16 – 1 = 15，姑且把它们简单翻译为移位数和掩码，这两个值马上就会用到
####  jdk1.8
* 





### 六、LinkedHashMap

**参考资料**

* [Java 8系列之重新认识HashMap](https://tech.meituan.com/2016/06/24/java-hashmap.html)
* [Java7/8 中的 HashMap 和 ConcurrentHashMap 全解析](http://www.importnew.com/28263.html)
* [Hashmap的结构，1.7和1.8有哪些区别](https://blog.csdn.net/qq_36520235/article/details/82417949)