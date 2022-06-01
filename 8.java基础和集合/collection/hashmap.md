https://www.bilibili.com/video/BV1x741117jq?p=4

# HashMap和ConcurrentHashMap深入解析源码合集，看完吊打面试官！|图灵周瑜

![image-20220106164245049](C:\Users\打倒芭蕉\AppData\Roaming\Typora\typora-user-images\image-20220106164245049.png)无p2

## P1总结：



1.    jdk1.7的hashmap的数据结构： 数组+链表，链表是为了解决hash冲突，也就是我们常说的拉链式
2. 链表的插入：jdk1.7采用头插法， jdk1.8采用了尾插法。头插法与尾插法的效率差不多，不存在老师讲的遍历问题。因为在hash覆盖还是插入，必行需要进行遍历equal判断。头插并不能跳过遍历。
3. hash扰动算法：hashcode()并不是Entry中的hash，而是经过hash()扰动后的值，加入高位信息，减少碰撞。
4. 为什么数组大小为2的整数次幂， 通过hash值在特定size中平均映射，采用取模思想，为了提高效率，当n为2的整数次幂时，(hash & n-1)等效于(hash % n)，效率更高。
5. key为null的时候，hash为0，放在table【0】





## P2小结：

P2依然是JDK1.7的内容，在 1:32:40 之前讲解的是 HashMap,之后讲解的是ConcurrentHashMAP.

### HashMap:

1. 扩容。当size>cap*loadfactor的时候，开启扩容，数组变为原来2倍。所以，扩容后，元素要不在原数组index上，要不在index+old_cap。
2. 扩容头插法导致链表反转，多线程下，会形成循环列表，导致在put或get时死循环。
3. modCount。与fail-fast，也就是快速失败有关。当利用iterator迭代器进行循环时，若同时进行map.remove操作，会导致ConcurrentModificationException。主要是在获取迭代器时，会记录modecount(expectedmodCount)，在进行迭代时，会判断二者是否相等(也就是modCount在此期间是否发生改变)，若不相等则抛出异常。所以，iterator.remove则时更新自身expectedModCount来避免这个问题。

### 1.7ConcurrentHashMap:

1. 分段锁解决并发问题。Segment【】，其中每个segment内拥有自身的HashEntry【】。默认参数segment【16】,  loadfactor:0.75, cap为16。HashEntry【n】  n=cap/segmentCount，由于有最小值限制，最小为2，所以初始化的每个segment有2个HashEntry.
2. 构造函数，初始化了segment【】数组，以及segment【0】中的内容(cap, loadfactor, HashEntry【】)作为原型，1-15在使用时再初始化。
3. 扩容，segment【】数组的大小初始化后不再改变大小，所以扩容是扩segment内的HashEntry【】。
4. UNSAFE：CAS思想，原子操作，CompareAndSet(name,oldValue, newValue);

  

## p3小结


JDK1.7的ConcurrentHashMap。假设segment的length为2^sn，内部HashEntry的length为2^hn。

1. put方法。Segment的Sindex是通过hash的高sn位  &(segment.length-1)获得，HashEntry的Hindex是通过hash的低hn位  &(hashEntry.length-1)获得。防止部分空间未被利用，提高数组利用率。为什么不都是低位？如果segment的长度为16， hashEntry的长度为2，那么落在segment(0)的只会落在hashEntry(0)，而不会落在segment(1)。

   

2. 之前提到过初始化只会初始化segment(0)，其他的都是在put的时候初始化的，所以进入put方法的第一件事情是判断是否存在，不存在的话由于线程安全的考虑，需要使用 自旋+CAS的方式初始化HashEntry数组。

   

3. segment.put方法。自旋获取segment锁，此期间判断该node是否存在，不存在会完成node的新建（这部分用了很长时间讲解源码）。获取锁后，判断是否存在相同key的node存在就覆盖，不存在就插入。插入后，判断是否大于阈值，大于则需要扩容。

   

4. 扩容。采用头插法扩容，由于已经在put的时候拿到了锁，所以不会出现之前HashMap扩容的线程安全的问题。这部分的部分源码也讲了很长时间，特别的地方就是需要尾部部分node的落在相同的扩容后的数组位置上的话，会进行整体转移，而非一个一个转移。不用过分care

   。

