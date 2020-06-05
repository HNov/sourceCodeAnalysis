HashMap(JDK8)知识汇总

[TOC]



HashMap入门比较视频

[https://www.bilibili.com/video/av24032788](https://yq.aliyun.com/go/articleRenderRedirect?spm=a2c4e.11153940.0.0.41525eebnlnxc9&url=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2Fav24032788)

hashmap原理

https://www.toutiao.com/i6815043369220178435/?tt_from=weixin&utm_campaign=client_share&wxshare_count=1&timestamp=1586961142&app=news_article&utm_source=weixin&utm_medium=toutiao_android&req_id=202004152232210100140470130A47DAE0&group_id=6815043369220178435

# 一、数据结构

## 特性

- HashMap存储键值对，实现快速存取数据，时间复杂度见 [常用数据结构的时间复杂度](https://www.cnblogs.com/aspirant/p/8902285.html)；

- 允许null键/值；
- 非线程安全；

- 不保证有序(比如插入的顺序)

- 实现map接口
- 继承AbstractMap

## 存储结构

这里需要区分一下，JDK1.7和 JDK1.8之后的 HashMap 存储结构。在JDK1.7及之前，是用数组加链表的方式存储的。

但是，众所周知，当链表的长度特别长的时候，查询效率将直线下降，查询的时间复杂度为 O(n)。因此，JDK1.8 把它设计为达到一个特定的阈值之后，就将链表转化为红黑树。（数组默认初始化大小为16,当链接长度大于8的时候就会转换为红黑树,实现logn的复杂度查询,如果小于6就会再次转换为链表）

这里简单说下红黑树的特点：

> 1. 每个节点只有两种颜色：红色或者黑色
> 2. 根节点必须是黑色
> 3. 每个叶子节点（NIL）都是黑色的空节点
> 4. 从根节点到叶子节点，不能出现两个连续的红色节点
> 5. 从任一节点出发，到它下边的子节点的路径包含的黑色节点数目都相同

由于红黑树，是一个自平衡的二叉搜索树，因此可以使查询的时间复杂度降为O(logn)。（红黑树不是本文重点，不了解的童鞋可自行查阅相关资料哈）

HashMap 结构示意图：

![HashMap数据结构](https://gitee.com/HNov/image/raw/master/typora/20200417002337.jpeg)

HashMap底层数据结构:**数组+链表+红黑树**,可以接受null的key和value值.数组默认初始化大小为16,当链接长度大于8的时候就会转换为红黑树,实现logn的复杂度查询,如果小于6就会再次转换为链表.
其中的结构单体是`entry`,包括四个元素:key,value,next,hash值.

 常量

在 HashMap源码中，比较重要的常用变量。

- DEFAULT_INITIAL_CAPACITY = 1 << 4

- MAXIMUM_CAPACITY = 1 << 30

- DEFAULT_LOAD_FACTOR = 0.75f

- TREEIFY_THRESHOLD = 8

- UNTREEIFY_THRESHOLD = 6

- MIN_TREEIFY_CAPACITY = 64

  

```java
//默认的初始化容量为16，必须是2的n次幂
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

//最大容量为 2^30
static final int MAXIMUM_CAPACITY = 1 << 30;

//默认的加载因子0.75，乘以数组容量得到的值，用来表示元素个数达到多少时，需要扩容。
//为什么设置 0.75 这个值呢，简单来说就是时间和空间的权衡。
//若小于0.75如0.5，则数组长度达到一半大小就需要扩容，空间使用率大大降低，
//若大于0.75如0.8，则会增大hash冲突的概率，影响查询效率。
static final float DEFAULT_LOAD_FACTOR = 0.75f;

//刚才提到了当链表长度过长时，会有一个阈值，超过这个阈值8就会转化为红黑树
static final int TREEIFY_THRESHOLD = 8;

//当红黑树上的元素个数，减少到6个时，就退化为链表
static final int UNTREEIFY_THRESHOLD = 6;

//链表转化为红黑树，除了有阈值的限制，还有另外一个限制，需要数组容量至少达到64，才会树化。
//这是为了避免，数组扩容和树化阈值之间的冲突。
static final int MIN_TREEIFY_CAPACITY = 64;
```

## 常用变量

```java
//存放所有Node节点的数组
transient Node<K,V>[] table;

//存放所有的键值对
transient Set<Map.Entry<K,V>> entrySet;

//map中的实际键值对个数，即数组中元素个数
transient int size;

//每次结构改变时，都会自增，fail-fast机制，这是一种错误检测机制。
//当迭代集合的时候，如果结构发生改变，则会发生 fail-fast，抛出异常。
transient int modCount;

//数组扩容阈值
int threshold;

//加载因子
final float loadFactor;
```

## 内部类 

![image-20200416002452275](https://gitee.com/HNov/image/raw/master/typora/20200417002351.png)

###  普通链表节点

```java
//普通单向链表节点类
static class Node<K,V> implements Map.Entry<K,V> {
    //key的hash值，put和get的时候都需要用到它来确定元素在数组中的位置
    final int hash;
    final K key;
    V value;
    //指向单链表的下一个节点
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
}
```

### 红黑树节点

```java
//转化为红黑树的节点类
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    //当前节点的父节点
    TreeNode<K,V> parent;  
    //左孩子节点
    TreeNode<K,V> left;
    //右孩子节点
    TreeNode<K,V> right;
    //指向前一个节点
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    //当前节点是红色或者黑色的标识
    boolean red;
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }
}
```



- HashMap中size表示当前共有多少个KV对;

- capacity表示当前HashMap的容量是多少，默认值是16，每次扩容都是成倍的;

- loadFactor是装载因子，当Map中元素个数超过`loadFactor* capacity`的值时，会触发扩容。`loadFactor* capacity`可以用threshold表示。

# 二、方法





![image-20200416001352499](https://gitee.com/HNov/image/raw/master/typora/20200417003937.png)

## 构造函数

HashMap有四个构造函数可供我们使用，一起来看下：

- HashMap()
- HashMap(int initialCapacity)
- HashMap(int initialCapacity, float loadFactor)
- HashMap(Map<? extends K, ? extends V> m)

```java
//1.默认无参构造，指定一个默认的加载因子
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; 
}

//2.可指定容量的有参构造，但是需要注意当前我们指定的容量并不一定就是实际的容量，下面会说
public HashMap(int initialCapacity) {
    //同样使用默认加载因子
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

//3.可指定容量和加载因子，但是笔者不建议自己手动指定非0.75的加载因子
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
    //这里就是把我们指定的容量改为一个大于它的的最小的2次幂值，如传过来的容量是14，则返回16
    //注意这里，按理说返回的值应该赋值给 capacity，即保证数组容量总是2的n次幂，为什么这里赋值给了 threshold 呢？
    //先卖个关子，等到 resize 的时候再说
    this.threshold = tableSizeFor(initialCapacity);
}

//4.可传入一个已有的map
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

###  putMapEntries()

- putMapEntries(Map<? extends K, ? extends V> m, boolean evict)

```java
//把传入的map里边的元素都加载到当前map
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        if (table == null) { // pre-size
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        else if (s > threshold)
            resize();
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            //put方法的具体实现，后边讲
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```

###  tableSizeFor()

上边的第三个构造函数中，调用了 tableSizeFor 方法，这个方法是怎么实现的呢？

```java
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

我们以传入参数为14 来举例，计算这个过程。

首先，14传进去之后先减1，n此时为13。然后是一系列的无符号右移运算。

```java
//13的二进制
0000 0000 0000 0000 0000 0000 0000 1101 
//无右移1位，高位补0
0000 0000 0000 0000 0000 0000 0000 0110 
//然后把它和原来的13做或运算得到，此时的n值
0000 0000 0000 0000 0000 0000 0000 1111 
//再以上边的值，右移2位
0000 0000 0000 0000 0000 0000 0000 0011
//然后和上一次或运算之后的 n 值再做或运算，此时得到的n值
0000 0000 0000 0000 0000 0000 0000 1111
//再以上边的值，右移4位
0000 0000 0000 0000 0000 0000 0000 0000
//然后和上一次或运算之后的 n 值再做或运算，此时得到的n值
0000 0000 0000 0000 0000 0000 0000 1111
...
//我们会发现，再执行右移 8,16位，同样n的值不变
//当n小于0时，返回1，否则判断是否大于最大容量，是的话返回最大容量，否则返回 n+1
return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
//很明显我们这里返回的是 n+1 的值，
0000 0000 0000 0000 0000 0000 0000 1111
+                                     1
0000 0000 0000 0000 0000 0000 0001 0000
```

将它转为十进制，就是 2^4 = 16 。我们会发现一个规律，以上的右移运算，最终会把最低位的值都转化为 1111 这样的结构，然后再加1，就是1 0000 这样的结构，它一定是 2的n次幂。因此，这个方法返回的就是大于当前传入值的最小（最接近当前值）的一个2的n次幂的值。

##  核心方法

- put(K key, V value)
- putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict)
- get(Object key)
- getNode(int hash, Object key)
- resize()
- treeifyBin(Node<K,V>[] tab, int hash)
- containsKey(Object key)
- containsValue(Object value)

###  put()

**概述**

Map的put方法接受两个参数，key和value，该方法用于存储键值对。

HashMap的put方法只有一行代码

```java
return putVal(hash(key), key, value, false, true); //参见：hash方法解析
```

```java
static final int hash(Object key) {
   int h;
   return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

首选对key值进行hash算法，key的hash值高16位不变，低16位与高16位异或作为key的最终hash值。（h >>> 16，表示无符号右移16位，高位补0，任何数跟0异或都是其本身，因此key的hash值高16位是不变的。）

**方法解析**

put方法是一个方便用户使用的快捷方式，具体逻辑都是在putVal方法中实现的，我们就针对putVal方法的实现来做解析。

首先判断table是不是为空，如果为空，说明是第一次执行put操作，需要确认数组的大小，从源码中可以看到，数组的初始大小是16，最大值为Integer.max_value。数组的大小必须是2的n次方。然后需要确认数据落在数组的哪一个位置上，也就是确认数组下标，确认数组下标是通过hash&（数组大小-1）相当于取余数的方式，如果数组下标位置没有存在其他节点，那么直接放入数组桶中（如图中的node0、node2），如果发生了碰撞（数组下标位置存在其他节点），如果key已经存在，说明节点已经存在，将对应节点的旧value换成新value，如果不存在将node链接到链表的后面（如图中的node1和node3）。

![img](https://gitee.com/HNov/image/raw/master/typora/20200417003843.png)

hashmap如果数据变大，数组是可以扩充的，常量定义了一个负载因子，默认是0.75，也就是说当数组的元素个数大于了扩容阀值(16*0.75=12)的时候，数组就开始扩充，扩充的大小是原来的两倍，因为要保证数组大小是2的n次方。如果链表过长会影响查询速度，jdk1.8对此做了改进，有一个常量`TREEIFY_THRESHOLD=8`和`UNTREEIFY_THRESHOLD=6`，如果链表的长度大于TREEIFY_THRESHOLD=8时，链表会转换成红黑树。如果执行remove操作的时候，红黑树节点又会变少，如果节点小于UNTREEIFY_THRESHOLD=6时，又会从红黑树转成链表。


```java
static final float DEFAULT_LOAD_FACTOR = 0.75f; // 负载因子
```

```java
 /**
     * @param hash         key的hash值
     * @param key          键
     * @param value        值
     * @param onlyIfAbsent 设为true表示如果键不存在，才会写入值。
     * @param evict
     * @return 返回value
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K, V>[] tab;
        Node<K, V> p;
        int n, i; // 定义元素数组、当前元素变量
        // 如果当前Map的元素数组为空 或者 数组长度为0，那么需要初始化元素数组
        // tab = resize() 初始化了元素数组，resize方法同时也可以实现数组扩容，可参见：resize方法解析
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length; // n 设置为 数组长度
        // 根据hash值和数组长度取摸计算出数组下标
        if ((p = tab[i = (n - 1) & hash]) == null)  // 如果该位置不存在元素，那么创建一个新元素存储到数组的该位置。
            tab[i] = newNode(hash, key, value, null); // 此处单独解析
        //如果当前位置已经有元素了，分为三种情况。
        else {
            // e 用来指向根据key匹配到的元素
            Node<K, V> e;
            K k;
            //1.当前位置元素的hash值等于传过来的hash，并且他们的key值也相等，
            //则把p赋值给e，跳转到①处，后续需要做值的覆盖处理
            if (p.hash == hash &&
                    ((k = p.key) == key || (key != null && key.equals(k))))
                e = p; // 用e指向到当前元素

            //2.如果当前是红黑树结构，则把它加入到红黑树
            // 如果要写入的key的hash值和当前元素的key的hash值不同，或者key不相等，说明不是同一个key，要通过其他数据结构来存储新来的数据
            else if (p instanceof TreeNode)
                e = ((TreeNode<K, V>) p).putTreeVal(this, tab, hash, key, value); // 参见：putTreeVal方法解析
            else {
                //3.说明此位置已存在元素，并且是普通链表结构，则采用尾插法，把新节点加入到链表尾部
                // 运行到这里，说明采用链表结构来存储
                for (int binCount = 0; ; ++binCount) {  // 要逐一对比看要写入的key是否和链表上的某个key相同
                    if ((e = p.next) == null) { // 如果当前元素没有下一个节点
                        // 根据键值对创建一个新节点，挂到链表的尾部
                        p.next = newNode(hash, key, value, null);
                        //  如果新增节点前链表上元素的个数已经达到了阀值（可以改变存储结构的临界值），其实在这这一时刻当前链表上的元素已经达到了9个。
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            // 将该链表上所有元素改为TreeNode方式存储（是为了增加查询性能，元素越多，链表的查询性能越差） 或者 扩容
                            treeifyBin(tab, hash); // 参见：treeifyBin方法解析
                        break;// 跳出循环，因为没有可遍历的元素了
                    }
                    // 如果下一个节点的 hash值和key值都和要写入的hash 和 key相同
                    if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k))))
                        break;    // 跳出循环，因为找到了相同的key对应的元素
                    p = e;
                }
            }
            //① 此时e有两种情况
            //1.说明发生了碰撞，e代表的是旧值，因此节点位置不变，但是需要替换为新值
            //2.说明e是插入链表或者红黑树，成功后的新节点
            if (e != null) { // 说明找了和要写入的key对应的元素，根据情况来决定是否覆盖值
                V oldValue = e.value;
                //用新值替换旧值，并返回旧值。
                //oldValue为空，说明e是新增的节点或者也有可能旧值本来就是空的，因为hashmap可存空值
                if (!onlyIfAbsent || oldValue == null)    // 如果旧值为空  后者  指定了需要覆盖旧值，那么更改元素的值为新值
                    e.value = value;
                //看方法名字即可知，这是在node被访问之后需要做的操作。其实此处是一个空实现，
                //只有在 LinkedHashMap才会实现，用于实现根据访问先后顺序对元素进行排序，hashmap不提供排序功能
                afterNodeAccess(e);
                return oldValue; // 返回旧值
            }
        }

        // 执行到这里，说明是增加了新的元素，而不是替换了老的元素，所以相关计数需要累加

        //fail-fast机制
        ++modCount; // 修改计数器递增
        // 当前map的元素个数递增
        if (++size > threshold) // 如果当前map的元素个数大于了扩容阀值，那么需要扩容元素数组了
            resize(); // 元素数组扩容
        afterNodeInsertion(evict); // 添加新元素之后的后后置处理， LinkedHashMap中有具体实现
        return null; // 返回空
    }
```

###  putTreeVal()

**概述**

我们都知道，目前HashMap是采用数组+链表+红黑树的方式来存储和组织数据的。

在put数据的时候，根据键的hash值寻址到具体数组位置，如果不存在hash碰撞，那么这个位置就只存储这么一个键值对。参见：<a href="#put()">put方法分析</a>

如果两个key的hash值相同，那么对应数组位置上就需要用链表的方式将这两个数据组织起来，当同一个位置上链表中的元素达到9个(8+1 原链表上的8个元素+新put的元素 )的时候，就会再将这些元素构建成一个红黑树（参见：[treeifyBin方法分析](#treeifyBin())，同时把原来的单链表结构变成了双链表结构，也就是这些元素即维持着红黑树的结构又维持着双链表的结构。当新的相同hash值的键值对put过来时，发现该位置已经是一个树节点了，那么就会调用putTreeVal方法，将这个新的值设置到指定的key上。

**方法解析**

```java
 /**
     * 当存在hash碰撞的时候，且元素数量大于8个时候，就会以红黑树的方式将这些元素组织起来
     * map 当前节点所在的HashMap对象
     * tab 当前HashMap对象的元素数组
     * h   指定key的hash值
     * k   指定key
     * v   指定key上要写入的值
     * 返回：指定key所匹配到的节点对象，针对这个对象去修改V（返回空说明创建了一个新节点）
     */
    final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                   int h, K k, V v) {
        Class<?> kc = null; // 定义k的Class对象
        boolean searched = false; // 标识是否已经遍历过一次树，未必是从根节点遍历的，但是遍历路径上一定已经包含了后续需要比对的所有节点。
        TreeNode<K,V> root = (parent != null) ? root() : this; // 父节点不为空那么查找根节点，为空那么自身就是根节点
        for (HashMap.TreeNode<K,V> p = root;;) {
            int dir, ph;
            K pk; // 声明方向、当前节点hash值、当前节点的键对象
            if ((ph = p.hash) > h) // 如果当前节点hash 大于 指定key的hash值
                dir = -1; // 要添加的元素应该放置在当前节点的左侧
            else if (ph < h) // 如果当前节点hash 小于 指定key的hash值
                dir = 1; // 要添加的元素应该放置在当前节点的右侧
            else if ((pk = p.key) == k || (k != null && k.equals(pk))) // 如果当前节点的键对象 和 指定key对象相同
                return p; // 那么就返回当前节点对象，在外层方法会对v进行写入

                // 走到这一步说明 当前节点的hash值  和 指定key的hash值  是相等的，但是equals不等
            else if ((kc == null &&
                    (kc = comparableClassFor(k)) == null) ||
                    (dir = compareComparables(kc, k, pk)) == 0) {

                // 走到这里说明：指定key没有实现comparable接口   或者   实现了comparable接口并且和当前节点的键对象比较之后相等（仅限第一次循环）


                /*
                 * searched 标识是否已经对比过当前节点的左右子节点了
                 * 如果还没有遍历过，那么就递归遍历对比，看是否能够得到那个键对象equals相等的的节点
                 * 如果得到了键的equals相等的的节点就返回
                 * 如果还是没有键的equals相等的节点，那说明应该创建一个新节点了
                 */
                if (!searched) { // 如果还没有比对过当前节点的所有子节点
                    TreeNode<K, V> q, ch; // 定义要返回的节点、和子节点
                    searched = true; // 标识已经遍历过一次了
                    /*
                     * 红黑树也是二叉树，所以只要沿着左右两侧遍历寻找就可以了
                     * 这是个短路运算，如果先从左侧就已经找到了，右侧就不需要遍历了
                     * find 方法内部还会有递归调用。参见：find方法解析
                     */
                    if (((ch = p.left) != null &&
                            (q = ch.find(h, k, kc)) != null) ||
                            ((ch = p.right) != null &&
                                    (q = ch.find(h, k, kc)) != null))
                        return q; // 找到了指定key键对应的
                }

                // 走到这里就说明，遍历了所有子节点也没有找到和当前键equals相等的节点
                dir = tieBreakOrder(k, pk); // 再比较一下当前节点键和指定key键的大小
            }

            TreeNode<K, V> xp = p; // 定义xp指向当前节点
            /*
             * 如果dir小于等于0，那么看当前节点的左节点是否为空，如果为空，就可以把要添加的元素作为当前节点的左节点，如果不为空，还需要下一轮继续比较
             * 如果dir大于等于0，那么看当前节点的右节点是否为空，如果为空，就可以把要添加的元素作为当前节点的右节点，如果不为空，还需要下一轮继续比较
             * 如果以上两条当中有一个子节点不为空，这个if中还做了一件事，那就是把p已经指向了对应的不为空的子节点，开始下一轮的比较
             */
            if ((p = (dir <= 0) ? p.left : p.right) == null) {
                // 如果恰好要添加的方向上的子节点为空，此时节点p已经指向了这个空的子节点
                Node<K, V> xpn = xp.next; // 获取当前节点的next节点
                TreeNode<K, V> x = map.newTreeNode(h, k, v, xpn); // 创建一个新的树节点
                if (dir <= 0)
                    xp.left = x;  // 左孩子指向到这个新的树节点
                else
                    xp.right = x; // 右孩子指向到这个新的树节点
                xp.next = x; // 链表中的next节点指向到这个新的树节点
                x.parent = x.prev = xp; // 这个新的树节点的父节点、前节点均设置为 当前的树节点
                if (xpn != null) // 如果原来的next节点不为空
                    ((TreeNode<K, V>) xpn).prev = x; // 那么原来的next节点的前节点指向到新的树节点
                moveRootToFront(tab, balanceInsertion(root, x));// 重新平衡，以及新的根节点置顶
                return null; // 返回空，意味着产生了一个新节点
            }
        }
    }
```

###  treeifyBin()

**概述**

treeifyBin方法，应该可以解释为：把容器里的元素变成树结构。当HashMap的内部元素数组中某个位置上存在多个hash值相同的键值对，这些Node已经形成了一个链表，当该链表的长度大于等于9（为什么是9？TREEIFY_THRESHOLD默认值为8呀？[详见put方法解析](#put())的时候，会调用该方法来进行一个特殊处理。

**方法解析**

```java
/**
 * tab：元素数组，
 * hash：hash值（要增加的键值对的key的hash值）
 */
final void treeifyBin(Node<K,V>[] tab, int hash) {
 
    int n, index; Node<K,V> e;
    /*
     * 如果元素数组为空 或者 数组长度小于 树结构化的最小限制
     * MIN_TREEIFY_CAPACITY 默认值64，对于这个值可以理解为：如果元素数组长度小于这个值，没有必要去进行结构转换
     * 当一个数组位置上集中了多个键值对，那是因为这些key的hash值和数组长度取模之后结果相同。（并不是因为这些key的hash值相同）
     * 因为hash值相同的概率不高，所以可以通过扩容的方式，来使得最终这些key的hash值在和新的数组长度取模之后，拆分到多个数组位置上。
     */
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize(); // 扩容，可参见resize方法解析

 
    // 如果元素数组长度已经大于等于了 MIN_TREEIFY_CAPACITY，那么就有必要进行结构转换了
    // 根据hash值和数组长度进行取模运算后，得到链表的首节点
    else if ((e = tab[index = (n - 1) & hash]) != null) { 
        TreeNode<K,V> hd = null, tl = null; // 定义首、尾节点
        do { 
            TreeNode<K,V> p = replacementTreeNode(e, null); // 将该节点转换为 树节点
            if (tl == null) // 如果尾节点为空，说明还没有根节点
                hd = p; // 首节点（根节点）指向 当前节点
            else { // 尾节点不为空，以下两行是一个双向链表结构
                p.prev = tl; // 当前树节点的 前一个节点指向 尾节点
                tl.next = p; // 尾节点的 后一个节点指向 当前节点
            }
            tl = p; // 把当前节点设为尾节点
        } while ((e = e.next) != null); // 继续遍历链表
 
        // 到目前为止 也只是把Node对象转换成了TreeNode对象，把单向链表转换成了双向链表
 
        // 把转换后的双向链表，替换原来位置上的单向链表
        if ((tab[index] = hd) != null)
            hd.treeify(tab);//此处可参见treeify方法解析
    }
}
```

### resize()

**概述**

HashMap的resize方法的作用：在向HashMap里put元素的时候，HashMap基于扩容规则发现需要扩容的时候会调用该方法来进行扩容。

**总结：`HashMap` 扩容后，原来的元素，要么在原位置(低位)，要么在`原位置+原数组长度` 那个位置上（高位）。**

**方法解析**

```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table; //当前所有元素所在的数组，称为老的元素数组
        int oldCap = (oldTab == null) ? 0 : oldTab.length; //老的元素数组长度
        int oldThr = threshold;    // 老的扩容阀值设置
        int newCap, newThr = 0;    // 新数组的容量，新数组的扩容阀值都初始化为0
        if (oldCap > 0) {  // 如果老数组长度大于0，说明已经存在元素
            // PS1
            if (oldCap >= MAXIMUM_CAPACITY) { // 如果数组元素个数大于等于限定的最大容量（2的30次方）
                // 扩容阀值设置为int最大值（2的31次方 -1 ），因为oldCap再乘2就溢出了。
                threshold = Integer.MAX_VALUE;
                return oldTab; // 返回老的元素数组
            }
 
           /*
            * 如果数组元素个数在正常范围内，那么新的数组容量为老的数组容量的2倍（左移1位相当于乘以2）
            * 如果扩容之后的新容量小于最大容量  并且  老的数组容量大于等于默认初始化容量（16），那么新数组的扩容阀值设置为老阀值的2倍。（老的数组容量大于16意味着：要么构造函数指定了一个大于16的初始化容量值，要么已经经历过了至少一次扩容）
            */
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
 
        // PS2
        // 运行到这个else if  说明老数组没有任何元素
        // 如果老数组的扩容阀值大于0，那么设置新数组的容量为该阀值
        // 这一步也就意味着构造该map的时候，指定了初始化容量。
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            // 能运行到这里的话，说明是调用无参构造函数创建的该map，并且第一次添加元素
            newCap = DEFAULT_INITIAL_CAPACITY; // 设置新数组容量 为 16
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY); // 设置新数组扩容阀值为 16*0.75 = 12。0.75为负载因子（当元素个数达到容量了4分之3，那么扩容）
        }
 
        // 如果扩容阀值为0 （PS2的情况）
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);  // 参见：PS2
        }
        threshold = newThr; // 设置map的扩容阀值为 新的阀值
        @SuppressWarnings({"rawtypes","unchecked"})
            // 创建新的数组（对于第一次添加元素，那么这个数组就是第一个数组；对于存在oldTab的时候，那么这个数组就是要需要扩容到的新数组）
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;    // 将该map的table属性指向到该新数组
        if (oldTab != null) {  // 如果老数组不为空，说明是扩容操作，那么涉及到元素的转移操作
            for (int j = 0; j < oldCap; ++j) { // 遍历老数组
                Node<K,V> e;
                if ((e = oldTab[j]) != null) { // 如果当前位置元素不为空，那么需要转移该元素到新数组
                    oldTab[j] = null; // 释放掉老数组对于要转移走的元素的引用（主要为了使得数组可被回收）
                    if (e.next == null) // 如果元素没有有下一个节点，说明该元素不存在hash冲突
                        // PS3
                        // 把元素存储到新的数组中，存储到数组的哪个位置需要根据hash值和数组长度来进行取模
                        // 【hash值  %   数组长度】   =    【  hash值   & （数组长度-1）】
                        //  这种与运算求模的方式要求  数组长度必须是2的N次方，但是可以通过构造函数随意指定初始化容量呀，如果指定了17,15这种，岂不是出问题了就？没关系，最终会通过tableSizeFor方法将用户指定的转化为大于其并且最相近的2的N次方。 15 -> 16、17-> 32
                    newTab[e.hash & (newCap - 1)] = e;
 
                        // 如果该元素有下一个节点，那么说明该位置上存在一个链表了（hash相同的多个元素以链表的方式存储到了老数组的这个位置上了）
                        // 例如：数组长度为16，那么hash值为1（1%16=1）的和hash值为17（17%16=1）的两个元素都是会存储在数组的第2个位置上（对应数组下标为1），当数组扩容为32（1%32=1）时，hash值为1的还应该存储在新数组的第二个位置上，但是hash值为17（17%32=17）的就应该存储在新数组的第18个位置上了。
                        // 所以，数组扩容后，所有元素都需要重新计算在新数组中的位置。
 
 
                    else if (e instanceof TreeNode)  // 如果该节点为TreeNode类型
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);  // 此处单独展开讨论
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;  // 按命名来翻译的话，应该叫低位首尾节点
                        Node<K,V> hiHead = null, hiTail = null;  // 按命名来翻译的话，应该叫高位首尾节点
                        // 以上的低位指的是新数组的 0  到 oldCap-1 、高位指定的是oldCap 到 newCap - 1
                        Node<K,V> next;
                        // 遍历链表
                        do {
                            next = e.next;
                            // 这一步判断好狠，拿元素的hash值  和  老数组的长度  做与运算
                            // PS3里曾说到，数组的长度一定是2的N次方（例如16），如果hash值和该长度做与运算，那么该hash值可参与计算的有效二进制位就是和长度二进制对等的后几位，如果结果为0，说明hash值中参与计算的对等的二进制位的最高位一定为0.
                            //因为数组长度的二进制有效最高位是1（例如16对应的二进制是10000），只有*..0**** 和 10000 进行与运算结果才为00000（*..表示不确定的多个二进制位）。又因为定位下标时的取模运算是以hash值和长度减1进行与运算，所以下标 = (*..0**** & 1111) 也= (*..0**** & 11111) 。1111是15的二进制、11111是16*2-1 也就是31的二级制（2倍扩容）。
                            // 所以该hash值再和新数组的长度取摸的话mod值也不会放生变化，也就是说该元素的在新数组的位置和在老数组的位置是相同的，所以该元素可以放置在低位链表中。
                            if ((e.hash & oldCap) == 0) {  
                                // PS4
                                if (loTail == null) // 如果没有尾，说明链表为空
                                    loHead = e; // 链表为空时，头节点指向该元素
                                else
                                    loTail.next = e; // 如果有尾，那么链表不为空，把该元素挂到链表的最后。
                                loTail = e; // 把尾节点设置为当前元素
                            }
 
                            // 如果与运算结果不为0，说明hash值大于老数组长度（例如hash值为17）
                            // 此时该元素应该放置到新数组的高位位置上
                            // 例：老数组长度16，那么新数组长度为32，hash为17的应该放置在数组的第17个位置上，也就是下标为16，那么下标为16已经属于高位了，低位是[0-15]，高位是[16-31]
                            else {  // 以下逻辑同PS4
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) { // 低位的元素组成的链表还是放置在原来的位置
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {  // 高位的元素组成的链表放置的位置只是在原有位置上偏移了老数组的长度个位置。
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead; // 例：hash为 17 在老数组放置在0下标，在新数组放置在16下标；    hash为 18 在老数组放置在1下标，在新数组÷放置在17下标；                   
                        }
                    }
                }
            }
        }
        return newTab; // 返回新数组
    }
