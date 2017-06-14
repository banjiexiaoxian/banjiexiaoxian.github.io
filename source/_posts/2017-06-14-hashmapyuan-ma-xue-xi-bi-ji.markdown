---
layout: post
title: "HashMap源码学习笔记"
date: 2017-03-20 18:18:51 +0800
comments: true
categories: JDK
---
理解HashMap的关键，在于理解它底层的数据结构，查找、增加、删除元素的方法，为了理解这些方法，就需要理解Hash函数的原理，HashMap如何触发自动扩容，以及如何解决散列冲突。
本文试图从上述几个关键问题说起来分享一下HashMap源码学习的过程。
###源码中的doc
HashMap大致上跟HashTable相同，但是HashMap是非线程安全的，而且支持Null Key和Null Value.
当有多个线程同时访问HashMap时，若其中存在线程对HashMap进行modify structurally（包括put、remove操作，不包括set、get操作），那么就需要考虑线程同步的HashMap，比如synchronizedMap或者concurrentHashMap。
多线程访问导致HashMap发生死锁的一个案例见[本博客另一篇博文](http://blog.csdn.net/liuyanling_cs/article/details/63276707)
HashMap不能保证插入顺序.
HashMap迭代遍历所有元素的时间复杂度与HashMap的capacity和size之和成正比，所以如果迭代的性能要求较高，就要考虑INITIAL_CAPACITY不能设置过高，LOAD_FACTOR不能设置过低。

```
    //下面是Java HashMap中的默认值
    /**
     * The default initial capacity - MUST be a power of two.
     */
    static final int DEFAULT_INITIAL_CAPACITY = 16;
    /**
     * The load factor used when none specified in constructor.
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
```





####HashMap的Size、Capacity、Load_Factor、Threshold说明 [^1]
![一个典型的HashMap结构](http://img.blog.csdn.net/20160407175503308)
把HashMap抽象成多个用数组形式组织起来的bucket，每个bucket装着的是映射到同一个hashcode位置的KV们
那么bucket的个数就是Capacity
KV的个数就是Size
Load_Factor就是装载因子，用来衡量HashMap的装载情况
Threshold是扩容的阈值，在插入新元素时若发现插入后Size>=（Threshold=Capacity*Load_Factor），则会对HashMap进行扩容（Capacity*2）并将现有元素重新插入到新的HashMap中。



回归正题，理解了HashMap中这几个重要概念之后，就能理解为了良好的权衡时间和空间成本，LOAD_FACTOR的设置比较重要，过高会降低空间成本但增加查找成本，建议使用默认设置。
另外，如果对HashMap中存储的KV对的数量有个比较准确的估计，那么需要合理的设置Capacity的值，尽量减少扩容和Rehash的情况发生。
###HashMap底层的数据结构
这里有几个关键的类，一个是HashMap本身，还有一个是Entry。java优秀封装性的一个体现在于容器类，HashMap也是一个容器类，我们可以在里面装入各种类型的KV对，无论K,V本身的类型。这是通过将KV封装成了一个Entry类来做到的。

```
static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        final int hash;

        public final int hashCode() {
            return (key==null   ? 0 : key.hashCode()) ^
                   (value==null ? 0 : value.hashCode());
        }

        public final String toString() {
            return getKey() + "=" + getValue();
        }
```
HashMap在底层维护了一个数组table，来存放这些Entry。
我们关注一下所有Entry都有一个属性为next，这是为了解决IndexFor冲突问题，在每一个table元素的位置实际上引入了一个链表结构，所有映射成相同Index的Entry将通过next指针组织放在同一条链表中。

```
 public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        threshold = (int)(DEFAULT_INITIAL_CAPACITY * DEFAULT_LOAD_FACTOR);
        table = new Entry[DEFAULT_INITIAL_CAPACITY];
        init();
    }
```

###HashMap中的插入和查找操作
先从put函数说起
```
/**
    put操作
     */
    public V put(K key, V value) {
        if (key == null)
        //由于HashMap支持Key为Null,所以需要单独考虑Key为Null的情况
            return putForNullKey(value);
        //key本身是一个对象，但key对象的hashCode生成方法该对象自带的，可能是jdk定义的，也可能是用户自定义的，是不可控的，可能是比较差的hash函数，所以需要再进行一次hash，根据这个二次hash值来定位应该把当前Entry放到HashMap的第i个bucket中    
        int hash = hash(key.hashCode());
        int i = indexFor(hash, table.length);
        
        //将该Entry插入第i个bucket维护的链表结尾
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            //如果该Entry已存在，返回旧值
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
	//如果是新插入的Entry,将HashMap的版本码++，并将该Entry加入到链表结尾
        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }
	
	  void addEntry(int hash, K key, V value, int bucketIndex) {
		Entry<K,V> e = table[bucketIndex];
	        table[bucketIndex] = new Entry<K,V>(hash, key, value, e);
	        //在插入时会判断是否需要扩容
	        if (size++ >= threshold)
	            resize(2 * table.length);
	    }
```
在put的过程中，有两个很重要的函数需要理解一下，分别是hash和indexfor。
hash的作用在前文已说明，是为了防御Key对象拥有的可能性能较差的hash函数。
当我们拥有了一个Key的hashCode，如何根据这个hashCode将这个Entry映射到数量为Capacity的某个bucket中去呢？
hashCode是一个32位的Int数，可表示的大小从-2147483648到2147483648，如果HashMap能有这么大的容量，那么根据HashCode的性质，若Key不同，则根本不存在冲突的可能。
当然不可能这么做。
比较容易想到的方法是将hashCode根据数组长度来取模。
然而取模运算比较低效，我们只需要达成一个目的，那就是hashCode会被映射为一个小于Capacity的值，并尽量均匀分布。具体做法见indexFor函数。
indexFor函数实际上只利用了HashCode的低n位来做散列，这里为了防止映射冲突，有两个trick[^2]：
1、Capacity需得是2的n次方幂；
2、hash优化函数将HashCode的每个四位都做了一次异或，意在混合原始哈希码的各个部分，以此来加大低位的随机性
```
/**
    HashMap中定义的Hash方法，Null始终被映射为0
     */
    static int hash(int h) {
        
        //java1.7
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
        //java1.8中简化了这个过程，具体见文末相关文章
    }

    /**
     * Returns index for hash code h.
     */
    static int indexFor(int h, int length) {
        return h & (length-1);
    }
```


理解了put操作，get操作就简单了
```

    public V get(Object key) {
        if (key == null)
            return getForNullKey();
        int hash = hash(key.hashCode());
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
                return e.value;
        }
        return null;
    }
```


----------
##相关文章
[^1]:[ HashMap中capacity、loadFactor、threshold、size等概念的解释 ](http://blog.csdn.net/fan2012huan/article/details/51087722)

[^2]:[知乎：JDK 源码中 HashMap 的 hash 方法原理是什么？](https://www.zhihu.com/question/20733617)

