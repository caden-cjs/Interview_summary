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

    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }

    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

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

> 扩容机制,实际上**hashmap**中Node是链表 理论上存储空间是无限的,但是为了权衡查找元素的效率，我们需要一种机制，这就是扩容机制
>
> 默认情况下 初始的**hashmap**最大容量是 2^4=16,并且有一个负载因子默认是0.75,都是可以调整的,并且map的容量总是2^n这样的.
>
> 通过前面可知**map**中是先通过hash确认元素存放的索引位置,之后再操作链表,那么**map**实际上的瓶颈就是在链表的长度上
>
> 所以map在权衡之后引入了**resize**扩容机制。
>
> ![img](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/e5aa99e811d1814e010afa7779b759d4_720w.png)

#### 1.7版本

```java
//1.7版本
  1 void resize(int newCapacity) {   //传入新的容量
 2     Entry[] oldTable = table;    //引用扩容前的Entry数组
 3     int oldCapacity = oldTable.length;         
 4     if (oldCapacity == MAXIMUM_CAPACITY) {  //扩容前的数组大小如果已经达到最大(2^30)了
 5         threshold = Integer.MAX_VALUE; //修改阈值为int的最大值(2^31-1)，这样以后就不会扩容了
 6         return ;
 7     }
 8  
 9     Entry[] newTable = new Entry[newCapacity];  //初始化一个新的Entry数组
10     transfer(newTable);                         //！！将数据转移到新的Entry数组里
11     table = newTable;                           //HashMap的table属性引用新的Entry数组
12     threshold = (int)(newCapacity * loadFactor);//修改阈值
13 }
//先用新的数组替换掉老的数组,再通过transfer方法将数据拷贝到新的数组中

1 void transfer(Entry[] newTable) {
 2     Entry[] src = table;                   //src引用了旧的Entry数组
 3     int newCapacity = newTable.length;
 4     for (int j = 0; j < src.length; j++) { //遍历旧的Entry数组
 5         Entry<K,V> e = src[j];             //取得旧Entry数组的每个元素
 6         if (e != null) {
 7             src[j] = null;//释放旧Entry数组的对象引用（for循环后，旧的Entry数组不再引用任何对象）
 8             do {
 9                 Entry<K,V> next = e.next;
10                 int i = indexFor(e.hash, newCapacity); //！！重新计算每个元素在数组中的位置
11                 e.next = newTable[i]; //标记[1]
12                 newTable[i] = e;      //将元素放在数组上
13                 e = next;             //访问下一个Entry链上的元素
14             } while (e != null);
15         }
16     }
17 }
```

#### 1.8版本

