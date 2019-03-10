本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### Hashtable

注意是Hashtable不是HashTable(t为小写)，这不是违背了驼峰定理了嘛？这还得从Hashtable的出生说起，Hashtable是在`Java1.0`的时候创建的，而集合的统一规范命名是在后来的`Java2`开始约定的，而当时又发布了新的集合代替它，所以这个命名也一直使用到现在，所以Hashtable是一个**过时**的集合了，不推崇大家使用这个类，虽说Hashtable是过时的了，我们还是有必要分析一下它，以便对Java集合框架有一个整体的认知。
 首先`Hashtable`采用**拉链法**处理哈希冲突，是**线程安全**的，键值不允许为`null`，然后Hashtable继承自Dictionary，实现Map接口，Hashtable有几个重要的成员变量`table`、`count`、`threshold`、`loadFactor`

- table：是一个`Entry[]`数据类型，而`Entry`实际是一个单链表
- count：Hashtable的大小，即Hashtable中保存的键值对数量
- threshold：Hashtable的阈值，用于判断是否需要调整Hashtable的容量，threshold = 容量*负载因子*，threshold=11*0.75 取整即8
- loadFactor：用来实现快速失败机制的 



#### 构造函数

`Hashtable`有4个构造函数

```
//无参构造函数 默认Hashtable容量是11，默认负载因子是0.75
public Hashtable() {
    this(11, 0.75f);
}

//指定Hashtable容量，默认负载因子是0.75
public Hashtable(int initialCapacity) {
    this(initialCapacity, 0.75f);
}

//指定Hashtable的容量和负载因子
public Hashtable(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal Load: "+loadFactor);

    if (initialCapacity==0)
        initialCapacity = 1;
    this.loadFactor = loadFactor;
    //new一个指定容量的Hashtable
    table = new Entry<?,?>[initialCapacity];
    //阈值threshold=容量*负载因子
    threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
}

//包含指定Map的构造函数
public Hashtable(Map<? extends K, ? extends V> t) {
    this(Math.max(2*t.size(), 11), 0.75f);
    putAll(t);
}
```

这里的Hashtable容量和HashMap的容量就有区别，Hashtable并不要求容量是2的幂次方，而HashMap要求容量是2的幂次方。负载因子则默认都是0.75。

#### put方法

`put`方法是**同步**的，即线程安全的，这点和`HashMap`不一样，还有具体的`put`操作和`HashMap`也存在很大的差别，Hashtable插入的时候是插入到**链表头部**，而HashMap是插入到**链表尾部**。

```
//synchronized同步锁，所以Hashtable是线程安全的
public synchronized V put(K key, V value) {
    // Make sure the value is not null
    //如果值value为空，则抛出异常 至于为什么官方不允许为空，下面给出分析
    if (value == null) {
        throw new NullPointerException();
    }

    // Makes sure the key is not already in the hashtable.
    Entry<?,?> tab[] = table;
    //直接取key的hashCode()作为哈希地址，这与HashMap的取hashCode()之后再进行hash()的结果作为哈希地址 不一样
    int hash = key.hashCode();
    //数组下标=(哈希地址 & 0x7FFFFFFF) % Hashtable容量，这与HashMap的数组下标=哈希地址 & (HashMap容量-1)计算数组下标方式不一样，前者是取模运算，后者是位于运算，这也就是为什么HashMap的容量要是2的幂次方的原因，效率上后者的效率更高。
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> entry = (Entry<K,V>)tab[index];
    //遍历Entry链表，如果链表中存在key、哈希地址相同的节点，则将值更新，返回旧值
    for(; entry != null ; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            entry.value = value;
            return old;
        }
    }
    //如果为新的节点，则调用addEntry()方法添加新的节点
    addEntry(hash, key, value, index);
    //插入成功返回null
    return null;
}

private void addEntry(int hash, K key, V value, int index) {
    modCount++;

    Entry<?,?> tab[] = table;
    //如果当前键值对数量>=阈值，则执行rehash()方法扩容Hashtable的容量
    if (count >= threshold) {
        // Rehash the table if the threshold is exceeded
        rehash();

        tab = table;
        //获取key的hashCode();
        hash = key.hashCode();
        //重新计算下标，因为Hashtable已经扩容了。
        index = (hash & 0x7FFFFFFF) % tab.length;
    }

    // Creates the new entry.
    @SuppressWarnings("unchecked")
    //获取当前Entry链表的引用 复赋值给e
    Entry<K,V> e = (Entry<K,V>) tab[index];
    //创建新的Entry链表的 将新的节点插入到Entry链表的头部，再指向之前的Entry，即在链表头部插入节点，这个和HashMap在尾部插入不一样。
    tab[index] = new Entry<>(hash, key, value, e);
    count++;
}
```

> hashCode()为什么要& 0x7FFFFFFF呢？因为某些对象的hashCode()可能是负值，& 0x7FFFFFFF保证了进行%运算时候得到的下标是个正数



#### get方法

`get`方法也是同步的，和`HashMap`不一样,即线程安全，具体的`get`操作和`HashMap`也有区别。

```
//同步
public synchronized V get(Object key) {
    Entry<?,?> tab[] = table;
    //和put方法一样 都是直接获取key的hashCode()作为哈希地址
    int hash = key.hashCode();
    //和put方法一样 通过(哈希地址 & 0x7FFFFFFF)与Hashtable容量做%运算 计算出下标
    int index = (hash & 0x7FFFFFFF) % tab.length;
    //遍历Entry链表，如果链表中存在key、哈希地址一样的节点，则找到 返回该节点的值，否者返回null
    for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            return (V)e.value;
        }
    }
    return null;
}
```

