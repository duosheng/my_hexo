---
title: hashMap的实现原理
date: 2017-05-10 14:42:14
tags: java
---

<div class="post">

# [HashMap实现原理及源码分析](http://www.cnblogs.com/chengxiao/p/6059914.html)

<div class="postBody">

<div id="cnblogs_post_body">

　　哈希表（hash table）也叫散列表，是一种非常重要的数据结构，应用场景及其丰富，许多缓存技术（比如memcached）的核心其实就是在内存中维护一张大的哈希表，而HashMap的实现原理也常常出现在各类的面试题中，重要性可见一斑。本文会对java集合框架中的对应实现HashMap的实现原理进行讲解，然后会对JDK7的HashMap源码进行分析。

<span style="font-size: 18px;">**目录**</span>

<span style="color: #000000; font-size: 18px;">**　<span style="font-size: 16px;">　<span style="color: #993366;">一、<span style="font-size: 15px;">[<span style="color: #993366;">什么是哈希表</span>](#t1)</span></span></span>**</span>

<span style="color: #993366; font-size: 15px;">**　　二、[<span style="color: #993366;">HashMap实现原理</span>](#t2)**</span>

<span style="color: #993366; font-size: 15px;">**　　三、[<span style="color: #993366;">为何HashMap的数组长度一定是2的次幂？</span>](#t3)**</span>

<span style="color: #993366; font-size: 15px;">**　　四、[<span style="color: #993366;">重写equals方法需同时重写hashCode方法</span>](#t4)**</span>

<span style="color: #993366; font-size: 15px;">**　　五、[<span style="color: #993366;">总结</span>](#t5)**</span>

# <a name="t1"></a>一、什么是哈希表

　　在讨论哈希表之前，我们先大概了解下其他数据结构在新增，查找等基础操作执行性能

　　**数组**：采用一段连续的存储单元来存储数据。对于指定下标的查找，时间复杂度为O(1)；通过给定值进行查找，需要遍历数组，逐一比对给定关键字和数组元素，时间复杂度为O(n)，当然，对于有序数组，则可采用二分查找，插值查找，斐波那契查找等方式，可将查找复杂度提高为O(logn)；对于一般的插入删除操作，涉及到数组元素的移动，其平均复杂度也为O(n)

　　**线性链表**：对于链表的新增，删除等操作（在找到指定操作位置后），仅需处理结点间的引用即可，时间复杂度为O(1)，而查找操作需要遍历链表逐一进行比对，复杂度为O(n)

　　**二叉树**：对一棵相对平衡的有序二叉树，对其进行插入，查找，删除等操作，平均复杂度均为O(logn)。

　　**哈希表**：相比上述几种数据结构，在哈希表中进行添加，删除，查找等操作，性能十分之高，不考虑哈希冲突的情况下，仅需一次定位即可完成，时间复杂度为O(1)，接下来我们就来看看哈希表是如何实现达到惊艳的常数阶O(1)的。

　　我们知道，数据结构的物理存储结构只有两种：<span style="color: #ff0000;">**顺序存储结构**</span>和<span style="color: #ff0000;">**链式存储结构**<span style="color: #000000;">（像栈，队列，树，图等是从逻辑结构去抽象的，映射到内存中，也这两种物理组织形式），而在上面我们提到过，在数组中根据下标查找某个元素，一次定位就可以达到，哈希表利用了这种特性，<span style="color: #ff0000;">**哈希表的主干就是数组**</span>。</span></span>

<span style="color: #ff0000;"><span style="color: #000000;">　　比如我们要新增或查找某个元素，我们通过把当前元素的关键字 通过某个函数映射到数组中的某个位置，通过数组下标一次定位就可完成操作。</span>**<span style="color: #000000;">
</span>**</span>

<span style="color: #ff0000;"><span style="color: #000000;">　　　　　　　　<span style="font-size: 18px; color: #ff0000;">**存储位置 = f(关键字)**</span></span></span>

<span style="color: #ff0000;"><span style="color: #000000;"><span style="font-size: 18px; color: #ff0000;"><span style="color: #000000; font-size: 15px;">　其中，这个函数f一般称为**哈希函数**，这个函数的设计好坏会直接影响到哈希表的优劣。举个例子，比如我们要在哈希表中执行插入操作：</span></span></span></span>

<span style="color: #ff0000;"><span style="color: #000000;"><span style="font-size: 18px; color: #ff0000;"><span style="color: #000000; font-size: 15px;">　　![](http://images2015.cnblogs.com/blog/1024555/201611/1024555-20161113180447499-1953916974.png)</span></span></span></span>

<span style="color: #ff0000;"><span style="color: #000000;"><span style="font-size: 18px; color: #ff0000;"><span style="color: #000000; font-size: 15px;">　　查找操作同理，先通过哈希函数计算出实际存储地址，然后从数组中对应地址取出即可。</span></span></span></span>

<span style="color: #ff0000;"><span style="color: #000000;"><span style="font-size: 18px; color: #ff0000;"><span style="color: #000000; font-size: 15px;">　　<span style="font-size: 16px;">**哈希冲突**</span></span></span></span></span>

<span style="color: #ff0000;"><span style="color: #000000;"><span style="font-size: 18px; color: #ff0000;"><span style="color: #000000; font-size: 15px;">　　然而万事无完美，如果两个不同的元素，通过哈希函数得出的实际存储地址相同怎么办？也就是说，当我们对某个元素进行哈希运算，得到一个存储地址，然后要进行插入的时候，发现已经被其他元素占用了，其实这就是所谓的<span style="color: #ff0000;">**哈希冲突**</span>，也叫哈希碰撞。前面我们提到过，哈希函数的设计至关重要，好的哈希函数会尽可能地保证 **计算简单**和**散列地址分布均匀,**但是，我们需要清楚的是，数组是一块连续的固定长度的内存空间，再好的哈希函数也不能保证得到的存储地址绝对不发生冲突。那么哈希冲突如何解决呢？哈希冲突的解决方案有多种:开放定址法（发生冲突，继续寻找下一块未被占用的存储地址），再散列函数法，链地址法，而HashMap即是采用了链地址法，也就是**数组+链表**的方式，</span></span></span></span>

# <a name="t2"></a>二、HashMap实现原理

　HashMap的主干是一个Entry数组。Entry是HashMap的基本组成单元，每一个Entry包含一个key-value键值对。

<div class="cnblogs_code">

<pre><span style="color: #008000;">//</span><span style="color: #008000;">HashMap的主干数组，可以看到就是一个Entry数组，初始值为空数组{}，主干数组的长度一定是2的次幂，至于为什么这么做，后面会有详细分析。</span>
<span style="color: #0000ff;">transient</span> Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;</pre>

</div>

 Entry是HashMap中的一个静态内部类。代码如下

<div class="cnblogs_code">

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

<pre>    <span style="color: #0000ff;">static</span> <span style="color: #0000ff;">class</span> Entry<K,V> <span style="color: #0000ff;">implements</span> Map.Entry<K,V> <span style="color: #000000;">{</span> <span style="color: #0000ff;">final</span> <span style="color: #000000;">K key;
        V value;
        Entry</span><K,V> <span style="color: #000000;">next;<span style="color: #008000;">//存储指向下一个Entry的引用，单链表结构</span></span><span style="color: #0000ff;">int</span> <span style="color: #000000;">hash;<span style="color: #008000;">//对key的hashcode值进行hash运算后得到的值，存储在Entry，避免重复计算</span></span><span style="color: #008000;">/**</span> <span style="color: #008000;">* Creates new entry.</span> <span style="color: #008000;">*/</span> <span style="color: #000000;">Entry(</span><span style="color: #0000ff;">int</span> h, K k, V v, Entry<K,V> <span style="color: #000000;">n) {
            value</span> = <span style="color: #000000;">v;
            next</span> = <span style="color: #000000;">n;
            key</span> = <span style="color: #000000;">k;
            hash</span> = <span style="color: #000000;">h;
        }</span> </pre>

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

</div>

 所以，HashMap的整体结构如下

<span style="color: #000000; font-size: 16px;">![](http://images2015.cnblogs.com/blog/1024555/201611/1024555-20161113235348670-746615111.png)<span style="font-size: 15px;">　　</span></span>

<span style="font-size: 15px;">　　<span style="color: #000000;">**简单来说，HashMap由数组+链表组成的，数组是HashMap的主体，链表则是主要为了解决哈希冲突而存在的，如果定位到的数组位置不含链表（当前entry的next指向null）,那么对于查找，添加等操作很快，仅需一次寻址即可；如果定位到的数组包含链表，对于添加操作，其时间复杂度依然为O(1)，因为最新的Entry会插入链表头部，急需要简单改变引用链即可，而对于查找操作来讲，此时就需要遍历链表，然后通过key对象的equals方法逐一比对查找。所以，性能考虑，HashMap中的链表出现越少，性能才会越好。**</span></span>

其他几个重要字段

<div class="cnblogs_code">

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

<pre><span style="color: #008000;">//</span><span style="color: #008000;">实际存储的key-value键值对的个数</span>
<span style="color: #0000ff;">transient</span> <span style="color: #0000ff;">int</span> <span style="color: #000000;">size;</span> <span style="color: #008000;">//</span><span style="color: #008000;">阈值，当table == {}时，该值为初始容量（初始容量默认为16）；当table被填充了，也就是为table分配内存空间后，threshold一般为 capacity*loadFactory。HashMap在进行扩容时需要参考threshold，后面会详细谈到</span>
<span style="color: #0000ff;">int</span> <span style="color: #000000;">threshold;</span> <span style="color: #008000;">//</span><span style="color: #008000;">负载因子，代表了table的填充度有多少，默认是0.75</span>
<span style="color: #0000ff;">final</span> <span style="color: #0000ff;">float</span> <span style="color: #000000;">loadFactor;</span> <span style="color: #008000;">//用于快速失败，由于HashMap非线程安全，在对HashMap进行迭代时，如果</span><span style="color: #008000;">期间其他线程的参与导致HashMap的结构发生变化了（比如put，remove等操作），需要抛出异常ConcurrentModificationException</span>
<span style="color: #0000ff;">transient</span> <span style="color: #0000ff;">int</span> modCount;</pre>

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

</div>

HashMap有4个构造器，其他构造器如果用户没有传入initialCapacity 和loadFactor这两个参数，会使用默认值

initialCapacity默认为16，loadFactory默认为0.75

我们看下其中一个

<div class="cnblogs_code">

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

<pre><span style="color: #0000ff;">public</span> HashMap(<span style="color: #0000ff;">int</span> initialCapacity, <span style="color: #0000ff;">float</span> <span style="color: #000000;">loadFactor) {
　　　　　<span style="color: #008000;">//此处对传入的初始容量进行校验，最大不能超过MAXIMUM_CAPACITY = 1<<30(2<sup>30</sup>)</span></span><span style="color: #0000ff;">if</span> (initialCapacity < 0<span style="color: #000000;">)</span> <span style="color: #0000ff;">throw</span> <span style="color: #0000ff;">new</span> IllegalArgumentException("Illegal initial capacity: " + <span style="color: #000000;">initialCapacity);</span> <span style="color: #0000ff;">if</span> (initialCapacity > <span style="color: #000000;">MAXIMUM_CAPACITY)
            initialCapacity</span> = <span style="color: #000000;">MAXIMUM_CAPACITY;</span> <span style="color: #0000ff;">if</span> (loadFactor <= 0 || <span style="color: #000000;">Float.isNaN(loadFactor))</span> <span style="color: #0000ff;">throw</span> <span style="color: #0000ff;">new</span> IllegalArgumentException("Illegal load factor: " + <span style="color: #000000;">loadFactor);</span> <span style="color: #0000ff;">this</span>.loadFactor = <span style="color: #000000;">loadFactor;
        threshold</span> = <span style="color: #000000;">initialCapacity;

        init();<span style="color: #008000;">//init方法在HashMap中没有实际实现，不过在其子类如 linkedHashMap中就会有对应实现</span>
    }</span></pre>

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

</div>

　　从上面这段代码我们可以看出，**<span style="color: #000000;">在常规构造器中，没有为数组table分配内存空间（有一个入参为指定Map的构造器例外），而是在执行put操作的时候才真正构建table数组</span>**

<span style="color: #000000; font-size: 14px;">　　OK,接下来我们来看看put操作的实现吧</span>

<div class="cnblogs_code">

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

<pre>    <span style="color: #0000ff;">public</span> <span style="color: #000000;">V put(K key, V value) {
        <span style="color: #008000;">//如果table数组为空数组{}，进行数组填充（为table分配实际内存空间），入参为threshold，此时threshold为initialCapacity 默认是1</span></span><span style="color: #008000;"><<4(2<sup>4</sup>=16)</span> <span style="color: #0000ff;">if</span> (table == <span style="color: #000000;">EMPTY_TABLE) {
            inflateTable(threshold);
        }</span> <span style="color: #008000;">//</span><span style="color: #008000;">如果key为null，存储位置为table[0]或table[0]的冲突链上</span>
        <span style="color: #0000ff;">if</span> (key == <span style="color: #0000ff;">null</span><span style="color: #000000;">)</span> <span style="color: #0000ff;">return</span> <span style="color: #000000;">putForNullKey(value);</span> <span style="color: #0000ff;">int</span> hash = hash(key);<span style="color: #008000;">//</span><span style="color: #008000;">对key的hashcode进一步计算，确保散列均匀</span>
        <span style="color: #0000ff;">int</span> i = indexFor(hash, table.length);<span style="color: #008000;">//</span><span style="color: #008000;">获取在table中的实际位置</span>
        <span style="color: #0000ff;">for</span> (Entry<K,V> e = table[i]; e != <span style="color: #0000ff;">null</span>; e = <span style="color: #000000;">e.next) {</span> <span style="color: #008000;">//</span><span style="color: #008000;">如果该对应数据已存在，执行覆盖操作。用新value替换旧value，并返回旧value</span>
 <span style="color: #000000;">Object k;</span> <span style="color: #0000ff;">if</span> (e.hash == hash && ((k = e.key) == key || <span style="color: #000000;">key.equals(k))) {
                V oldValue</span> = <span style="color: #000000;">e.value;
                e.value</span> = <span style="color: #000000;">value;
                e.recordAccess(</span><span style="color: #0000ff;">this</span><span style="color: #000000;">);</span> <span style="color: #0000ff;">return</span> <span style="color: #000000;">oldValue;
            }
        }
        modCount</span>++<span style="color: #000000;">;<span style="color: #008000;">//保证并发访问时，若HashMap内部结构发生变化，快速响应失败</span>
        addEntry(hash, key, value, i);</span><span style="color: #008000;">//</span><span style="color: #008000;">新增一个entry</span>
        <span style="color: #0000ff;">return</span> <span style="color: #0000ff;">null</span><span style="color: #000000;">;
    }</span> </pre>

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

</div>

 先来看看inflateTable这个方法

<div class="cnblogs_code">

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

<pre><span style="color: #0000ff;">private</span> <span style="color: #0000ff;">void</span> inflateTable(<span style="color: #0000ff;">int</span> <span style="color: #000000;">toSize) {</span> <span style="color: #0000ff;">int</span> capacity = <span style="color: #000000;">roundUpToPowerOf2(toSize);<span style="color: #008000;">//capacity一定是2的次幂</span>
        threshold</span> = (<span style="color: #0000ff;">int</span>) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1<span style="color: #000000;">);<span style="color: #008000;">//此处为threshold赋值，取capacity*loadFactor和MAXIMUM_CAPACITY+1的最小值，capaticy一定不会超过MAXIMUM_CAPACITY，除非loadFactor大于1</span>
        table</span> = <span style="color: #0000ff;">new</span> <span style="color: #000000;">Entry[capacity];
        initHashSeedAsNeeded(capacity);
    }</span></pre>

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

</div>

　　inflateTable这个方法用于为主干数组table在内存中分配存储空间，通过roundUpToPowerOf2(toSize)可以确保capacity为大于或等于toSize的最接近toSize的二次幂，比如toSize=13,则capacity=16;to_size=16,capacity=16;to_size=17,capacity=32.

<div class="cnblogs_code">

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

<pre> <span style="color: #0000ff;">private</span> <span style="color: #0000ff;">static</span> <span style="color: #0000ff;">int</span> roundUpToPowerOf2(<span style="color: #0000ff;">int</span> <span style="color: #000000;">number) {</span> <span style="color: #008000;">//</span> <span style="color: #008000;">assert number >= 0 : "number must be non-negative";</span>
        <span style="color: #0000ff;">return</span> number >= <span style="color: #000000;">MAXIMUM_CAPACITY</span> ? <span style="color: #000000;">MAXIMUM_CAPACITY
                : (number</span> > 1) ? Integer.highestOneBit((number - 1) << 1) : 1<span style="color: #000000;">;
    }</span></pre>

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

</div>

roundUpToPowerOf2中的这段处理使得数组长度一定为2的次幂，Integer.highestOneBit是用来获取最左边的bit（其他bit位为0）所代表的数值.

hash函数

<div class="cnblogs_code">

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

<pre><span style="color: #008000;">//</span><span style="color: #008000;">这是一个神奇的函数，用了很多的异或，移位等运算，对key的hashcode进一步进行计算以及二进制位的调整等来保证最终获取的存储位置尽量分布均匀</span>
<span style="color: #0000ff;">final</span> <span style="color: #0000ff;">int</span> <span style="color: #000000;">hash(Object k) {</span> <span style="color: #0000ff;">int</span> h = <span style="color: #000000;">hashSeed;</span> <span style="color: #0000ff;">if</span> (0 != h && k <span style="color: #0000ff;">instanceof</span> <span style="color: #000000;">String) {</span> <span style="color: #0000ff;">return</span> <span style="color: #000000;">sun.misc.Hashing.stringHash32((String) k);
        }

        h</span> ^= <span style="color: #000000;">k.hashCode();

        h</span> ^= (h >>> 20) ^ (h >>> 12<span style="color: #000000;">);</span> <span style="color: #0000ff;">return</span> h ^ (h >>> 7) ^ (h >>> 4<span style="color: #000000;">);
    }</span></pre>

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

</div>

以上hash函数计算出的值，通过indexFor进一步处理来获取实际的存储位置

<div class="cnblogs_code">

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

<pre><span style="color: #008000;">　　/**</span> <span style="color: #008000;">* 返回数组下标</span> <span style="color: #008000;">*/</span>
    <span style="color: #0000ff;">static</span> <span style="color: #0000ff;">int</span> indexFor(<span style="color: #0000ff;">int</span> h, <span style="color: #0000ff;">int</span> <span style="color: #000000;">length) {</span> <span style="color: #0000ff;">return</span> h & (length-1<span style="color: #000000;">);
    }</span></pre>

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

</div>

h&（length-1）保证获取的index一定在数组范围内，举个例子，默认容量16，length-1=15，h=18,转换成二进制计算为

<div class="cnblogs_code">

<pre> <span style="color: #ff0000;">1  0  0  1  0
    &   0  1  1  1  1
    __________________
        0  0  0  1  0    = 2</span></pre>

</div>

　　最终计算出的index=2。有些版本的对于此处的计算会使用 取模运算，也能保证index一定在数组范围内，不过位运算对计算机来说，性能更高一些（HashMap中有大量位运算）

所以最终存储位置的确定流程是这样的：

![](http://images2015.cnblogs.com/blog/1024555/201611/1024555-20161115133556388-1098209938.png)

再来看看addEntry的实现：

<div class="cnblogs_code">

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

<pre><span style="color: #0000ff;">void</span> addEntry(<span style="color: #0000ff;">int</span> hash, K key, V value, <span style="color: #0000ff;">int</span> <span style="color: #000000;">bucketIndex) {</span> <span style="color: #0000ff;">if</span> ((size >= threshold) && (<span style="color: #0000ff;">null</span> != <span style="color: #000000;">table[bucketIndex])) {
            resize(</span>2 * <span style="color: #000000;">table.length);<span style="color: #008000;">//当size超过临界阈值threshold，并且即将发生哈希冲突时进行扩容</span>
            hash</span> = (<span style="color: #0000ff;">null</span> != key) ? hash(key) : 0<span style="color: #000000;">;
            bucketIndex</span> = <span style="color: #000000;">indexFor(hash, table.length);
        }

        createEntry(hash, key, value, bucketIndex);
    }</span></pre>

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

</div>

　　通过以上代码能够得知，当发生哈希冲突并且size大于阈值的时候，需要进行数组扩容，扩容时，需要新建一个长度为之前数组2倍的新的数组，然后将当前的Entry数组中的元素全部传输过去，扩容后的新数组长度为之前的2倍，所以扩容相对来说是个耗资源的操作。

# <a name="t3"></a>三、为何HashMap的数组长度一定是2的次幂？

我们来继续看上面提到的resize方法

<div class="cnblogs_code">

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

<pre> <span style="color: #0000ff;">void</span> resize(<span style="color: #0000ff;">int</span> <span style="color: #000000;">newCapacity) {
        Entry[] oldTable</span> = <span style="color: #000000;">table;</span> <span style="color: #0000ff;">int</span> oldCapacity = <span style="color: #000000;">oldTable.length;</span> <span style="color: #0000ff;">if</span> (oldCapacity == <span style="color: #000000;">MAXIMUM_CAPACITY) {
            threshold</span> = <span style="color: #000000;">Integer.MAX_VALUE;</span> <span style="color: #0000ff;">return</span><span style="color: #000000;">;
        }

        Entry[] newTable</span> = <span style="color: #0000ff;">new</span> <span style="color: #000000;">Entry[newCapacity];
        transfer(newTable, initHashSeedAsNeeded(newCapacity));
        table</span> = <span style="color: #000000;">newTable;
        threshold</span> = (<span style="color: #0000ff;">int</span>)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1<span style="color: #000000;">);
    }</span></pre>

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

</div>

如果数组进行扩容，数组长度发生变化，而存储位置 index = h&(length-1),index也可能会发生变化，需要重新计算index，我们先来看看transfer这个方法

<div class="cnblogs_code">

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

<pre><span style="color: #0000ff;">void</span> transfer(Entry[] newTable, <span style="color: #0000ff;">boolean</span> <span style="color: #000000;">rehash) {</span> <span style="color: #0000ff;">int</span> newCapacity = <span style="color: #000000;">newTable.length;
　　　　　<span style="color: #008000;">//for循环中的代码，逐个遍历链表，重新计算索引位置，将老数组数据复制到新数组中去（数组不存储实际数据，所以仅仅是拷贝引用而已）</span></span><span style="color: #0000ff;">for</span> (Entry<K,V> <span style="color: #000000;">e : table) {</span> <span style="color: #0000ff;">while</span>(<span style="color: #0000ff;">null</span> != <span style="color: #000000;">e) {
                Entry</span><K,V> next = <span style="color: #000000;">e.next;</span> <span style="color: #0000ff;">if</span> <span style="color: #000000;">(rehash) {
                    e.hash</span> = <span style="color: #0000ff;">null</span> == e.key ? 0 <span style="color: #000000;">: hash(e.key);
                }</span> <span style="color: #0000ff;">int</span> i = <span style="color: #000000;">indexFor(e.hash, newCapacity);
　　　　　　　　　 <span style="color: #008000;">//将当前entry的next链指向新的索引位置,newTable[i]有可能为空，有可能也是个entry链，如果是entry链，直接在链表头部插入。</span>
                e.next</span> = <span style="color: #000000;">newTable[i];
                newTable[i]</span> = <span style="color: #000000;">e;
                e</span> = <span style="color: #000000;">next;
            }
        }
    }</span></pre>

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

</div>

　　这个方法将老数组中的数据逐个链表地遍历，扔到新的扩容后的数组中，我们的数组索引位置的计算是通过 对key值的hashcode进行hash扰乱运算后，再通过和 length-1进行位运算得到最终数组索引位置。

　　hashMap的数组长度一定保持2的次幂，比如16的二进制表示为 10000，那么length-1就是15，二进制为01111，同理扩容后的数组长度为32，二进制表示为100000，length-1为31，二进制表示为011111。从下图可以我们也能看到这样会保证低位全为1，而扩容后只有一位差异，也就是多出了最左位的1，这样在通过 h&(length-1)的时候，只要h对应的最左边的那一个差异位为0，就能保证得到的新的数组索引和老数组索引一致(大大减少了之前已经散列良好的老数组的数据位置重新调换)，个人理解。

　　![](http://images2015.cnblogs.com/blog/1024555/201611/1024555-20161115215812138-679881037.png)

 还有，数组长度保持2的次幂，length-1的低位都为1，会使得获得的数组索引index更加均匀，比如：

![](http://images2015.cnblogs.com/blog/1024555/201611/1024555-20161116001404732-625340289.png)

　　我们看到，上面的&运算，高位是不会对结果产生影响的（hash函数采用各种位运算可能也是为了使得低位更加散列），我们只关注低位bit，如果低位全部为1，那么对于h低位部分来说，任何一位的变化都会对结果产生影响，也就是说，要得到index=21这个存储位置，h的低位只有这一种组合。这也是数组长度设计为必须为2的次幂的原因。

![](http://images2015.cnblogs.com/blog/1024555/201611/1024555-20161116001717560-1455096254.png)

　　如果不是2的次幂，也就是低位不是全为1此时，要使得index=21，h的低位部分不再具有唯一性了，哈希冲突的几率会变的更大，同时，index对应的这个bit位无论如何不会等于1了，而对应的那些数组位置也就被白白浪费了。

get方法

<div class="cnblogs_code">

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

<pre> <span style="color: #0000ff;">public</span> <span style="color: #000000;">V get(Object key) {
　　　　 <span style="color: #008000;">//如果key为null,则直接去table[0]处去检索即可。</span></span><span style="color: #0000ff;">if</span> (key == <span style="color: #0000ff;">null</span><span style="color: #000000;">)</span> <span style="color: #0000ff;">return</span> <span style="color: #000000;">getForNullKey();
        Entry</span><K,V> entry = <span style="color: #000000;">getEntry(key);</span> <span style="color: #0000ff;">return</span> <span style="color: #0000ff;">null</span> == entry ? <span style="color: #0000ff;">null</span> <span style="color: #000000;">: entry.getValue();
 }</span></pre>

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

</div>

get方法通过key值返回对应value，如果key为null，直接去table[0]处检索。我们再看一下getEntry这个方法

<div class="cnblogs_code">

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

<pre><span style="color: #0000ff;">final</span> Entry<K,V> <span style="color: #000000;">getEntry(Object key) {</span> <span style="color: #0000ff;">if</span> (size == 0<span style="color: #000000;">) {</span> <span style="color: #0000ff;">return</span> <span style="color: #0000ff;">null</span><span style="color: #000000;">;
        }</span> <span style="color: #008000;">//</span><span style="color: #008000;">通过key的hashcode值计算hash值</span>
        <span style="color: #0000ff;">int</span> hash = (key == <span style="color: #0000ff;">null</span>) ? 0 <span style="color: #000000;">: hash(key);</span> <span style="color: #008000;">//</span><span style="color: #008000;">indexFor (hash&length-1) 获取最终数组索引，然后遍历链表，通过equals方法比对找出对应记录</span>
        <span style="color: #0000ff;">for</span> (Entry<K,V> e = <span style="color: #000000;">table[indexFor(hash, table.length)];
             e</span> != <span style="color: #0000ff;">null</span><span style="color: #000000;">;
             e</span> = <span style="color: #000000;">e.next) {
            Object k;</span> <span style="color: #0000ff;">if</span> (e.hash == hash && <span style="color: #000000;">((k</span> = e.key) == key || (key != <span style="color: #0000ff;">null</span> && <span style="color: #000000;">key.equals(k))))</span> <span style="color: #0000ff;">return</span> <span style="color: #000000;">e;
        }</span> <span style="color: #0000ff;">return</span> <span style="color: #0000ff;">null</span><span style="color: #000000;">;
    }</span> </pre>

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

</div>

　　可以看出，get方法的实现相对简单，key(hashcode)-->hash-->indexFor-->最终索引位置，找到对应位置table[i]，再查看是否有链表，遍历链表，通过key的equals方法比对查找对应的记录。要注意的是，有人觉得上面在定位到数组位置之后然后遍历链表的时候，e.hash == hash这个判断没必要，仅通过equals判断就可以。其实不然，试想一下，如果传入的key对象重写了equals方法却没有重写hashCode，而恰巧此对象定位到这个数组位置，如果仅仅用equals判断可能是相等的，但其hashCode和当前对象不一致，这种情况，根据Object的hashCode的约定，不能返回当前对象，而应该返回null，后面的例子会做出进一步解释。

# <a name="t4"></a>四、重写equals方法需同时重写hashCode方法

　　关于HashMap的源码分析就介绍到这儿了，最后我们再聊聊老生常谈的一个问题，各种资料上都会提到，“重写equals时也要同时覆盖hashcode”，我们举个小例子来看看，如果重写了equals而不重写hashcode会发生什么样的问题

<div class="cnblogs_code">

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

<pre><span style="color: #008000;">/**</span> <span style="color: #008000;">* Created by chengxiao on 2016/11/15.</span> <span style="color: #008000;">*/</span>
<span style="color: #0000ff;">public</span> <span style="color: #0000ff;">class</span> <span style="color: #000000;">MyTest {</span> <span style="color: #0000ff;">private</span> <span style="color: #0000ff;">static</span> <span style="color: #0000ff;">class</span> <span style="color: #000000;">Person{</span> <span style="color: #0000ff;">int</span> <span style="color: #000000;">idCard;
        String name;</span> <span style="color: #0000ff;">public</span> Person(<span style="color: #0000ff;">int</span> <span style="color: #000000;">idCard, String name) {</span> <span style="color: #0000ff;">this</span>.idCard = <span style="color: #000000;">idCard;</span> <span style="color: #0000ff;">this</span>.name = <span style="color: #000000;">name;
        }
        @Override</span> <span style="color: #0000ff;">public</span> <span style="color: #0000ff;">boolean</span> <span style="color: #000000;">equals(Object o) {</span> <span style="color: #0000ff;">if</span> (<span style="color: #0000ff;">this</span> == <span style="color: #000000;">o) {</span> <span style="color: #0000ff;">return</span> <span style="color: #0000ff;">true</span><span style="color: #000000;">;
            }</span> <span style="color: #0000ff;">if</span> (o == <span style="color: #0000ff;">null</span> || getClass() != <span style="color: #000000;">o.getClass()){</span> <span style="color: #0000ff;">return</span> <span style="color: #0000ff;">false</span><span style="color: #000000;">;
            }
            Person person</span> = <span style="color: #000000;">(Person) o;</span> <span style="color: #008000;">//</span><span style="color: #008000;">两个对象是否等值，通过idCard来确定</span>
            <span style="color: #0000ff;">return</span> <span style="color: #0000ff;">this</span>.idCard == <span style="color: #000000;">person.idCard;
        }

    }</span> <span style="color: #0000ff;">public</span> <span style="color: #0000ff;">static</span> <span style="color: #0000ff;">void</span> <span style="color: #000000;">main(String []args){
        HashMap</span><Person,String> map = <span style="color: #0000ff;">new</span> HashMap<Person, String><span style="color: #000000;">();
        Person person</span> = <span style="color: #0000ff;">new</span> Person(1234,"乔峰"<span style="color: #000000;">);</span> <span style="color: #008000;">//</span><span style="color: #008000;">put到hashmap中去</span>
        map.put(person,"天龙八部"<span style="color: #000000;">);</span> <span style="color: #008000;">//</span><span style="color: #008000;">get取出，从逻辑上讲应该能输出“天龙八部”</span>
        System.out.println("结果:"+map.get(<span style="color: #0000ff;">new</span> Person(1234,"萧峰"<span style="color: #000000;">)));
    }
}</span></pre>

<div class="cnblogs_code_toolbar"><span class="cnblogs_code_copy">[![复制代码](//common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")</span></div>

</div>

实际输出结果：

<div class="cnblogs_code">

<pre><span style="color: #ff0000;">结果：null</span></pre>

</div>

　　如果我们已经对HashMap的原理有了一定了解，这个结果就不难理解了。尽管我们在进行get和put操作的时候，使用的key从逻辑上讲是等值的（通过equals比较是相等的），但由于没有重写hashCode方法，所以put操作时，key(hashcode1)-->hash-->indexFor-->最终索引位置 ，而通过key取出value的时候 key(hashcode1)-->hash-->indexFor-->最终索引位置，由于hashcode1不等于hashcode2，导致没有定位到一个数组位置而返回逻辑上错误的值null（也有可能碰巧定位到一个数组位置，但是也会判断其entry的hash值是否相等，上面get方法中有提到。）

　　所以，在重写equals的方法的时候，必须注意重写hashCode方法，同时还要保证通过equals判断相等的两个对象，调用hashCode方法要返回同样的整数值。而如果equals判断不相等的两个对象，其hashCode可以相同（只不过会发生哈希冲突，应尽量避免）。

# <a name="t5"></a>五、总结

　　本文描述了HashMap的实现原理，并结合源码做了进一步的分析，也涉及到一些源码细节设计缘由，最后简单介绍了为什么重写equals的时候需要重写hashCode方法。希望本篇文章能帮助到大家，同时也欢迎讨论指正，谢谢支持！

</div>

<div style="display: block;" id="MySignature">

<div style="margin-bottom: 10px; padding: 5px 10px; background-color: #D9FFFF; border: #53FF53 1px solid;">

作者： [dreamcatcher-cx](http://www.cnblogs.com/chengxiao/)

出处： [<http://www.cnblogs.com/chengxiao/>](http://www.cnblogs.com/chengxiao/)

本文版权归作者和博客园共有，欢迎转载，但未经作者同意必须保留此段声明，且在页面明显位置给出原文链接。

</div>

</div>

<div id="blog_post_info_block">

<div id="BlogPostCategory">分类: [java集合框架](http://www.cnblogs.com/chengxiao/category/909714.html)</div>

<div id="blog_post_info">

<div id="green_channel">[好文要顶](javascript:void(0);) [关注我](javascript:void(0);) [收藏该文](javascript:void(0);) [![](//common.cnblogs.com/images/icon_weibo_24.png)](javascript:void(0); "分享至新浪微博") [![](//common.cnblogs.com/images/wechat.png)](javascript:void(0); "分享至微信")</div>

<div id="author_profile">

<div id="author_profile_info" class="author_profile_info">[![](//pic.cnblogs.com/face/1024555/20170217101538.png)](http://home.cnblogs.com/u/chengxiao/)

<div id="author_profile_detail" class="author_profile_info">[dreamcatcher-cx](http://home.cnblogs.com/u/chengxiao/)
[关注 - 19](http://home.cnblogs.com/u/chengxiao/followees)
[粉丝 - 21](http://home.cnblogs.com/u/chengxiao/followers)</div>

</div>

<div id="author_profile_follow">[+加关注](javascript:void(0);)</div>

</div>

<div id="div_digg">

<div class="diggit" onclick="votePost(6059914,'Digg')"><span class="diggnum" id="digg_count">11</span></div>

<div class="buryit" onclick="votePost(6059914,'Bury')"><span class="burynum" id="bury_count">0</span></div>

</div>

</div>

<div id="post_next_prev">[«](http://www.cnblogs.com/chengxiao/p/6032183.html) 上一篇：[Oracle约束(Constraint)详解](http://www.cnblogs.com/chengxiao/p/6032183.html "发布于2016-11-06 22:34")
[»](http://www.cnblogs.com/chengxiao/p/6103002.html) 下一篇：[图解排序算法(一)之3种简单排序(选择，冒泡，直接插入)](http://www.cnblogs.com/chengxiao/p/6103002.html "发布于2016-11-26 12:46")
</div>

</div>

</div>

<div class="postDesc">posted @ <span id="post-date">2016-11-16 00:27</span> [dreamcatcher-cx](http://www.cnblogs.com/chengxiao/) 阅读(<span id="post_view_count">4211</span>) 评论(<span id="post_comment_count">2</span>) [编辑](https://i.cnblogs.com/EditPosts.aspx?postid=6059914) [收藏](#)</div>

</div>

<script type="text/javascript">var allowComments=true,cb_blogId=304993,cb_entryId=6059914,cb_blogApp=currentBlogApp,cb_blogUserGuid='8319750a-5977-e611-9fc1-ac853d9f53cc',cb_entryCreatedDate='2016/11/16 0:27:00';loadViewCount(cb_entryId);</script>