5. get。获得Segment位置，再获得HashEntry位置，再遍历链表。其中都用到了UNSAFE，读取内核空间，而非工作空间。

   

6. size。先在不加锁的情况下，A累加所有segment的count获得size，和modCount获得sum。B重复上面操作，如果两次统计出来的sum相同，则返回size，表示统计期间size没有发生改动，modCount具有只增不减的设计规范。若不相同，则再C进行一次统计，若C与B 统计出来的sum相同，则返回size。若不相同，则将每个segment加锁后再统计size，这样必然可信。

     

## p4小结

P4小结：
本节只讲了红黑树的插入原理，没有讲JDK1.8的HashMap和ConcurrentHashMap，所以已经掌握红黑树的可以跳过。如果只是想大概了解下红黑树，可以观看；如果想全面掌握红黑树，建议看其他视频。
\1. 红黑树的性质：
1）. 节点是红色或者是黑色
2）. 根节点是黑色
3）. 叶子节点都是黑色
-------重点---------
4）. 每个红节点的两个子节点都是黑色（注定不可能有两个相连的红色父子节点）
5）. 从任意一个节点出发，到每个所能到达的叶子节点的路径上的黑节点的数目是相同的。

![image-20220107140257700](C:\Users\打倒芭蕉\AppData\Roaming\Typora\typora-user-images\image-20220107140257700.png)\-------------------

\2. 红黑树插入的规律：
1). 由于1.5的特性，为避免影响已有特征，所以插入的节点都是红节点
2). 当插入节点的父节点  是黑色时，插入无需调整
3). 当插入节点的父节点是红色时，
  a. 叔叔节点是红色时，将父亲和叔叔节点改为黑色，祖父节点改为红色
  b. 叔叔节点是空或者黑色时，需要旋转且变色。

祖父节点-父节点(左)-节点(左)，需右旋，祖父变红，父亲变黑。
祖父节点-父节点(左)-节点(右)，先左旋后右旋，父亲祖父变儿子，祖父变红，自己变黑。
祖父节点-父节点(右)-节点(右)，需左旋，祖父变红，父亲变黑。
祖父节点-父节点(右)-节点(左)，先右旋后左旋，父亲祖父变儿子，祖父变红，自己变黑。

源代码

![image-20220107175830796](C:\Users\打倒芭蕉\AppData\Roaming\Typora\typora-user-images\image-20220107175830796.png)

## p5小结

JDK1.8的HashMap底层的数据结构是：数组+链表(Node)/红黑树(TreeNode)，其中需要注意，TreeNode中既有红黑树属性，也是一个双向链表。
\1. 构造方法。构造方法只会初始化一些属性，cap，loadfactor.并不会初始化数组。
\2.  put方法。当第一次调用put的时候会利用resize方法初始化数组tab。利用(hash&length-1)获得数组下标index，判断元素是否存在，存在则更新，不存在则添加。如果是红黑树，则遍历红黑树查找，不存在插入该元素，并重新平衡红黑树。如果是链表，则遍历链表查找，不存在则尾插元素，判断插入后是否需树化，树化条件：①该链表元素插入后大于8，②数组长度达到64，若不满足②，则只扩容。插入后，modCount++，size++，判断size是否超过阈值，超过则扩容。
\3. 树化。A:将Node转化为TreeNode形成双向链表，B:将双向链表从头结点开始平衡插入红黑树，C:将红黑树的root节点切换为双向链表的头结点。
\4. 红黑树的比较。A :hash比较； B: 如果为comparable则使用compareTo； C: className比较，D: 系统hash
\3. resize方法。 A: 进行数组初始化；B: 扩容。若为链表，则遍历链表，利用  (hash&oldcap)，1为高桶table【index+oldCap】，0为低桶table【index】。若为红黑树，同样遍历双向链表，统计高低桶扩容后的个数，如果分割后的红黑树节点个数不大于6，则退化为链表（双向链表TreeNode转化为单向Node）。否则重新平衡插入红黑树形成高低桶的红黑树。

链表 转 红黑树：该链表元素插入后大于8，且，数组长度达到64
红黑树退化为链表： ①扩容后，红黑树上元素个数不超过6个 ②remove元素



## JDK1.8-ConcurrentHashMapp6小结

\1. 构造方法。构造方法只会初始化一些属性,并不会初始化数组。