```



#### resize过程图解

经过上面的代码分析可以发现，我们使用的是2次幂的扩展(指长度扩为原来2倍)，所以，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置。看下图可以明白这句话的意思，n为table的长度，图（a）表示扩容前的key1和key2两种key确定索引位置的示例，图（b）表示扩容后key1和key2两种key确定索引位置的示例，其中hash1是key1对应的哈希与高位运算结果。

![img](https://gitee.com/HNov/image/raw/master/typora/20200516152727.png)

元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：

![img](https://gitee.com/HNov/image/raw/master/typora/20200516153129.png)

因此，我们在扩充HashMap的时候，不需要像JDK1.7的实现那样重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”，可以看看下图为16扩充为32的resize示意图.

![img](https://gitee.com/HNov/image/raw/master/typora/20200516152745.png)

这个设计确实非常的巧妙，既省去了重新计算hash值的时间，而且同时，由于新增的1bit是0还是1可以认为是随机的，因此resize的过程，均匀的把之前的冲突的节点分散到新的bucket了。这一块就是JDK1.8新增的优化点。有一点注意区别，JDK1.7中rehash的时候，旧链表迁移新链表的时候，如果在新表的数组索引位置相同，则链表元素会倒置，但是从上图可以看出，JDK1.8不会倒置。



### replacementTreeNode()—构建红黑树的节点对象

**概述**

  HashMap中每一个键值对都是以一个HashMap.Node<K,V>对象实例进行存储的，当不同的键对应的多个键值对路由到元素数组的同一个位置时，这多个键值对对应的Node对象之间会以链表的方式组织起来。这是默认的组织方式。

  当我们恰好要根据key寻找一个在链表上的对象的时候，就涉及到遍历链表，逐个调用key对象的equals方法来比对我们要查找的到底是哪个键值对了。可想当链表的长度越长，匹配的时间复杂度就越高，和链表的长度成正比。这也就是HashMap内部实现时会根据链表的长度超过限定的阈值时，会将链表结构转换为红黑树结构，用来提升查询性能。replacementTreeNode方法就是此时被调用。TreeNode类也就是红黑树的节点对象。

**方法解析**

```java
// 该方法只是调用了TreeNode类的构造方法，依据当前节点信息构造一个树节点对象
TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
    return new TreeNode<>(p.hash, p.key, p.value, next);
}
```

接下来看下TreeNode类的定义

```java
/**
 * 首先该类是HashMap类的一个静态内部类
 * 包内可见、不可被继承
 * 该类继承了LinkedHashMap.Entry，而LinkedHashMap.Entry继承了HashMap.Node
 * PS：要知道LinkedHashMap是HashMap的子类，然而目前的状况是HashMap作为父类，他的一个静态内部类（TreeNode）居然继承了子类LinkedHashMap的一个静态内部类
 *（LinkedHashMap.Entry），这个设计不太理解。
 * 红黑树是一个二叉树，父节点、左节点、右节点、红黑标识都是二叉树中的元素
 */
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next); // 此处调用 LinkedHashMap.Entry的构造方法
    }
 
    // 以下省略了很多方法，后文会逐步解析
}
```



接下来再看下LinkedHashMap.Entry类的定义

```java
/**
 * 首先该类是LinkedHashMap的一个静态内部类
 * 包内可见、又因为没有final修饰符（所以HashMap中的TreeNode类才能继承到他）
 * 该类除了增加了before、after两个实例变量之外，没有任何的行为扩展，也就是说他的所有行为都继承自HashMap.Node
 * 该类也只有一个构造方法，且该构造方法就是通过调用HashMap.Node的构造方法构造一个HashMap.Node对象
 * PS：看到这里就更加不理解为何HashMap.TreeNode不直接继承HashMap.Node，而要绕个弯来继承LinkedHashMap.Entry，难道是为了使用before、after？可貌似也没有使用到。
 */
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