```java
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;//获取当前的数组
        int oldCap = (oldTab == null) ? 0 : oldTab.length;//获取当前数组长度
        int oldThr = threshold;//获取当前的阈值
        int newCap, newThr = 0;//设置新的值用来存放resize之后的数组长度和扩容阈值
        if (oldCap > 0) {//1:判断当前数组是否大于0
            if (oldCap >= MAXIMUM_CAPACITY) {
                //1.1:判断当前数组长度是否大于2^30,如果大于,则不再扩容
                threshold = Integer.MAX_VALUE;//并且将阈值设置成2^32-1
                return oldTab;//不扩容
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)//如果新的容量小于2<30 并且旧的数组长度大于等于16
                newThr = oldThr << 1; // double threshold,就会对旧的长度*2
        }
        else if (oldThr > 0) // 当旧的数组长度=0的时候,那么证明初始化map时设置了长度
            newCap = oldThr;//所以需要将计算好的oldthr设置成新的数组长度
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;//相当于使用了空参构造器,设置默认的数组长度;16
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);//设置默认的阈值 16*0.75=12
        }
        if (newThr == 0) {//就是说oldthr>0的时候且oldCap=0(就是说初始化resize的时候,且设置了长度),那么newThr到目前为止都没有被扩容过,则使用我们初始化时设置的负载因子
            float ft = (float)newCap * loadFactor;//计算当前的阈值
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);//判断,当前的数组容量小于2^30并且新计算的阈值也小于2^30时,则使用新计算的阈值,否则使用2^32-1
        }
        threshold = newThr;//将新得到的阈值复制给类变量threshold
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;//创建一个按照新计算出的长度的数组并且赋值到成员变量table中
        if (oldTab != null) {//如果旧的table不是空的,就是说不是初始化resize时,会遍历整个map对所有数据重新计算索引位置 并放到新的map中
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {//判断当前索引位置链表不为空则重新计算位置并copy
                    oldTab[j] = null;
                    if (e.next == null)//这个index只有一个数据,计算好新的索引位置 copy
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)//如果这个node是红黑树,TODO:后面详解
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order 否则,则该节点是一个长度不超过8的链表,并且存在值
                        Node<K,V> loHead = null, loTail = null;//存放索引不变的节点
                        Node<K,V> hiHead = null, hiTail = null;//存放索引改变的节点
                        Node<K,V> next;
                        do {
                            next = e.next;//通过next遍历链表
                            if ((e.hash & oldCap) == 0) {// 实际上是计算了高位,若新增的位置和当前值与运算得到的是=,则表示索引位置没变
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
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
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

![img](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/a285d9b2da279a18b052fe5eed69afe9_720w.png)

![img](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/b2cb057773e3d67976c535d6ef547d51_720w.png)

### 线程安全性

```java
public class HashMapInfiniteLoop {  

    private static HashMap<Integer,String> map = new HashMap<Integer,String>(2，0.75f);  
    public static void main(String[] args) {  
        map.put(5， "C");  

        new Thread("Thread1") {  
            public void run() {  
                map.put(7, "B");  
                System.out.println(map);  
            };  
        }.start();  
        new Thread("Thread2") {  
            public void run() {  
                map.put(3, "A);  
                System.out.println(map);  
            };  
        }.start();        
    }  
}
```



![img](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/fa10635a66de637fe3cbd894882ff0c7_720w.png)

> 如果在jdk7的环境下,
>
> 其中，map初始化为一个长度为2的数组，loadFactor=0.75，threshold=2*0.75=1，也就是说当put第二个key的时候，map就需要进行resize。
>
> 我们可以看之前的transfer方法
>
> 通过设置断点让线程1和线程2同时debug到transfer方法(3.3小节代码块)的首行。注意此时两个线程已经成功添加数据。放开thread1的断点至transfer方法的“Entry next = e.next;” 这一行；然后放开线程2的的断点，让线程2进行resize。结果如下图。

![img](https://pic4.zhimg.com/80/fa10635a66de637fe3cbd894882ff0c7_720w.png)

> 注意，Thread1的 e 指向了key(3)，而next指向了key(7)，其在线程二rehash后，指向了线程二重组后的链表。
>
> 线程一被调度回来执行，先是执行 newTalbe[i] = e， 然后是e = next，导致了e指向了key(7)，而下一次循环的next = e.next导致了next指向了key(3)。

![img](https://pic2.zhimg.com/80/5f3cf5300f041c771a736b40590fd7b1_720w.png)

> 于是，当我们用线程一调用map.get(11)时，悲剧就出现了——Infinite Loop。

## 小总结

> (1) 扩容是一个特别耗性能的操作，所以当程序员在使用HashMap的时候，估算map的大小，初始化的时候给一个大致的数值，避免map进行频繁的扩容。
>
> (2) 负载因子是可以修改的，也可以大于1，但是建议不要轻易修改，除非情况非常特殊。
>
> (3) HashMap是线程不安全的，不要在并发的环境中同时操作HashMap，建议使用ConcurrentHashMap。
>
> (4) JDK1.8引入红黑树大程度优化了HashMap的性能。
>
> (5) 还没升级JDK1.8的，现在开始升级吧。HashMap的性能提升仅仅是JDK1.8的冰山一角。

https://zhuanlan.zhihu.com/p/21673805