#### remove方法

```
//同步
public synchronized V remove(Object key) {
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>)tab[index];
    //遍历Entry链表，e为当前节点，prev为上一个节点
    for(Entry<K,V> prev = null ; e != null ; prev = e, e = e.next) {
        //找到key、哈希地址一样的节点
        if ((e.hash == hash) && e.key.equals(key)) {
            modCount++;
            //如果上一个节点不为空(即不是当前节点头结点)，将上一个节点的next指向当前节点的next，即将当前节点移除链表
            if (prev != null) {
                prev.next = e.next;
            } else { //如果上一个节点为空，即当前节点为头结点，将table数组保存的链表头结点地址改成当前节点的下一个节点
                tab[index] = e.next;
            }
            //Hashtable的键值对数量-1
            count--;
            //获取被删除节点的值 并且返回
            V oldValue = e.value;
            e.value = null;
            return oldValue;
        }
    }
    return null;
}
```



#### rehash方法

Hashtable的`rehash`方法和HashMap的`resize`方法一样，是用来扩容哈希表的，但是扩容的实现又有区别。

```
protected void rehash() {
    //获取旧的Hashtable的容量
    int oldCapacity = table.length;
    //获取旧的Hashtable引用，为旧哈希表
    Entry<?,?>[] oldMap = table;

    // overflow-conscious code
    //新的Hashtable容量=旧的Hashtable容量 * 2 + 1，这里和HashMap的扩容不一样，HashMap是新的Hashtable容量=旧的Hashtable容量 * 2。
    int newCapacity = (oldCapacity << 1) + 1;
    //如果新的Hashtable容量大于允许的最大容量值(Integer的最大值 - 8)
    if (newCapacity - MAX_ARRAY_SIZE > 0) {
        //如果旧的容量等于允许的最大容量值则返回
        if (oldCapacity == MAX_ARRAY_SIZE)
            // Keep running with MAX_ARRAY_SIZE buckets
            return;
        //新的容量等于允许的最大容量值
        newCapacity = MAX_ARRAY_SIZE;
    }
    //new一个新的Hashtable 容量为新的容量
    Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

    modCount++;
    //计算新的阈值
    threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    table = newMap;
    //扩容后迁移Hashtable的Entry链表到正确的下标上
    for (int i = oldCapacity ; i-- > 0 ;) {
        for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
            Entry<K,V> e = old;
            old = old.next;

            int index = (e.hash & 0x7FFFFFFF) % newCapacity;
            e.next = (Entry<K,V>)newMap[index];
            newMap[index] = e;
        }
    }
}
```



###HashMap和Hashtable区别

到这里我们分析了HashMap和Hashtable的原理，现在比较以下他们的区别。

对于想[深入分析java 7或者java 8中的HashMap](./Java7-8下的HashMap与ConcurrentHashMap.md)；

#### 不同点

- **继承的类不一样**：HashMap继承的`AbstractMap`抽象类，Hashtable继承的`Dictionay`抽象类
- **应对多线程处理方式不一样**：HashMap是非线程安全的，Hashtable是线程安全的，所以Hashtable效率比较低
- **定位算法不一样**：HashMap通过`key`的hashCode()进行hash()得到哈希地址，数组下标=哈希地址 & (容量 - 1)，采用的是与运算，所以**容量需要是2的幂次方结果才和取模运算结果一样**。而Hashtable则是：数组下标=(key的hashCode() & 0x7FFFFFFF ) % 容量，采用的取模运算，所以容量没要求
- **键值对规则不一样**：HashMap允许键值为`null`，而Hashtable不允许键值为`null`
- **哈希表扩容算法不一样**：HashMap的容量扩容按照原来的容量*2，而Hashtable的容量扩容按照原来的容量*2+1
- **容量(capacity)默认值不一样**：HashMap的容量默认值为16，而Hashtable的默认值是11
- **put方法实现不一样**：HashMap是将节点插入到链表的尾部，而Hashtable是将节点插入到链表的头部

> 为什么HashMap允许`null`键值呢，而Hashtable不允许`null`键值呢？这里还得先介绍一下什么是`null`，我们知道Java语言中有两种类型，一种是**基本类型**还有一种是**引用类型**，其实还有一种特殊的类型就是`null`类型，它不代表一个对象(Object)也不是一个对象(Object)，然后在HashMap和Hashtable对键的操作中使用到了**Object**类中的`equals`方法，所以如果在Hashtable中置键值为`null`的话就可想而知会报错了，但是为什么HashMap可以呢？因为HashMap采用了特殊的方式，将`null`转为了对象(Object)，具体怎么转的，这里就不深究了。

#### 相同点

- **实现相同的接口**：HashMap和Hashtable均实现了`Map`接口
- **负载因子(loadFactor)默认值一样**：HashMap和Hashtable的负载因子默认都是0.75
- **采用相同的方法处理哈希冲突**：都是采用链地址法即拉链法处理哈希冲突
- **相同哈希地址可能分配到不同的链表，同一个链表内节点的哈希地址不一定相同**：因为HashMap和Hashtable都会扩容，扩容后容量变化了，相同的哈希地址取到的数组下标也就不一样。