### newNode方法、Node类

**概述**

在Map中存储的每一个键值对都是以一个Map.Entry<K,V>的实现对象存储的，Map.Entry是一个接口，不能实例化，所以Map的不同实现类，存储键值对的方式也会有所不同，只要这个键值对的类实现了Map.Entry接口即可。

在HashMap中键值对的存储方式有两种：

其中一种是我们比较熟知的链表存储结构，也就是以HashMap.Node<K,V>类的对象方式存储的， Node类是HashMap的一个静态内部类，实现了 Map.Entry<K,V>接口。在调用put方法创建一个新的键值对时，会调用newNode方法来创建Node对象。

**方法解析**

```java
// 该方法很简单，只是调用了Node类的构造函数
Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
        return new Node<>(hash, key, value, next);
}
```

我们接下来看下Node类的代码

```java
/**
* 该类只实现了 Map.Entry 接口，
* 所以该类只需要实现getKey、getValue、setValue三个方法即可
* 除此之外以什么样的方式来组织数据，就和接口无关了
*/
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash; // hash值，不可变
        final K key; // 键，不可变
        V value; // 值
        Node<K,V> next; // 下一个节点
 
        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
 
        // 实现接口定义的方法，且该方法不可被重写
        public final K getKey()        { return key; }
        // 实现接口定义的方法，且该方法不可被重写
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }
 
        // 重写父类Object的hashCode方法，且该方法不可被自己的子类再重写
        // 返回：key的hashCode值和value的hashCode值进行异或运算结果
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }
 
        // 实现接口定义的方法，且该方法不可被重写
        // 设值，返回旧值
        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }
 
        /*
         * 重写父类Object的equals方法，且该方法不可被自己的子类再重写
         * 判断相等的依据是，只要是Map.Entry的一个实例，并且键键、值值都相等就返回True
         */
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



### TreeNode类的treeify方法

**概述**

treeify方法是TreeNode类的一个实例方法，通过TreeNode对象调用，实现该对象打头的链表转换为树结构。

**方法解析**

```java
/**
 * 参数为HashMap的元素数组
 */