\2. sizeCtl是初始化和扩容的并发控制的锁。<0代表有线程，正在初始化或者扩容。-1，代表初始化， -(1+ resize thread nums)，代表扩容中。

\3.  1.8中红黑树节点为TreeBin，其hash值为-2，对TreeNode进行了一层封装，为什么需要封装，在put的时候会对桶的第一个元素进行加锁，如果使用HashMap的TreeNode方式，在重新平衡后，root节点会发生变化，导致加锁对象的不一致。而使用TreeBin整个对象初始化后不再改变，改变的是其内部的TreeNode树结构。

\3. put  方法。首次put，需要初始化table。若所在桶为空，则自旋+CAS，添加Node。若Node的hash值为-1，代表正在扩容，会参与帮助扩容。若存在头结点f，则synchronized(f)，fh大于等于0，则代表为链表，遍历，存在更新，不存在尾插；若为红黑树，则遍历，存在更新，不存在，获得TreeBin写锁，balanceInsertion。链表插入后，需要判断是否需要树化，元素>8，且桶数大于等于64，则给链表头结点加锁，然后变为TreeNode双向链表，然后new TreeBin，TreeBin内部将双向链表平衡插入构造成红黑树。

\4. addCount。
若counterCell为空，则尝试CAS改变baseCount。若修改失败，则自旋，尝试获得CELLSBUSY锁，获得则初始化counterCell数组并根据自身线程初始化其中的桶数据；如果没有获得CELLBUSY锁，则CAS修改baseCount。
若counterCell不为空，则查看自身线程所对应的桶是否为空。为空，则自旋获得CELLSBUSY锁来初始化该桶数值。
若counterCell不为空，且自身线程所对应的桶也不为空，则尝试CAS修改该桶数据，尝试1次失败后，则rehash尝试修改重新rehash到的桶的数据，最多再rehash1次后，则自旋尝试获得CELLSBUSY锁，来扩容counterCells数组。如果rehash到的桶为空桶，则重新获得rehash到非空桶的机会1次。

\5. size。就是baseSize + 所有counterCell的值。

JDK1.8-ConcurrentHashMap，无论是1.7还是1.8都不支持null的key和value



## p7小结

扩容。JDK1.8的ConcurrentHashMap是支持多线程并发扩容的。JDK1.7只支持不同segment的并发扩容，同一个segment的扩容不能并发进行。
\1.  sizeCtl变量。扩容中，非常关键的标志位sizeCtl，之前在initTable的时候介绍过，当其为-1的时候代表正在初始化table，当首个线程触发了扩容，会根据当前table的长度n生成一个负数f(n)，所以当table的长度n确定了这个负数也是固定的，利用CAS更新sizeCtl=f(n)。helpTransfer: 当其他线程加入协助扩容之前，会CAS去修改f(n)+1。然后执行Transfer方法。
\2.  Transfer方法。先确定一个步长stride(最小16)。如果nextTab为空，则初始化一个。根据步长进行转移，从右往左，获取桶的首节点，如果为空，则直接设置为fwd，继续向左前进。不为空synchronized加锁，如果是链表，则寻找最后一段hash&n不再发生改变的，整体转移，剩下的再通过遍历，根据hash&n的结果进行逐个头插转移（类似于1.7的）。更新nextTab。如果是TreeBin，则通过TreeBin的双向链表的性质进行遍历，根据hash&n的结果，分为high与low，然后分别判断下high与low是否需要退化为链表（node个数不大于7），如果需要，则将TreeBin改为Node，根据TreeBin内部的TreeNode的双向链表遍历生成新的单向链表。如果不需要，则重新new  TreeBin进行balanceInsertion，更新nextTab。更新完毕后，将原来的table位置上设置为fwd，hash为MOVED。各个扩容线程依次循环，直到自身没有任务，则CAS更新sizeCtl-1，如果不为f(n)，说明还有别的线程在进行扩容，则直接返回。若为f(n)，则表示我是仅剩的一个扩容线程，需要去将map的table更新为nextTab，并设置sizeCtl = (n << 1) - (n >>> 1);（这就是为什么要0.75，仅需要位操作与加减操作，效率高）
\3. fwd节点的作用就是，当put的时候得到某个节点为fwd，就协助扩容。协助扩容结束后，返回的是nextTab。
\4. 扩容结束后，会再统计一次size，然后判断是否需要扩容。