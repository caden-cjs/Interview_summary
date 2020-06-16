# 关于hashMap的复习

![img](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/26341ef9fe5caf66ba0b7c40bba264a5_720w.png)

![img](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/8db4a3bdfb238da1a1c4431d2b6e075c_720w.png)



## 源码

### 构造器

```java
 /**
     * Constructs an empty <tt>HashMap</tt> with the default initial capacity
     * (16) and the default load factor (0.75).
     默认拥有16个容量和0.75的负载因子
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```

### put方法

![img](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/58e67eae921e4b431782c07444af824e_720w.png)

#### put

```java
   /**
     * Associates the specified value with the specified key in this map.
     * If the map previously contained a mapping for the key, the old
     * value is replaced.
     通过hash此映射中的指定键指定的值。 如果映射以前包含该键的映射关系，则替换旧值。
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
     *         (A <tt>null</tt> return can also indicate that the map
     *         previously associated <tt>null</tt> with <tt>key</tt>.)
     */ 
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```

![img](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/8e8203c1b51be6446cda4026eaaccf19_720w.png)

#### hash

```java
 static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
/**
hash实际上是三步的,取key的hashCode值、高位运算、取模运算。
// h = key.hashCode() 为第一步 取hashCode值
// h ^ (h >>> 16)  为第二步 高位参与运算
*/
```

#### putval

```java
/**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     该表，初始化的第一次使用，并调整是必要的。 当分配，长度始终是二的幂。 （我们在某些操作容忍长度为零，允许自举当前不需要力学。）
     */
    transient Node<K,V>[] table;
/**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;//定义几个后面使用的值
        //tab:当前的map数据对象
        //p:
        //n:当前hash表的最大长度(hashmap分成hash表和数据链表)
        // 步骤①：tab为空则创建一个table
        if ((tab = table) == null || (n = tab.length) == 0)//第一次使用的时候,设置n的值
            n = (tab = resize()).length;//对表大小执行resize(),并将数量设置到n
        // 步骤②：计算index，并对null做处理 ,上面的取模运算
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            // 步骤③：节点key存在，直接覆盖value
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // 步骤④: 判断当前Node是否是红黑树,是的话直接将数据插入红黑树
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
             //步骤⑤:若不是则是链表
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        //步骤⑥:判断当前链表的长度是否超过了8,如果超过,则将当前链转换成红黑树
                        break;
                    }
                    //步骤⑦:如果当前key已经存在(通过equls判断),那么则直接覆盖
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //对容量++,并且当容量超过当前容量*负载因子时,执行resize
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

#### put的小总结

> 1. 判断table是否为空
>    1. 是：resize（）
> 2. 则根据key调用key的hashcode()方法，并且通过得到的hash值左移16位和当前数字做移位运算后**(hashcode>>16 ^ hashcode) & 当前的最大长度-1(得到一个除了最高位其他为全为1的二进制数)**相对于%2取模运算 效率要高!,通过前面的计算得到了**索引值i**
> 3. 判断table[i]的对象是否为null
>    1. 是:直接将当前的key和value插入到table[i]
>    2. 否:判断当前key是否存在**链表table[i]**中
>       1. 是：直接覆盖之前的值
>       2. 否：判断当前的Node是否是TreeNode？
>          1. 是：直接将Key和Value插入到TreeNode中
>          2. 否：开始遍历链表
>             1. 链表长度是否大于>8
>                1. 是：将链表转换成红黑树，并且插入键值对
>                2. 否：如果对应的key存在，则覆盖，否则插入到链表末尾。
> 4. 插入值之后，会判断当前的容量是否已经大于**(最大容量*负载因子)**，大于则扩容，否则就结束。



### resize方法





https://zhuanlan.zhihu.com/p/21673805