final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null; // 定义树的根节点
    for (TreeNode<K,V> x = this, next; x != null; x = next) { // 遍历链表，x指向当前节点、next指向下一个节点
        next = (TreeNode<K,V>)x.next; // 下一个节点
        x.left = x.right = null; // 设置当前节点的左右节点为空
        if (root == null) { // 如果还没有根节点
            x.parent = null; // 当前节点的父节点设为空
            x.red = false; // 当前节点的红色属性设为false（把当前节点设为黑色）
            root = x; // 根节点指向到当前节点
        }
        else { // 如果已经存在根节点了
            K k = x.key; // 取得当前链表节点的key
            int h = x.hash; // 取得当前链表节点的hash值
            Class<?> kc = null; // 定义key所属的Class
            for (TreeNode<K,V> p = root;;) { // 从根节点开始遍历，此遍历没有设置边界，只能从内部跳出
                // GOTO1
                int dir, ph; // dir 标识方向（左右）、ph标识当前树节点的hash值
                K pk = p.key; // 当前树节点的key
                if ((ph = p.hash) > h) // 如果当前树节点hash值 大于 当前链表节点的hash值
                    dir = -1; // 标识当前链表节点会放到当前树节点的左侧
                else if (ph < h)
                    dir = 1; // 右侧
 
                /*
                 * 如果两个节点的key的hash值相等，那么还要通过其他方式再进行比较
                 * 如果当前链表节点的key实现了comparable接口，并且当前树节点和链表节点是相同Class的实例，那么通过comparable的方式再比较两者。
                 * 如果还是相等，最后再通过tieBreakOrder比较一次
                 */
                else if ((kc == null &&
                            (kc = comparableClassFor(k)) == null) ||
                            (dir = compareComparables(kc, k, pk)) == 0)
                    dir = tieBreakOrder(k, pk);
 
                TreeNode<K,V> xp = p; // 保存当前树节点
 
                /*
                 * 如果dir 小于等于0 ： 当前链表节点一定放置在当前树节点的左侧，但不一定是该树节点的左孩子，也可能是左孩子的右孩子 或者 更深层次的节点。
                 * 如果dir 大于0 ： 当前链表节点一定放置在当前树节点的右侧，但不一定是该树节点的右孩子，也可能是右孩子的左孩子 或者 更深层次的节点。
                 * 如果当前树节点不是叶子节点，那么最终会以当前树节点的左孩子或者右孩子 为 起始节点  再从GOTO1 处开始 重新寻找自己（当前链表节点）的位置
                 * 如果当前树节点就是叶子节点，那么根据dir的值，就可以把当前链表节点挂载到当前树节点的左或者右侧了。
                 * 挂载之后，还需要重新把树进行平衡。平衡之后，就可以针对下一个链表节点进行处理了。
                 */
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp; // 当前链表节点 作为 当前树节点的子节点
                    if (dir <= 0)
                        xp.left = x; // 作为左孩子
                    else
                        xp.right = x; // 作为右孩子
                    root = balanceInsertion(root, x); // 重新平衡
                    break;
                }
            }
        }
    }
 
    // 把所有的链表节点都遍历完之后，最终构造出来的树可能经历多个平衡操作，根节点目前到底是链表的哪一个节点是不确定的
    // 因为我们要基于树来做查找，所以就应该把 tab[N] 得到的对象一定根节点对象，而目前只是链表的第一个节点对象，所以要做相应的处理。
    moveRootToFront(tab, root); // 单独解析
}
```

### TreeNode类的balanceInsertion方法

**概述**

balanceInsertion指的是红黑树的插入平衡算法，当树结构中新插入了一个节点后，要对树进行重新的结构化，以保证该树始终维持红黑树的特性。

关于红黑树的特性：

性质1. 节点是红色或黑色。

性质2. 根节点是黑色。

性质3 每个叶节点（NIL节点，空节点）是黑色的。

性质4 每个红色节点的两个子节点都是黑色。(从每个叶子到根的所有路径上不能有两个连续的红色节点)

性质5. 从任一节点到其每个叶子的路径上包含的黑色节点数量都相同。

**方法解析**

```java
/**
 * 红黑树插入节点后，需要重新平衡
 * root 当前根节点
 * x 新插入的节点
 * 返回重新平衡后的根节点
 */
static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                                    TreeNode<K,V> x) {
    x.red = true; // 新插入的节点标为红色
 
    /*
     * 这一步即定义了变量，又开起了循环，循环没有控制条件，只能从内部跳出
     * xp：当前节点的父节点、xpp：爷爷节点、xppl：左叔叔节点、xppr：右叔叔节点
     */
    for (TreeNode<K,V> xp, xpp, xppl, xppr;;) { 
 
        // 如果父节点为空、说明当前节点就是根节点，那么把当前节点标为黑色，返回当前节点
        if ((xp = x.parent) == null) { // L1
            x.red = false;
            return x;
        }
 
        // 父节点不为空
        // 如果父节点为黑色 或者 【（父节点为红色 但是 爷爷节点为空） -> 这种情况何时出现？】
        else if (!xp.red || (xpp = xp.parent) == null) // L2
            return root;
        if (xp == (xppl = xpp.left)) { // 如果父节点是爷爷节点的左孩子  // L3
            if ((xppr = xpp.right) != null && xppr.red) { // 如果右叔叔不为空 并且 为红色  // L3_1
                xppr.red = false; // 右叔叔置为黑色
                xp.red = false; // 父节点置为黑色
                xpp.red = true; // 爷爷节点置为红色
                x = xpp; // 运行到这里之后，就又会进行下一轮的循环了，将爷爷节点当做处理的起始节点 
            }
            else { // 如果右叔叔为空 或者 为黑色 // L3_2
                if (x == xp.right) { // 如果当前节点是父节点的右孩子 // L3_2_1
                    root = rotateLeft(root, x = xp); // 父节点左旋，见下文左旋方法解析
                    xpp = (xp = x.parent) == null ? null : xp.parent; // 获取爷爷节点
                }
                if (xp != null) { // 如果父节点不为空 // L3_2_2
                    xp.red = false; // 父节点 置为黑色
                    if (xpp != null) { // 爷爷节点不为空
                        xpp.red = true; // 爷爷节点置为 红色
                        root = rotateRight(root, xpp);  //爷爷节点右旋，见下文右旋方法解析
                    }
                }
            }
        }
        else { // 如果父节点是爷爷节点的右孩子 // L4
            if (xppl != null && xppl.red) { // 如果左叔叔是红色 // L4_1
                xppl.red = false; // 左叔叔置为 黑色
                xp.red = false; // 父节点置为黑色
                xpp.red = true; // 爷爷置为红色
                x = xpp; // 运行到这里之后，就又会进行下一轮的循环了，将爷爷节点当做处理的起始节点 
            }
            else { // 如果左叔叔为空或者是黑色 // L4_2
                if (x == xp.left) { // 如果当前节点是个左孩子 // L4_2_1
                    root = rotateRight(root, x = xp); // 针对父节点做右旋，见下文右旋方法解析
                    xpp = (xp = x.parent) == null ? null : xp.parent; // 获取爷爷节点
                }
                if (xp != null) { // 如果父节点不为空 // L4_2_4
                    xp.red = false; // 父节点置为黑色
                    if (xpp != null) { //如果爷爷节点不为空
                        xpp.red = true; // 爷爷节点置为红色
                        root = rotateLeft(root, xpp); // 针对爷爷节点做左旋
                    }
                }
            }
        }
    }
}
 
 
/**
 * 节点左旋
 * root 根节点
 * p 要左旋的节点
 */
static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
                                              TreeNode<K,V> p) {
    TreeNode<K,V> r, pp, rl;
    if (p != null && (r = p.right) != null) { // 要左旋的节点以及要左旋的节点的右孩子不为空
        if ((rl = p.right = r.left) != null) // 要左旋的节点的右孩子的左节点 赋给 要左旋的节点的右孩子 节点为：rl
            rl.parent = p; // 设置rl和要左旋的节点的父子关系【之前只是爹认了孩子，孩子还没有答应，这一步孩子也认了爹】
 
        // 将要左旋的节点的右孩子的父节点  指向 要左旋的节点的父节点，相当于右孩子提升了一层，
        // 此时如果父节点为空， 说明r 已经是顶层节点了，应该作为root 并且标为黑色
        if ((pp = r.parent = p.parent) == null) 
            (root = r).red = false;
        else if (pp.left == p) // 如果父节点不为空 并且 要左旋的节点是个左孩子
            pp.left = r; // 设置r和父节点的父子关系【之前只是孩子认了爹，爹还没有答应，这一步爹也认了孩子】
        else // 要左旋的节点是个右孩子
            pp.right = r; 
        r.left = p; // 要左旋的节点  作为 他的右孩子的左节点
        p.parent = r; // 要左旋的节点的右孩子  作为  他的父节点
    }
    return root; // 返回根节点
}
 
/**
 * 节点右旋
 * root 根节点
 * p 要右旋的节点
 */
static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,
                                               TreeNode<K,V> p) {
    TreeNode<K,V> l, pp, lr;
    if (p != null && (l = p.left) != null) { // 要右旋的节点不为空以及要右旋的节点的左孩子不为空
        if ((lr = p.left = l.right) != null) // 要右旋的节点的左孩子的右节点 赋给 要右旋节点的左孩子 节点为：lr
            lr.parent = p; // 设置lr和要右旋的节点的父子关系【之前只是爹认了孩子，孩子还没有答应，这一步孩子也认了爹】
 
        // 将要右旋的节点的左孩子的父节点  指向 要右旋的节点的父节点，相当于左孩子提升了一层，
        // 此时如果父节点为空， 说明l 已经是顶层节点了，应该作为root 并且标为黑色
        if ((pp = l.parent = p.parent) == null) 
            (root = l).red = false;
        else if (pp.right == p) // 如果父节点不为空 并且 要右旋的节点是个右孩子
            pp.right = l; // 设置l和父节点的父子关系【之前只是孩子认了爹，爹还没有答应，这一步爹也认了孩子】
        else // 要右旋的节点是个左孩子
            pp.left = l; // 同上
        l.right = p; // 要右旋的节点 作为 他左孩子的右节点
        p.parent = l; // 要右旋的节点的父节点 指向 他的左孩子
    }
    return root;
}
```

**图例**

- 无旋转

![img](https://gitee.com/HNov/image/raw/master/typora/20200420203827.png)

- 有旋转

![img](https://gitee.com/HNov/image/raw/master/typora/20200417004013.png)

![img](https://gitee.com/HNov/image/raw/master/typora/20200417004024.png)

### TreeNode类的moveRootToFront方法

 **概述**

TreeNode在增加或删除节点后，都需要对整个树重新进行平衡，平衡之后的根节点也许就会发生变化，此时为了保证：如果HashMap元素数组根据下标取得的元素是一个TreeNode类型，那么这个TreeNode节点一定要是这颗树的根节点，同时也要是整个链表的首节点。

**方法解析**

```java
/**
 * 把红黑树的根节点设为  其所在的数组槽 的第一个元素
 * 首先明确：TreeNode既是一个红黑树结构，也是一个双链表结构
 * 这个方法里做的事情，就是保证树的根节点一定也要成为链表的首节点
 */
static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {
    int n;
    if (root != null && tab != null && (n = tab.length) > 0) { // 根节点不为空 并且 HashMap的元素数组不为空
        int index = (n - 1) & root.hash; // 根据根节点的Hash值 和 HashMap的元素数组长度  取得根节点在数组中的位置
        TreeNode<K,V> first = (TreeNode<K,V>)tab[index]; // 首先取得该位置上的第一个节点对象
        if (root != first) { // 如果该节点对象 与 根节点对象 不同
            Node<K,V> rn; // 定义根节点的后一个节点
            tab[index] = root; // 把元素数组index位置的元素替换为根节点对象
            TreeNode<K,V> rp = root.prev; // 获取根节点对象的前一个节点
            if ((rn = root.next) != null) // 如果后节点不为空 
                ((TreeNode<K,V>)rn).prev = rp; // root后节点的前节点  指向到 root的前节点，相当于把root从链表中摘除
            if (rp != null) // 如果root的前节点不为空
                rp.next = rn; // root前节点的后节点 指向到 root的后节点
            if (first != null) // 如果数组该位置上原来的元素不为空
                first.prev = root; // 这个原有的元素的 前节点 指向到 root，相当于root目前位于链表的首位
            root.next = first; // 原来的第一个节点现在作为root的下一个节点，变成了第二个节点
            root.prev = null; // 首节点没有前节点
        }
 
        /*
         * 这一步是防御性的编程
         * 校验TreeNode对象是否满足红黑树和双链表的特性
         * 如果这个方法校验不通过：可能是因为用户编程失误，破坏了结构（例如：并发场景下）；也可能是TreeNode的实现有问题（这个是理论上的以防万一）；
         */ 
        assert checkInvariants(root); 
    }
}
```

### hash方法

 **概述**

我们知道在HashMap中，一个键值对存储在HashMap内部数据的哪个位置上和K的hashCode值有关，这也是因为HashMap的hash算法要基于hashCode值来进行。

这里要注意区分三个概念：hashCode值、hash值、hash方法、数组下标

hashCode值：是KV对中的K对象的hashCode方法的返回值（若没有重写则默认用Object类的hashCode方法的生成值）

hash值：是在hashCode值的基础上又进行了一步运算后的结果，这个运算过程就是hash方法。

数组下标：根据该hash值和数组长度计算出数组下标，计算公式：hash值  &（数组长度-1）= 下标。

我们要讨论的就是上面提到的hash方法，首先他是HashMap类中的一个静态方法，该方法的代码特别简单，只有两行，语法很容易懂，可是其中具体用意确值得好好深入研究下。

我们首先讨论下数组下标计算过程，这个有助于我们理解hash方法的设计思想

 **数组下标计算**

```
hash：hash值
length：数组长度
计算公式：hash  &（length-1）

若length=16，hash=1,那么下标值计算过程如下：
    0000 0000 0000 0000 0000 0000 0000 1111    16-1 = 15，15对应的二进制值为1111
    0000 0000 0000 0000 0000 0000 0000 0001    1的二进制
    0000 0000 0000 0000 0000 0000 0000 0001    上两行进行与运算，最终结果是1

若length=16，hash=17,那么下标值计算过程如下：
    0000 0000 0000 0000 0000 0000 0000 1111    16-1 = 15，15对应的二进制值为1111
    0000 0000 0000 0000 0000 0000 0001 0001    17的二进制
    0000 0000 0000 0000 0000 0000 0000 0001    上两行进行与运算，最终结果是1

若length=16，hash=33,那么下标值计算过程如下：
    0000 0000 0000 0000 0000 0000 0000 1111    16-1 = 15，15对应的二进制值为1111
    0000 0000 0000 0000 0000 0000 0010 0001    33的二进制
    0000 0000 0000 0000 0000 0000 0000 0001    上两行进行与运算，最终结果是1

若length=32，hash=1,那么下标值计算过程如下：
    0000 0000 0000 0000 0000 0000 0001 1111    32-1 = 31，31对应的二进制值为11111
    0000 0000 0000 0000 0000 0000 0000 0001    1的二进制
    0000 0000 0000 0000 0000 0000 0000 0001    上两行进行与运算，最终结果是1

若length=32，hash=17,那么下标值计算过程如下：
    0000 0000 0000 0000 0000 0000 0001 1111    32-1 = 31，31对应的二进制值为11111
    0000 0000 0000 0000 0000 0000 0001 0001    17的二进制
    0000 0000 0000 0000 0000 0000 0001 0001    上两行进行与运算，最终结果是17

若length=32，hash=33,那么下标值计算过程如下：
    0000 0000 0000 0000 0000 0000 0001 1111    32-1 = 31，31对应的二进制值为11111
    0000 0000 0000 0000 0000 0000 0010 0001    33的二进制
    0000 0000 0000 0000 0000 0000 0000 0001    上两行进行与运算，最终结果是1


(1,16)->1、(17,16)->1、(33,16)->1
(1,32)->1、(17,32)->17、(33,32)->1
总结：
hash  &（length-1）的运算效果等同于 hash%length。
我们知道hashMap的扩容规则是2倍扩容、数组长度一定是2的N次方。假设数组长度不是2的N次方，那么再重复以上的计算过程，就达不到求模的效果了。
```



讨论完hashMap数组下标寻址过程后，我们可以得知

> 在put(K,V)的时候，会根据K的hash值和当前数组长度求模，得到该键值对应该存储在数组的哪个位置。

> 在get(K)的时候，同样会根据K的hash值和当前数组长度求模，定位到应该去数组的哪个位置寻找V。

按照上面举得例子，put(K,V)，多个hash值和数组长度求模之后的结果相同时，大家都存储在数据的同一位置，这种情况也叫hash碰撞，发生了这种碰撞时，大家会以先来后到的方式以链表的方式组织起来，那么也就意味着我在get(K)的时候，虽然能够定位到数组位置，还需要遍历链表，通过equals逐个对比之后才能确定需要返回的V。

假设hash值都不相同，那么hashMap数组的长度越大，碰撞的机率越小。

在数组长度为16时，当hash值为1、17、33时，大家都定位到下标1，那么此时我们也可以这么想一下，当我的hashMap数组的长度不变或者不会再变得时候，大于数组长度的hash值（17、33）对我hashMap来讲和hash值1的效果是等同的。

在数组长度为65535时，当hash值为1、131071、262143时，大家都定位到下标1，那么此时我们也可以这么想一下，当我的hashMap数组的长度不变或者不会再变得时候，大于数组长度的hash值（131071、262143）对我hashMap来讲和hash值1的效果是等同的。

以上两种场景，如果假设：hash值 = hashCode值，就意味着无论是我们自己定义的hashCode还是默认的hashCode生成方式，大于数组长度的值并没有产生我们想要达到的理想效果（我们希望，不同的hashCode能够让他分布到一个不会碰撞的位置），因为取模之后，他的位置可能至少会和某个小于数组长度的值碰撞。

什么场景下可以避免碰撞：hashMap数组最大长度可知（要存储的数据不超过数组最大长度），创建时就指定初始容量为最大长度，每个K的hash值各不同，且每个hash值都小于数组最大长度。这种场景有么？有，但是我们大部分的应用场景都不会如此巧合或如此故意而为之。

再看下我们可能最常用的场景：

```java
Map<String,Object> user = new HashMap<String,Object>(); // 默认内部数据长度为16
user.put("name", "kevin");
user.put("sex", 1);
user.put("level", 1);
user.put("phone", "13000000000");
user.put("address", "beijing fengtai");
```

以字符串作为key应该是比较常用的情况了，那么这些K对应的hashCode是什么呢？如果根据hashCode来定位下标的话又是什么？

```java
System.out.println("\"name\".hashCode()	: " +"name".hashCode());
System.out.println("\"sex\".hashCode()	: " +"sex".hashCode());
System.out.println("\"level\".hashCode()	: " +"level".hashCode());
System.out.println("\"phone\".hashCode()	: " +"phone".hashCode());
System.out.println("\"address\".hashCode()	: " +"address".hashCode());
 
System.out.println("--------*****---------");
 
System.out.println("\"name\".hashCode() & (16 - 1) 	:" + ("name".hashCode() & (16 - 1)));
System.out.println("\"sex\".hashCode() & (16 - 1) 	:" + ("sex".hashCode() & (16 - 1)));
System.out.println("\"level\".hashCode() & (16 - 1)	:" + ("level".hashCode() & (16 - 1)));
System.out.println("\"phone\".hashCode() & (16 - 1) 	:" + ("phone".hashCode() & (16 - 1)));
System.out.println("\"address\".hashCode() & (16 - 1) :" + ("address".hashCode() & (16 - 1)));
 
输出结果：
 
"name".hashCode()	: 3373707
"sex".hashCode()	: 113766
"level".hashCode()	: 102865796
"phone".hashCode()	: 106642798
"address".hashCode()	: -1147692044
--------*****---------
"name".hashCode() & (16 - 1) 	:11
"sex".hashCode() & (16 - 1) 	:6
"level".hashCode() & (16 - 1)	:4
"phone".hashCode() & (16 - 1) 	:14
"address".hashCode() & (16 - 1) :4
```

如上输出结果，我们向hashMap里put了5个键值对，每个K的hashCode值均不相同，但是根据hashCode计算出来的下标却存在碰撞（有两个4），虽然hashCode的值很随机，但是限于我们数组长度，即便是很少的几个数据，也是有很高机率碰撞的，这也就说明了这些看起来（绝对值）很大hashCode值和【0-15】范围内的值对于长度为16的初始数组容量来讲效果等同。

但是在数据量很少的情况下，即便发生了碰撞，性能上的劣势也几乎可以忽略不计。

但是我们上面的下标求值是一种假设，假设直接根据K的hashCode和数组长度求模，但是实际上是根据K的hash值和数组长度求模

那么K的hash值是什么，如何计算出来的呢，接下来就来讨论hash值的计算过程，hash方法。

**hash方法解析**

```java
static final int hash(Object key) {
     int h;
     return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

1、如果key为空，那么hash值置为0。HashMap允许null作为键，虽然这样，因为null的hash值一定是0，而且null==null为真，所以HashMap里面最多只会有一个null键。而且这个null键一定是放在数组的第一个位置上。但是如果存在hash碰撞，该位置上形成链表了，那么null键对应的节点就不确定在链表中的哪个位置了（取决于插入顺序，并且每次扩容其在链表中的位置都可能会改变）。

2、如果key是个不为空的对象，那么将key的hashCode值h和h无符号右移16位后的值做异或运算，得到最终的hash值。

从代码中目前我们可确定的信息是：hashCode值（h）是计算基础，在h的基础上进行了两次位运算（无符号右移、异或）

我们还针对前面的user的key来重新进行一下测试

```java
System.out.println("hash(\"name\")	: " + hash("name"));
System.out.println("hash(\"sex\")	: " + hash("sex"));
System.out.println("hash(\"level\")	: " + hash("level"));
System.out.println("hash(\"phone\")	: " + hash("phone"));
System.out.println("hash(\"address\")	: " + hash("address"));
 
System.out.println("--------*****---------");
 
System.out.println("hash(\"name\") & (16 - 1) 	:" + (hash("name") & (16 - 1)));
System.out.println("hash(\"sex\") & (16 - 1) 		:" + (hash("sex") & (16 - 1)));
System.out.println("hash(\"level\") & (16 - 1)	:" + (hash("level") & (16 - 1)));
System.out.println("hash(\"phone\") & (16 - 1) 	:" + (hash("phone") & (16 - 1)));
System.out.println("hash(\"address\") & (16 - 1) 	:" + (hash("address") & (16 - 1)));
 
输出结果：
 
hash("name")	: 3373752
hash("sex")	: 113767
hash("level")	: 102866341
hash("phone")	: 106642229
hash("address")	: -1147723677
--------*****---------
hash("name") & (16 - 1) 	:8
hash("sex") & (16 - 1) 		:7
hash("level") & (16 - 1)	:5
hash("phone") & (16 - 1) 	:5
hash("address") & (16 - 1) 	:3
```

我们对比一下两次的输出结果

```java
"name".hashCode()	    : 3373707               hash("name")	: 3373752
"sex".hashCode()	    : 113766                hash("sex")	        : 113767
"level".hashCode()	    : 102865796             hash("level")	: 102866341
"phone".hashCode()	    : 106642798             hash("phone")	: 106642229
"address".hashCode()	    : -1147692044           hash("address")	: -1147723677
--------*****---------
"name".hashCode() & (16 - 1) 	:11                 hash("name") & (16 - 1) 	:8
"sex".hashCode() & (16 - 1) 	:6                  hash("sex") & (16 - 1) 	:7
"level".hashCode() & (16 - 1)	:4                  hash("level") & (16 - 1)	:5
"phone".hashCode() & (16 - 1) 	:14                 hash("phone") & (16 - 1) 	:5
"address".hashCode() & (16 - 1) :4                  hash("address") & (16 - 1) 	:3
```

虽然经过hash算法之后与直接使用hashCode的输出不同，但是数组下标还是出现了碰撞的情况（有两个5）。所以hash方法也不能解决碰撞的问题（实际上碰撞不算是问题，我们只是想尽可能的少发生）。那么为什么不直接用hashCode而非要经过这么一种位运算来产生一个hash值呢。

我们先通过几个例子来看下 h ^ (h >>> 16)  的计算过程

```java
若h=17，那么h ^ (h >>> 16)的计算可体现为如下过程：
    0000 0000 0000 0000 0000 0000 0001 0001    此时就是：h（17）的二进制。【高位是16个0】
    0000 0000 0000 0000 0000 0000 0000 0000    h（17）的二进制无符号右移16位后，此时就是：（h >>> 16）的二进制
    0000 0000 0000 0000 0000 0000 0001 0001    上两行进行异或运算，此时就是：h ^ (h >>> 16)的二进制
最终可知（当h=17时）：h ^ (h >>> 16) = 17，并没有发生变化。

若h=65535，那么h ^ (h >>> 16)的计算可体现为如下过程：
    0000 0000 0000 0000 1111 1111 1111 1111    h【高位还是16个0】
    0000 0000 0000 0000 0000 0000 0000 0000    h >>> 16
    0000 0000 0000 0000 1111 1111 1111 1111    h ^ (h >>> 16)
最终可知（当h=65535时）：h ^ (h >>> 16) = 65535，并没有发生变化。

若h=65536，那么h ^ (h >>> 16)的计算可体现为如下过程：
    0000 0000 0000 0001 0000 0000 0000 0000     h【高位含有一个1】
    0000 0000 0000 0000 0000 0000 0000 0001     h >>> 16
    0000 0000 0000 0001 0000 0000 0000 0001     h ^ (h >>> 16)
最终可知（当h=65536时）：h ^ (h >>> 16) = 65537，hash后的值和原值不同。

若h=1147904，那么h ^ (h >>> 16)的计算可体现为如下过程：
    0000 0000 0001 0001 1000 0100 0000 0000     h【高位含有两个1，并不连续】
    0000 0000 0000 0000 0000 0000 0001 0001     h >>> 16
    0000 0000 0001 0001 1000 0100 0001 0001     h ^ (h >>> 16)
最终可知（当h=1147904时）：h ^ (h >>> 16) = 1147921，hash后的值和原值不同。

再来看个负数的情况，负数的情况稍微复杂些，主要因为负数的二进制和十进制之间的转换会有【加|减|取反】等操作。
若h=-5，那么h ^ (h >>> 16)的计算可体现为如下过程：
    0000 0000 0000 0000 0000 0000 0000 0101    先得到5的二进制
    1111 1111 1111 1111 1111 1111 1111 1010    5的二进制取反

    1111 1111 1111 1111 1111 1111 1111 1011    5的二进制取反后加1，此时就是：h（-5）的二进制。
    0000 0000 0000 0000 1111 1111 1111 1111    h（-5）的二进制无符号右移16位后，此时就是：（h >>> 16）的二进制
    1111 1111 1111 1111 0000 0000 0000 0100    上两行进行异或运算，此时就是：h ^ (h >>> 16)的二进制
                                               接下来求一下这个二进制对应的10进制数值
    1111 1111 1111 1111 0000 0000 0000 0011    上一个二进制值减1
    0000 0000 0000 0000 1111 1111 1111 1100    再取反，此时十进制值为65532，但是需要加个负号
最终可知（当h=-5时）：h ^ (h >>> 16) = -65532，hash后的值和原值相差比较悬殊

```

1）上面的例子只考虑了正数的情况，但可以得出以下结论
当h（hashCode值）在【0-65535】时，位运算后的结果仍然是h
当h（hashCode值）在【65535-N】时，位运算后的结果和h不同

当h（hashCode值）为负数时，位运算后的结果和h也不尽相同

2）我们上面user对象里的key的hashCode值都没有在【0-65535】范围内，所以计算出来的结果与hashCode值存在差异。

 

为什么【0-65535】范围内的数值h，h ^ (h >>> 16) = h ？

从例子中的二进制的运算描述我们可以发现，【0-65535】范围内的数值的二进制的高16位都是0，在进行无符号右移16位后，原来的高16位变成了现在的低16位，现在的高16位补了16个0，这种操作之后当前值就是32个0，以32个0去和任何整数值N做异或运算结果都还是N。而不再【0-65535】范围内的数值的高16位都包含有1数字位，在无符号右移16位后，虽然高位仍然补了16个0，但是当前的低位任然包含有1数字位，所以最终的运算结果会发生变化。

而我们use对象里的key的hashCode要么大于65535、要么小于0所以最终的hash值和hashCode都不相同，因为在异或运算中hashCode的高位中的非0数字位参与到了运算中。

 

四、hash方法的作用

前文通过实例已经印证了hash方法并不能杜绝碰撞。

```java
"name".hashCode()	    : 3373707               hash("name")	: 3373752
"sex".hashCode()	    : 113766                hash("sex")	        : 113767
"level".hashCode()	    : 102865796             hash("level")	: 102866341
"phone".hashCode()	    : 106642798             hash("phone")	: 106642229
```

而且通过对比观察，hash后的值和hashCode值虽然不尽相同，而对于正数的hashCode的产生的hash值即便和原值不同，差别也不是很大。为什么不直接使用hashCode作为hash值呢？为什么非要经过 h ^ (h >>> 16)  这一步运算呢？

在hash方法的注释是这样描述的：

```java
/**
* Computes key.hashCode() and spreads (XORs) higher bits of hash
* to lower.  Because the table uses power-of-two masking, sets of
* hashes that vary only in bits above the current mask will
* always collide. (Among known examples are sets of Float keys
* holding consecutive whole numbers in small tables.)  So we
* apply a transform that spreads the impact of higher bits
* downward. There is a tradeoff between speed, utility, and
* quality of bit-spreading. Because many common sets of hashes
* are already reasonably distributed (so don't benefit from
* spreading), and because we use trees to handle large sets of
* collisions in bins, we just XOR some shifted bits in the
* cheapest possible way to reduce systematic lossage, as well as
* to incorporate impact of the highest bits that would otherwise
* never be used in index calculations because of table bounds.
*/
```

大概意思就是：

寻址计算时，能够参与到计算的有效二进制位仅仅是右侧和数组长度值对应的那几位，意味着发生碰撞的几率会高。

通过移位和异或运算让hashCode的高位能够参与到寻址计算中。

采用这种方式是为了在性能、实用性、分布质量上取得一个平衡。

有很多hashCode算法都已经是分布合理的了，并且大量碰撞时，还可以通过树结构来解决查询性能问题。

所以用了性能比较高的位运算来让高位参与到寻址运算中，位运算对于系统损耗相对低廉。

还提到了Float keys，个人认为说的应该是浮点数类型的对象作为key的时候，也顺便测试了下

```java
System.out.println("Float.valueOf(0.1f).hashCode()		:" + Float.valueOf(0.1f).hashCode());
System.out.println("Float.valueOf(1.3f).hashCode()		:" + Float.valueOf(1.3f).hashCode());
System.out.println("Float.valueOf(100.4f).hashCode()	:" + Float.valueOf(1.4f).hashCode());
System.out.println("Float.valueOf(987607.3f).hashCode()	:" + Float.valueOf(100000.3f).hashCode());
System.out.println("Float.valueOf(2809764.4f).hashCode()	:" + Float.valueOf(100000.4f).hashCode());
System.out.println("Float.valueOf(-100.3f).hashCode()	:" + Float.valueOf(-100.3f).hashCode());
System.out.println("Float.valueOf(-100.4f).hashCode()	:" + Float.valueOf(-100.4f).hashCode());
 
System.out.println("--------*****---------");
 
System.out.println("hash(0.1f)		:" + hash(0.1f));
System.out.println("hash(1.3f)		:" + hash(1.3f));
System.out.println("hash(1.4f)	:" + hash(1.4f));
System.out.println("hash(100000.3f)	:" + hash(100000.3f));
System.out.println("hash(100000.4f)	:" + hash(100000.4f));
System.out.println("hash(-100.3f)	:" + hash(-100.3f));
System.out.println("hash(-100.4f)	:" + hash(-100.4f));
 
输出结果：
 
Float.valueOf(0.1f).hashCode()		:1036831949
Float.valueOf(1.3f).hashCode()		:1067869798
Float.valueOf(100.4f).hashCode()	:1068708659
Float.valueOf(987607.3f).hashCode()	:1203982374
Float.valueOf(2809764.4f).hashCode()	:1203982387
Float.valueOf(-100.3f).hashCode()	:-1027040870
Float.valueOf(-100.4f).hashCode()	:-1027027763
--------*****---------
hash(0.1f)	:1036841217
hash(1.3f)	:1067866560
hash(1.4f)	:1068698752
hash(100000.3f)	:1203967973
hash(100000.4f)	:1203967984
hash(-100.3f)	:-1027056814
hash(-100.4f)	:-1027076603
```

示例中的浮点数数值大小跨度还是比较大的，但是生成hashCode相差都不大，因为hashCode相差不大，所以最终的hash值相差也不大。那么要怎么证明hash之后就会减少碰撞呢？这个不太理解，看来还得看看大神们的分析。

### comparableClassFor、compareComparables、tieBreakOrder方法

**概述**

在之前的文章里已经分析过，在发生hash碰撞（多个key的hash值相同）的时候，hashMap首先会采用链表进行存储，当链表节点数量达到一定阈值（8）会将链表上的节点再组织成一棵红黑树。红黑树是一种二叉树，每个父节点可以由左右两个节点。

当put一个新元素时，如果该元素键的hash值小于当前节点的hash值的时候，就会作为当前节点的左节点；hash值大于当前节点hash值得时候作为当前节点的右节点。那么hash值相同的时候呢？这时还是会先尝试看是否能够通过Comparable进行比较一下两个对象（当前节点的键对象和新元素的键对象），要想看看是否能基于Comparable进行比较的话，首先要看该元素键是否实现了Comparable接口，此时就需要用到comparableClassFor方法来获取该元素键的Class，然后再通过compareComparables方法来比较两个对象的大小。

**方法解析**

```java
/**
* 如果对象x的类是C，如果C实现了Comparable<C>接口，那么返回C，否则返回null
*/
static Class<?> comparableClassFor(Object x) {
    if (x instanceof Comparable) {
        Class<?> c; Type[] ts, as; Type t; ParameterizedType p;
        if ((c = x.getClass()) == String.class) // 如果x是个字符串对象
            return c; // 返回String.class
        /*
         * 为什么如果x是个字符串就直接返回c了呢 ? 因为String  实现了 Comparable 接口，可参考如下String类的定义
         * public final class String implements java.io.Serializable, Comparable<String>, CharSequence
         */ 
 
        // 如果 c 不是字符串类，获取c直接实现的接口（如果是泛型接口则附带泛型信息）    
        if ((ts = c.getGenericInterfaces()) != null) {
            for (int i = 0; i < ts.length; ++i) { // 遍历接口数组
                // 如果当前接口t是个泛型接口 
                // 如果该泛型接口t的原始类型p 是 Comparable 接口
                // 如果该Comparable接口p只定义了一个泛型参数
                // 如果这一个泛型参数的类型就是c，那么返回c
                if (((t = ts[i]) instanceof ParameterizedType) &&
                    ((p = (ParameterizedType)t).getRawType() ==
                        Comparable.class) &&
                    (as = p.getActualTypeArguments()) != null &&
                    as.length == 1 && as[0] == c) // type arg is c
                    return c;
            }
            // 上面for循环的目的就是为了看看x的class是否 implements  Comparable<x的class>
        }
    }
    return null; // 如果c并没有实现 Comparable<c> 那么返回空
}
```

```java
/**
* 如果x所属的类是kc，返回k.compareTo(x)的比较结果
* 如果x为空，或者其所属的类不是kc，返回0
*/
@SuppressWarnings({"rawtypes","unchecked"}) // for cast to Comparable
static int compareComparables(Class<?> kc, Object k, Object x) {
    return (x == null || x.getClass() != kc ? 0 :
            ((Comparable)k).compareTo(x));
}
```

如果两者不具有compare的资格，或者compare之后仍然没有比较出大小。那么就要通过一个**决胜局**再比一次，这个决胜局就是tieBreakOrder方法。

```java
/**
* 用这个方法来比较两个对象，返回值要么大于0，要么小于0，不会为0
* 也就是说这一步一定能确定要插入的节点要么是树的左节点，要么是右节点，不然就无法继续满足二叉树结构了
* 
* 先比较两个对象的类名，类名是字符串对象，就按字符串的比较规则
* 如果两个对象是同一个类型，那么调用本地方法为两个对象生成hashCode值，再进行比较，hashCode相等的话返回-1
*/
static int tieBreakOrder(Object a, Object b) {
    int d;
    if (a == null || b == null ||
        (d = a.getClass().getName().
            compareTo(b.getClass().getName())) == 0)
        d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
                -1 : 1);
    return d;
}
```

### TreeNode类的find方法

**概述**

当我们向HashMap里put一个键值对的时候，需要检测键是否已经存在，如果存在需要替换值，如果不存在需要添加。当要添加的键产生了hash碰撞，并且碰撞位置上已经是一个树结构时，那么就需要检查该键是否存在于该树上，此时调用find来查找该键对应的节点对象。如果找到了就替换该节点的值，如果未找到就会向树上添加一个新的节点。

**方法解析**

```java
/**
* 这个方法是TreeNode类的一个实例方法，调用该方法的也就是一个TreeNode对象，
* 该对象就是树上的某个节点，以该节点作为根节点，查找其所有子孙节点，
* 看看哪个节点能够匹配上给定的键对象
* h k的hash值
* k 要查找的对象
* kc k的Class对象，该Class应该是实现了Comparable<K>的，否则应该是null，参见：
*/
final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
    TreeNode<K,V> p = this; // 把当前对象赋给p，表示当前节点
    do { // 循环
        int ph, dir; K pk; // 定义当前节点的hash值、方向（左右）、当前节点的键对象
        TreeNode<K,V> pl = p.left, pr = p.right, q; // 获取当前节点的左孩子、右孩子。定义一个对象q用来存储并返回找到的对象
        if ((ph = p.hash) > h) // 如果当前节点的hash值大于k得hash值h，那么后续就应该让k和左孩子节点进行下一轮比较
            p = pl; // p指向左孩子，紧接着就是下一轮循环了
        else if (ph < h) // 如果当前节点的hash值小于k得hash值h，那么后续就应该让k和右孩子节点进行下一轮比较
            p = pr; // p指向右孩子，紧接着就是下一轮循环了
        else if ((pk = p.key) == k || (k != null && k.equals(pk))) // 如果h和当前节点的hash值相同，并且当前节点的键对象pk和k相等（地址相同或者equals相同）
            return p; // 返回当前节点
 
 
        // 执行到这里说明 hash比对相同，但是pk和k不相等
 
        else if (pl == null) // 如果左孩子为空
            p = pr; // p指向右孩子，紧接着就是下一轮循环了
        else if (pr == null)
            p = pl; // p指向左孩子，紧接着就是下一轮循环了
 
        // 如果左右孩子都不为空，那么需要再进行一轮对比来确定到底该往哪个方向去深入对比
        // 这一轮的对比主要是想通过comparable方法来比较pk和k的大小     
        else if ((kc != null ||
                    (kc = comparableClassFor(k)) != null) &&
                    (dir = compareComparables(kc, k, pk)) != 0)
            p = (dir < 0) ? pl : pr; // dir小于0，p指向右孩子，否则指向右孩子。紧接着就是下一轮循环了
 
        // 执行到这里说明无法通过comparable比较  或者 比较之后还是相等
        // 从右孩子节点递归循环查找，如果找到了匹配的则返回    
        else if ((q = pr.find(h, k, kc)) != null) 
            return q;
        else // 如果从右孩子节点递归查找后仍未找到，那么从左孩子节点进行下一轮循环
            p = pl;
    } while (p != null); 
    return null; // 为找到匹配的节点返回null
}
```

### get方法、containsKey方法、getNode方法

**概述**

HashMap存储的键值对，用put(K,V)方法来存储，用get(K)方法来获取V，用containsKey(K)方法来检查K是否存在。可先参见：[put方法解析](https://blog.csdn.net/weixin_42340670/article/details/80503369) 来了解键值对的存储原理，再来看get方法就很容易了。

**方法解析**

```java
/**
* 获取key对应的值，如果找不到则返回null
* 但是如果返回null并不意味着就没有找到，也可能key对应的值就是null，因为HashMap允许null值（也允许null键）
* 在返回值为null时，可以通过containsKey来方法来区分到底是因为key不存在，还是key对应的值就位null
*/
public V get(Object key) {
    Node<K,V> e; // 声明一个节点对象（键值对对象）
    // 调用getNode方法来获取键值对，如果没有找到返回null，找到了就返回键值对的值
    return (e = getNode(hash(key), key)) == null ? null : e.value; //真正的查找过程都是通过getNode方法实现的
}
 
/**
* 检查是否包含key
* 如果key有对应的节点对象，则返回ture，不关心节点对象的值是否为空
*/
public boolean containsKey(Object key) {
    // 调用getNode方法来获取键值对，如果没有找到返回false，找到了就返回ture
    return getNode(hash(key), key) != null; //真正的查找过程都是通过getNode方法实现的
}
 
/**
* 该方法是Map.get方法的具体实现
* 接收两个参数
* @param hash key的hash值，根据hash值在节点数组中寻址，该hash值是通过hash(key)得到的，可参见：hash方法解析
* @param key key对象，当存在hash碰撞时，要逐个比对是否相等
* @return 查找到则返回键值对节点对象，否则返回null
*/
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k; // 声明节点数组对象、链表的第一个节点对象、循环遍历时的当前节点对象、数组长度、节点的键对象
    // 节点数组赋值、数组长度赋值、通过位运算得到求模结果确定链表的首节点
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // 首先比对首节点，如果首节点的hash值和key的hash值相同 并且 首节点的键对象和key相同（地址相同或equals相等），则返回该节点
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first; // 返回首节点
 
        // 如果首节点比对不相同、那么看看是否存在下一个节点，如果存在的话，可以继续比对，如果不存在就意味着key没有匹配的键值对    
        if ((e = first.next) != null) {
            // 如果存在下一个节点 e，那么先看看这个首节点是否是个树节点
            if (first instanceof TreeNode)
                // 如果是首节点是树节点，那么遍历树来查找
                return ((TreeNode<K,V>)first).getTreeNode(hash, key); 
 
            // 如果首节点不是树节点，就说明还是个普通的链表，那么逐个遍历比对即可    
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k)))) // 比对时还是先看hash值是否相同、再看地址或equals
                    return e; // 如果当前节点e的键对象和key相同，那么返回e
            } while ((e = e.next) != null); // 看看是否还有下一个节点，如果有，继续下一轮比对，否则跳出循环
        }
    }
    return null; // 在比对完了应该比对的树节点 或者全部的链表节点 都没能匹配到key，那么就返回null
}
```

### remove方法、removeNode方法

**概述**

在HashMap中如果要根据key删除这个key对应的键值对，需要调用remove(key)方法，该方法将会根据查找到匹配的键值对，将其从HashMap中删除，并且返回键值对的值。

**方法解析**

我们先来看remove方法

```java
/**
* 从HashMap中删除掉指定key对应的键值对，并返回被删除的键值对的值
* 如果返回空，说明key可能不存在，也可能key对应的值就是null
* 如果想确定到底key是否存在可以使用containsKey方法
*/
public V remove(Object key) {
    Node<K,V> e; // 定义一个节点变量，用来存储要被删除的节点（键值对）
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value; // 调用removeNode方法
}
```



可以发现remove方法底层实际上是调用了removeNode方法来删除键值对节点，并且根据返回的节点对象取得key对应的值，那么我们再来详细分析下removeNode方法的代码

```java
/**
* 方法为final，不可被覆写，子类可以通过实现afterNodeRemoval方法来增加自己的处理逻辑（解析中有描述）
*
* @param hash key的hash值，该值是通过hash(key)获取到的
* @param key 要删除的键值对的key
* @param value 要删除的键值对的value，该值是否作为删除的条件取决于matchValue是否为true
* @param matchValue 如果为true，则当key对应的键值对的值equals(value)为true时才删除；否则不关心value的值
* @param movable 删除后是否移动节点，如果为false，则不移动
* @return 返回被删除的节点对象，如果没有删除任何节点则返回null
*/
final Node<K,V> removeNode(int hash, Object key, Object value,
                            boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index; // 声明节点数组、当前节点、数组长度、索引值
    /*
     * 如果 节点数组tab不为空、数组长度n大于0、根据hash定位到的节点对象p（该节点为 树的根节点 或 链表的首节点）不为空
     * 需要从该节点p向下遍历，找到那个和key匹配的节点对象
     */
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v; // 定义要返回的节点对象，声明一个临时节点变量、键变量、值变量
 
        // 如果当前节点的键和key相等，那么当前节点就是要删除的节点，赋值给node
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
 
        /*
         * 到这一步说明首节点没有匹配上，那么检查下是否有next节点
         * 如果没有next节点，就说明该节点所在位置上没有发生hash碰撞, 就一个节点并且还没匹配上，也就没得删了，最终也就返回null了
         * 如果存在next节点，就说明该数组位置上发生了hash碰撞，此时可能存在一个链表，也可能是一颗红黑树
         */
        else if ((e = p.next) != null) {
            // 如果当前节点是TreeNode类型，说明已经是一个红黑树，那么调用getTreeNode方法从树结构中查找满足条件的节点
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            // 如果不是树节点，那么就是一个链表，只需要从头到尾逐个节点比对即可    
            else {
                do {
                    // 如果e节点的键是否和key相等，e节点就是要删除的节点，赋值给node变量，调出循环
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                            (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
 
                    // 走到这里，说明e也没有匹配上
                    p = e; // 把当前节点p指向e，这一步是让p存储的永远下一次循环里e的父节点，如果下一次e匹配上了，那么p就是node的父节点
                } while ((e = e.next) != null); // 如果e存在下一个节点，那么继续去匹配下一个节点。直到匹配到某个节点跳出 或者 遍历完链表所有节点
            }
        }
 
        /*
         * 如果node不为空，说明根据key匹配到了要删除的节点
         * 如果不需要对比value值  或者  需要对比value值但是value值也相等
         * 那么就可以删除该node节点了
         */
        if (node != null && (!matchValue || (v = node.value) == value ||
                                (value != null && value.equals(v)))) {
            if (node instanceof TreeNode) // 如果该节点是个TreeNode对象，说明此节点存在于红黑树结构中，调用removeTreeNode方法（该方法单独解析）移除该节点
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p) // 如果该节点不是TreeNode对象，node == p 的意思是该node节点就是首节点
                tab[index] = node.next; // 由于删除的是首节点，那么直接将节点数组对应位置指向到第二个节点即可
            else // 如果node节点不是首节点，此时p是node的父节点，由于要删除node，所有只需要把p的下一个节点指向到node的下一个节点即可把node从链表中删除了
                p.next = node.next;
            ++modCount; // HashMap的修改次数递增
            --size; // HashMap的元素个数递减
            afterNodeRemoval(node); // 调用afterNodeRemoval方法，该方法HashMap没有任何实现逻辑，目的是为了让子类根据需要自行覆写
            return node;
        }
    }
    return null;
}
```

## 静态方法

- hash(Object key)
- comparableClassFor(Object x)
- compareComparables(Class<?> kc, Object k, Object x)
- tableSizeFor(int cap)

## 其他方法

-  putAll(Map<? extends K, ? extends V> m)
- remove(Object key) 
- removeNode(int hash, Object key, Object value,boolean matchValue, boolean movable)
- keySet()
- entrySet()
- getOrDefault(Object key, V defaultValue)
- replaceAll(BiFunction<? super K, ? super V, ? extends V> function)
- writeObject(java.io.ObjectOutputStream s)
- readObject(java.io.ObjectInputStream s)
- size()
- clear() 
- isEmpty()

# 三、JDK1.8与JDK1.7的区别

1. resize 扩容优化

   不需要像JDK1.7的实现那样重新计算hash，原来的元素，要么在原位置(低位)，要么在`原位置+原数组长度` 那个位置上（高位）。JDK1.7中rehash的时候，旧链表迁移新链表的时候，如果在新表的数组索引位置相同，则链表元素会倒置，但是从上图可以看出，JDK1.8不会倒置。参考 <a href="#resize过程图解">resize过程图解</a>

2. 引入了红黑树，目的是避免单条链表过长而影响查询效率

3. 解决了resize时多线程死循环问题，但仍是非线程安全的

# 四、灵魂拷问

##  如何扩容

默认数组大小为16,扩容每次都是两倍,扩容因子是0.75,即当数组容量达到12的时候就扩容,扩容的时候会创建一个新的数组,然后将原来数组的entry们复制到新数组中,原数组需要null,这样才会被gc回收.
`注意`之前的entry的位置只有两种情况,一种是原来位置,一种是原来位置+原容量大小.这个位置的判断是hash值&初始容量即16,如果高位为0则原来位置,如果高位为1则位置变为原来位置+扩容大小.

##  如何查找

首先根据key的hashcode()找到数组的下标,然后根据key的equals()找到数组链表上的对应的entry.

`注意`这里首先获取hash值有个函数是将高16位和低16位做异或运算,得到hash值.根据hash值&(容量-1)找到数组下标,如果碰撞了就使用equals.

## 为什么高低位要异或?hash算法为什么要高16位不变，低16位与高16位异或作为key的最终hash值？

因为数组的大小是2的n次方并且初始大小为16，假设目前数组大小为16，如果不进行这样运算的话，直接进行hash值与16-1=15的二进制与运算的话，其实只有hash值的后四位参与了运算，这样发生碰撞的概率会提高，而且高16位只有在数组很大的时候才能参与运算，所以用高16位和低16位进行运算，让高位也参与运算，在数组很小的时候增加运算可能性，减少碰撞。


## 为什么数组容量要2的幂次方?

数组的下标是用hash值&（数组大小-1）计算的，

1. 数组大小-1是为了让得到的下标结果控制在数组大小范围内,不越界.

2. 2的n次方的转换为二进制最高位是1，后面都是0，例如2的4次方是16，二进制表示为10000，16-1=15（二进制表示1111），如果不是2的n次方，比如数组大小为20。20-1=19（10011），这样于hashcode与运算，第2.3位为0，这样得到的数组下标只由第1.4.5位决定，大大减小了数组下标的利用率。

## 为什么负载因子是0.75？

如果负载因子是0.5的话：有一半的数组空间会被浪费，随着数组的增大，浪费的越多。

如果负载因子是1的话：数组扩充是为了减少hash碰撞的次数，如果是1，需要每个数组下标都有值才会扩充，如果有一个数组下标迟迟没有值，很有可能其他下标的链表已经非常长了，已经经历过很多次的碰撞，也经历过很多次链表查找并比较的操作，大大影响性能

## 为什么使用红黑树不使用平衡二叉树？

平衡二叉树追求绝对平衡，条件比较苛刻，左右两个子树的高度差的绝对值不超过1，实现起来比较麻烦，每次插入新节点之后需要旋转的次数不能预知。
红黑树放弃了追求完全平衡，追求大致平衡，保证每次只需要三次旋转就能达到平衡，实现起来简单。

## 数组扩充后，原来的节点放在新数组的哪里？

数组扩充后大小是原来的两倍。扩充后会对每个节点进行重hash，从下图源码可以看到这个值只有可能存在两个地方，一个是原下标位置，另一个是原下标+原数组大小的位置

## [HashMap为什么线程不安全(hash碰撞与扩容导致)](https://www.cnblogs.com/qiumingcheng/p/5259892.html)

参考:https://www.cnblogs.com/qiumingcheng/p/5259892.html

## put过程

1.判断键值对数组 table 是否为空或为 null,否则执行 resize()进行扩容;
2.根据键值 key 计算 hash 值得到插入的数组索引 i,如果 table[i]==null,直接新建节点添加,转向6,如果 table[i]不为空,转向3;
3.判断 table[i]的首个元素是否和 key 一样,如果相同直接覆盖 value,否则转向4,这里的相同指的是 hashCode 以及 equals;
4.判断 table[i] 是否为 treeNode,即 table[i] 是否是红黑树,如果是红黑树,则直接在树中插入键值对,否则转向5;
5.遍历 table[i],判断链表长度是否大于 8,大于 8 的话把链表转换为红黑树,在红黑树中执行插入操作,否则进行链表的插入操作;遍历过程中若发现 key 已经存在直接覆盖 value 即可;
6.插入成功后,判断实际存在的键值对数量 size 是否超多了最大容量 threshold,如果超过,进行扩容。

## hashMap和hashTable区别?

1. hashMap是非线程安全的,hashtable是线程安全的,其实就是在操作方法即get()和put()上加上synchronized.
2. hashMap可以添加key和value为null的值,而hashTable不行.

## 如何将hashMap变为线程安全呢?

可以使用Collections.synchronizedMap(map)方法.

## 面试题：初始构造器设置大小为25，hashmap实际大小是多少？

`注意:`hashmap赋值初始值容量的时候,一定是大于等于赋值数的2^n的值.
实际是64，首先，找到比25大的2^n，是32，负载因子为0.75，则能装24个，25>24，触发扩容，为64.

## 面试中关于HashMap的时间复杂度O(1)的思考

只有理想状态下是O(1)

对HashMap的查询分四步:

1.判断key，根据key算出索引。
2.根据索引获得索引位置所对应的键值对链表。
3.遍历键值对链表，根据key找到对应的Entry键值对。
4.拿到value

分析：
要保证HashMap的时间复杂度O(1)，需要保证以上每一步都是O(1)，现在看起来就第三步对链表的循环的时间复杂度影响最大，链表查找的时间复杂度为O(n)，与链表长度有关。我们要保证那个链表长度为1，才可以说时间复杂度能满足O(1)。但这么说来只有那个hash算法尽量减少冲突，才能使链表长度尽可能短，理想状态为1。因此可以得出结论：HashMap的查找时间复杂度只有在最理想的情况下才会为O(1)，而要保证这个理想状态不是我们开发者控制的。

参考文章:
[https://blog.csdn.net/donggua3694857/article/details/64127131/](https://yq.aliyun.com/go/articleRenderRedirect?url=https%3A%2F%2Fblog.csdn.net%2Fdonggua3694857%2Farticle%2Fdetails%2F64127131%2F)

## 为什么HashMap链表会形成死循环

准确的讲应该是 JDK1.7 的 HashMap 链表会有死循环的可能，因为JDK1.7是采用的头插法，在多线程环境下有可能会使链表形成环状，从而导致死循环。JDK1.8做了改进，用的是尾插法，不会产生死循环。

**那么，链表是怎么形成环状的呢？**

关于这一点的解释，我发现网上文章抄来抄去的，而且都来自左耳朵耗子，更惊奇的是，连配图都是一模一样的。（别问我为什么知道，因为我也看过耗子叔的文章，哈哈。然而，菜鸡的我，那篇文章，并没有看懂。。。）

我实在看不下去了，于是一怒之下，就有了这篇文章。我会照着源码一步一步的分析变量之间的关系怎么变化的，并有配图哦。

我们从 put()方法开始，最终找到线程不安全的那个方法。这里省略中间不重要的过程，我只把方法的跳转流程贴出来：

```
//添加元素方法 -> 添加新节点方法 -> 扩容方法 -> 把原数组元素重新分配到新数组中
put()  --> addEntry()  --> resize() -->  transfer()
```

问题就发生在 transfer 这个方法中。

![HashMap(JDK8)知识汇总](https://gitee.com/HNov/image/raw/master/typora/20200417004105.jpeg)



我们假设，原数组容量只有2，其中一条链表上有两个元素 A,B，如下图

![HashMap(JDK8)知识汇总](https://gitee.com/HNov/image/raw/master/typora/20200417004113.jpeg)



现在，有两个线程都执行 transfer 方法。每个线程都会在它们自己的工作内存生成一个newTable 的数组，用于存储变化后的链表，它们互不影响（这里互不影响，指的是两个新数组本身互不影响）。但是，需要注意的是，它们操作的数据却是同一份。

因为，真正的数组中的内容在堆中存储，它们指向的是同一份数据内容。就相当于，有两个不同的引用 X，Y，但是它们都指向同一个对象 Z。这里 X、Y就是两个线程不同的新数组，Z就是堆中的A，B 等元素对象。

假设线程一执行到了上图1中所指的代码①处，恰好 CPU 时间片到了，线程被挂起，不能继续执行了。 **记住此时，线程一中记录的 e = A ， e.next = B。**

然后线程二正常执行，扩容后的数组长度为 4， 假设 A，B两个元素又碰撞到了同一个桶中。然后，通过几次 while 循环后，采用头插法，最终呈现的结构如下：

![HashMap(JDK8)知识汇总](https://gitee.com/HNov/image/raw/master/typora/20200417004120.jpeg)



此时，线程一解挂，继续往下执行。注意，此时线程一，记录的还是 e = A，e.next = B，因为它还未感知到最新的变化。

我们主要关注图1中标注的①②③④处的变量变化：

```
/**
* next = e.next
* e.next = newTable[i]
* newTable[i] = e;
* e = next;
*/

//第一次循环,(伪代码)
e=A;next=B;
e.next=null //此时线程一的新数组刚初始化完成，还没有元素
newTab[i] = A->null //把A节点头插到新数组中
e=B; //下次循环的e值
```

第一次循环结束后，线程一新数组的结构如下图：

![HashMap(JDK8)知识汇总](https://gitee.com/HNov/image/raw/master/typora/20200417004126.jpeg)



然后，由于 e=B，不为空，进入第二次循环。

```
//第二次循环
e=B;next=A;  //此时A，B的内容已经被线程二修改为 B->A->null，然后被线程一读到，所以B的下一个节点指向A
e.next=A->null  // A->null 为第一次循环后线程一新数组的结构
newTab[i] = B->A->null //新节点B插入之后，线程一新数组的结构
e=A;  //下次循环的 e 值
```

第二次循环结束后，线程一新数组的结构如下图：

![HashMap(JDK8)知识汇总](https://gitee.com/HNov/image/raw/master/typora/20200417004137.jpeg)



此时，由于 e=A，不为空，继续循环。

```
//第三次循环
e=A;next=null;  // A节点后边已经没有节点了
e.next= B->A->null  // B->A->null 为第二次循环后线程一新数组的结构
//我们把A插入后，抽象的表达为 A->B->A->null，但是，A只能是一个，不能分身啊
//因此实际上是 e(A).next指向发生了变化，A的 next 由指向 null 改为指向了 B，
//而 B 本身又指向A，因此A和B互相指向，成环
newTab[i] = A->B 且 B->A 
e=next=null; //e此时为空，结束循环
```

第三次循环结束后，看下图，A的指向由 null ，改为指向为 B，因此 A 和 B 之间成环。

![HashMap(JDK8)知识汇总](https://gitee.com/HNov/image/raw/master/typora/20200417004142.jpeg)



这时，有的同学可能就会问了，就算他们成环了，又怎样，跟死循环有什么关系？

我们看下 get() 方法（最终调用 getEntry 方法），

![HashMap(JDK8)知识汇总](https://gitee.com/HNov/image/raw/master/typora/20200417004148.jpeg)



可以看到查找元素时，只要 e 不为空，就会一直循环查找下去。若有某个元素 C 的 hash 值也落在了和 A，B元素同一个桶中，则会由于， A，B互相指向，e.next 永远不为空，就会形成死循环。

# 五、题外话

## 常考ArrayList和Vector区别?

vector是线程安全的,它在add方法的时候使用了同步函数,方法上加了synchronized关键字,而ArrayList的add方法没有添加这个关键字.
当存储空间不足的时候,ArrayList默认大小变为原来的1.5倍,而vector默认大小变为原来的2倍.

## HashSet底层原理

`HashSet底层是HashMap`,这个打死也没想到.实现Set接口,由哈希表支持,不保证set的迭代顺序,允许null元素.

1. 默认构造函数是构建一个初始容量为16,负载因子为0.75的HashMap.封装了一个HashMap对象来存储所有的集合元素,所有放入HashSet中的集合元素实际由HashMap的key来保存,而HashMap的value则存储了一个PRESENT,他是一个静态的Object对象.
2. 将一个对象放入HashSet中保存,重写该类的equals和hashcode方法很重要,而且这两个方法的返回值必须保持一致:当该类的两个hashcode返回值相同时,他们通过equals方法比较.
3. HashSet的其他操作都是基于HashMap的

## 什么是ConcurentHashMap?

`现在用ConcurrentHashMap代替hashTable.`
ConcurrentHashMap采用分段锁,内部分为很多段,每段都相当于一个小的HashTable,他们有自己的锁,只要多个修改操作发生在不同的段上,就可以并发进行.

为了效率一般都会初始化HashMap容量,在这里jdk1.8是构造函数的时候就会实现扩容,而jdk1.7需要在第一次put的时候才会扩容.
jdk1.8构造函数传入的数字并不是HashMap的容量,而是 第一个大于传入数字的2次幂数,如
传入1-->容量2
传入3-->容量4
传入7-->容量8

**ConcurrentHashMap在jdk1.7和1.8的区别:**

参考:https://blog.csdn.net/qq_41884976/article/details/89532816

ConcurrentHashMap的1.8版本,取消了Segment分段锁的结构,而是采用CAS算法以及Synchronized的方式,数组结构和HashMap一致,锁的颗粒度调整为每个table元素,节点的value和next采用 volatile修饰

## 关于Hash值

关于Hash值如果采用求余算法的话,那除数要为不大于散列表长度的最大素数或者不包含质因数小于20的合数.
原因:
除数取合数的话，一旦数据是以合数的某个因子为间隔增长的，那么哈希值只会是几个值不停的重复，冲突很严重，而素数就不会发生这种情况

[Hash算法的讲解](https://www.cnblogs.com/xiohao/p/4389672.html)

## 解决hash冲突的四种常用方法

- 开放地址法；

- 再hash的方法；

- 拉链法；

- 建立公共溢出区法；

参考：https://www.iteye.com/blog/taoyongpan-2401102

# 六、感谢

版权声明：本文参考CSDN博主「老艮头」的原创文章，遵循 CC 4.0 BY-SA 版权协议
原文链接：https://blog.csdn.net/weixin_42340670/article/details/80574965

https://me.csdn.net/weixin_42340670

**参考:**
[https://blog.csdn.net/w_y_x_y/article/details/82288178](https://yq.aliyun.com/go/articleRenderRedirect?url=https%3A%2F%2Fblog.csdn.net%2Fw_y_x_y%2Farticle%2Fdetails%2F82288178)
[http://www.nowamagic.net/academy/detail/3008040](https://yq.aliyun.com/go/articleRenderRedirect?url=http%3A%2F%2Fwww.nowamagic.net%2Facademy%2Fdetail%2F3008040)

https://www.toutiao.com/i6815043369220178435/?tt_from=weixin&utm_campaign=client_share&wxshare_count=1&timestamp=1586961142&app=news_article&utm_source=weixin&utm_medium=toutiao_android&req_id=202004152232210100140470130A47DAE0&group_id=6815043369220178435

