# HashMap
## 1.什么是HashMap
HashMap是java开发中经常使用到的集合，它是`键值对`格式的，顶层接口是`Map`。它的特点是通过key快速找到value。

HashMap是允许key和value为null的。

```java
    Map<String, String> map = new HashMap<>();
// zhangsan -> 张三
    map.put("zhangsan", "张三");
String value = map.get("zhangsan");
// 输出：张三
    System.out.println(value);
```

## 2.HashMap的底层结构
在jd<font style="color:#000000;">k1.8之前，hashMap的底层结构是</font>`数组+链表`，jdk1.8之后就改为了`数组+链表+红黑树`，在后面会详细解释为什么引入红黑树。

数组也叫做table数组，table中的每个位置叫做bucket。当bucket为空时是null，不为空的时候，存储的是桶里的数据结构的入口，也就是头节点。

```java
    /**
 * The table, initialized on first use, and resized as
 * necessary. When allocated, length is always a power of two.
 * (We also tolerate length zero in some operations to allow
 * bootstrapping mechanics that are currently not needed.)
 */
transient Node<K,V>[] table;
```

```java
static class Node<K,V> implements Map.Entry<K,V> {
    // hash值
    final int hash;
    // key
    final K key;
    // value
    V value;
    // 下一个节点
    Node<K,V> next;
}
```



## 3.HashMap中的知识点
```java
    /**
 * HashMap的数组默认大小，16
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

/**
 * 最大容量，左移30位
 */
static final int MAXIMUM_CAPACITY = 1 << 30;

/**
 * 默认负载因子 0.75
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;

/**
 * 最小树化阈值（链表节点大于8，数组容量大于64会树化）
 */
static final int TREEIFY_THRESHOLD = 8;

/**
 * 退化阈值
 */
static final int UNTREEIFY_THRESHOLD = 6;

/**
 * 最小树化阈值
 */
static final int MIN_TREEIFY_CAPACITY = 64;
```

### 3.1负载因子
也就是HashMap的扩容判断条件，比如长度是16。那么阈值的计算如下：16 * 0.75 = 12。元素个数 size 大于 threshold 才会扩容。

### 3.2转红黑树的条件
当数组长度大于等于64，且链表节点大于等于8时，就会转成红黑树。

### 3.3为什么数组的长度是2的幂等

详细参考这一节：[4.2第二步获取数组下标](#4.2第二步获取数组下标)

### 3.4为什么key最好不可变

因为hashMap依赖key的hash值来找元素，如果key发生了变化，可能会导致计算出来的hash值不一致，会找不到指定的元素。所以一般不会用业务对象作为key。

## 4.put流程（重点掌握）
### 4.1第一步计算hash值
往HashMap中放入元素时，会根据key进行hash计算，获取hash值

```java
return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
```

重点看这个`h >>> 16`，也叫做`扰动计算`，目的是为了将hashCode的高位也参与到计算中，减少哈希冲突，使key均匀的分布在数组中。

### 4.2第二步获取数组下标
这个时候获取到的hash值都很大，比如81522，假设数组长度是16，还需要一次计算获取数组下标。

```java
if ((p = tab[i = (n - 1) & hash]) == null) {
tab[i] = newNode(hash, key, value, null);
}
```

重点看`(n - 1) & hash`，n就是数组的长度，减去1，跟hash值进行按位与计算，也就是`hash % n` （前提是数组容量是2的幂），得到的就是0~15的数组下标，这里之所以用按位与不用取模运算是因为按位与算的更快。这里的n-1也决定了为什么hashMap的数组长度一定要是2的幂。

```latex
16：10000
15：01111
n-1后二进制低位全都是1，可以使得结果均匀分布在0~15之间。如果不是2的幂，有些下标可能永远不会被使用上
```

### 4.3放入元素
如果table[index]为空，则直接放入。

如果不为空，说明发生了哈希冲突，hashMap中解决哈希冲突的方式是链地址法，也就是数组的位置可以挂一个链表或者红黑树。例如

```latex
table[1] -> Node("x","你好") -> Node("y", "你也好")。
```

出现冲突后，不是直接添加，而是进行hash值比较和key的equals比较（链表就链表的遍历方式，红黑树则树的遍历方式），发现相同的就覆盖，没有相同的则尾插法放在最后。插入完元素后会校验是否需要扩容（后面会详细讲）。



## 5.get流程（重点掌握）

get()方法，传入一个key，返回value，如下

```java
Map<String, String> map = new HashMap<>();
map.put("key","这是一个值");
String value = map.get("key");
// 输出：这是一个值
System.out.println(value);
```

### 5.1对key进行hash计算

详细步骤参考[4.1第一步计算hash值](#4.1第一步计算hash值)和[4.2第二步获取数组下标](#4.2第二步获取数组下标)

伪代码如下

```java
int hash = hash(key);
int index = (table.length - 1) & hash;

Node<K,V> first = table[index];

if (first == null) {
    return null;
}

// 永远检查第一个节点
if (first.hash == hash &&(first.key == key || key.equals(first.key))) {
    return first.value;
}
// 遍历，寻找key和hash都相等的结果
Node<K,V> node = first.next;
while (node != null) {
    if (node.hash == hash && (node.key == key || key.equals(node.key))) {
        return node.value;
    }
    node = node.next;
}

    return null;
```

## 6.扩容resize()机制

当数组元素size大小大于容量*负载因子的值时，会触发扩容。注意：

JDK8 中，普通新增节点后通常通过 ++size > threshold 判断是否扩容。
JDK7 的 put 流程中，新增前会结合 size、threshold 和目标桶情况判断是否需要扩容。
可以简单理解为：JDK7 更偏向插入前扩容，JDK8 更偏向插入后扩容。

源码如下：

先放前半段，主要就是先创建新数组的容大小（capacity）和阈值(threshold)大小。

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 左移一位，两倍扩容
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1;
    }
    else if (oldThr > 0)
        newCap = oldThr;
    else {
        // 旧数组为空，直接创建一个新的数组，容量大小16，阈值=16 * 0.75（负载因子）
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                (int)ft : Integer.MAX_VALUE);
    }
    // 赋值新的阈值大小
    threshold = newThr;
    //后半段在下面，它们都在这一个方法里，为了清晰这里拆开讲了
}
```

后半段是重点，迁移数据，我把源码贴上来，注释写在源码里。下面有举例子详解

```java

// 根据前面计算出来的容量大小，创建数组
final Node<K,V>[] resize() {
    Node<K, V>[] newTab = (Node<K, V>[]) new Node[newCap];
     // 更新引用
    table = newTab;
    if (oldTab != null) {
        // 遍历每个元素
        for (int j = 0; j < oldCap; ++j) {
            Node<K, V> e;
            // 如果元素不为空
            if ((e = oldTab[j]) != null) {
                // 清空旧数组当前位置的引用，帮助 GC 回收旧数组中的节点引用关系
                oldTab[j] = null;
                // 如果没有下一个节点，说明就一个元素，直接放入新数组
                if (e.next == null)
                    // 计算当前元素在新数组的位置
                    newTab[e.hash & (newCap - 1)] = e;
                // next不为空，判断是否为树，走树的迁移方式
                // 树节点也会按高低位拆分，只是 TreeNode.split 内部还会处理树化/退化逻辑
                else if (e instanceof TreeNode)
                    ((TreeNode<K, V>) e).split(this, newTab, j, oldCap);
                else {
                    // 链表迁移开始 -- 重点
                    Node<K, V> loHead = null, loTail = null;
                    Node<K, V> hiHead = null, hiTail = null;
                    Node<K, V> next;
                    do {
                        next = e.next;
                        // 与旧容量进行按位与计算，如果=0是低位，放在原位置不动
                        // e.hash & oldCap的结果只能【0和oldCap的值】
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                // 为空说明是首次
                                loHead = e;
                            else
                                // 不是首个元素，尾插
                                loTail.next = e;
                            loTail = e;
                        }
                        // 高位
                        else {
                            if (hiTail == null)
                                // 为空说明是首次
                                hiHead = e;
                            else
                                // 不是首个元素，尾插
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 遍历结束，链表根据高低位，拆分为两条链表
                    if (loTail != null) {
                        // 低位链表的最后一个元素置为空
                        loTail.next = null;
                        // 低位直接放在新数组[旧数组下标]，就是扩容后位置不变
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        // 高位链表的最后一个元素置为空
                        hiTail.next = null;
                        // 高位直接放在新数组[旧数组下标 + 旧数组的容量]
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
}
```

举例详解：

假设容量为：16，扩容后为32
16: 00010000
32: 00100000
hashMap计算数组下标的方式：(n-1)&hash。n就是容量大小
15: 00001111
31: 00011111

看低位：分别是01111和11111，如果任意一个hash值，跟这两个值进行按位与计算，结果一定是`一样`或者`相差16`。
源码：`(e.hash & oldCap) == 0`，就是为了判断出，当前这个节点在旧数组下标运算中，是属于高位还是低位的，如果是低位的，一定是=0的，高位的结果就是16。

比如19这个hash值。
19: 00010011
它在原数组的下标位置为：(16-1) & 19= 00000011 = 3
它在新数组的下标位置为：(32-1) & 19= 00010011 = 19
同上，就是多了个16这个大小的差值，也就是旧数组的容量大小
现在看(e.hash & oldCap) == 0的结果：
跟16进行按位与运算结果：(19 & 16) = 16（旧capacity的值）。所以它在新数组的位置就是 capacity + 当前所在数组位置，也就是 16 + 3 = 19



多例子：
假设 oldCap = 16，旧桶下标 j = 3。
有四个节点的 hash 分别是：

3、19、35、51

旧数组下标：
3  & 15 = 3
19 & 15 = 3
35 & 15 = 3
51 & 15 = 3

所以它们原来都在 table[3]。

扩容后判断 hash & oldCap，也就是 hash & 16：

3  & 16 = 0   -> lo，留在 3
19 & 16 = 16  -> hi，去 3 + 16 = 19
35 & 16 = 0   -> lo，留在 3
51 & 16 = 16  -> hi，去 19

所以原链表：

table[3] -> 3 -> 19 -> 35 -> 51

扩容后拆成：

newTable[3]  -> 3 -> 35
newTable[19] -> 19 -> 51

## 7.HashMap为什么快

因为HashMap不是从头到尾遍历元素，而是计算hash值，然后通过hash值找到具体的桶位置，所以它的平均时间复杂度是O(1)。举例：假设你有10000个元素，存放在map中和存放在list中

```java
    class User {
        private Long userId;
        private String userName;
    }

    // 找到id = 123的用户
    Long userId = 123L;
    // map获取（假设已经存入）
    Map<Long, User> userMap = new HashMap<>();
    // 平均复杂度O(1)
    userMap.get(userId);
    
    // list遍历
    List<User> userList = new ArrayList<>();
    // 只能循环单个遍历，时间复杂度是O(n)
    for(User user : userList) {
        if(userId.equals(user.getUserId())) {
        return user;
    }


```

## 8.什么是哈希冲突

当两个不同的key，被映射到同一个桶里时，且两个key进行equals比较不相等，就发生了哈希冲突。如果两个key的equals为true，就会进行元素替换，新value覆盖就value。

这里需要延申着重讲下，为什么一定要重写equals和hashCode方法，几个点如下：

- 如果两个值的equals方法相等，那么hash值也一定相等。如果hash值相等，equals不一定相等。
- 如果只重写了equals方法，那就会出现两个key的hash值不同，放到了不同的桶里。那么执行get的时候，可能就找到别的桶了（复习：找桶是通过hash计算数组下标的）。
- 如果只重写了hashCode方法，就会出现本该被替换的旧值，却因为equals方法不相等，导致新值和旧值放在一起了，没有替换掉。

HashMap解决哈希冲突的方式是`链地址法`，如果当前桶里的数据结构还没树化，就尾插法插在链表后面。如果树化了，则按照红黑树的插入逻辑处理。

## 9.HashMap在实战中的应用

一般用来快速通过一个key获取指定对象

```java
Map<Long, FileRecord> fileRecordMap = new HashMap<>();
FileRecord fileRecord = fileRecordMap.get(1L);
```

通过stream流，把list快速转为Map，注意stream流中collectors.toMap的时候，如果 key 重复，需要指定合并函数，否则会报 `Duplicate key`。所以(v1 , v2) -> v1。表示保持旧的值不覆盖，如果是->v2，就是覆盖旧值

```java

List<FileRecord> fileRecordList = new ArrayList<>();
Map<Long, FileRecord> fileMapNew = fileRecordList.stream().collect(Collectors.toMap(FileRecord::getId, Function.identity(), (v1, v2) -> v1));
```

去重使用
```java
Map<Long, FileRecord> map = new HashMap<>();

for (FileRecord record : fileRecordList) {
        map.put(record.getId(), record);
        }

List<FileRecord> distinctList = new ArrayList<>(map.values());
```

初始化容量--减少扩容

```java
int expectedSize = fileRecordList.size();
// 
Map<Long, FileRecord> map = new HashMap<>((int) (expectedSize / 0.75f) + 1);
```



## 10.HashMap是线程安全的吗

HashMap不是线程安全的，在多线程的情况下写入可能会出现问题。比如put操作，它们可能同时修改数组容量，size大小，节点指针等情况，所以不安全

如果想要线程安全，可以使用ConcurrentHashMap来实现。后续会出一篇关于ConcurrentHashMap的文章。

### 10.1 线程不安全---put覆盖

假设两个线程同时拿到一个空数组，分别put("A","1")和put("B","1")，假设A和B的hash值相同，散列后都放在table[1]这个位置。理想情况应该是：table[1] -> node[A] -> node[B]。实际可能会发生：线程1和线程2都判断了table[1]为空，线程1先放入数据，线程2此时还是拿到table[1]为空的情况，所以直接放入了结果，那么就是：table[1] -> 线程1放入node[A] -> 线程2覆盖node[B]。结果就是：table[1] -> node[B]。A数据丢失了

### 10.2 线程不安全---size计数不准确

还是以上一个为例，假设size = 3。线程1和线程2同时读取到了size=3。size的计数可简化为：

1. 获取size
2. size + 1
3. 回写size

那么问题就可能出现在这里：线程1和线程2同时获取到了size=3。放入两个元素后的size应该是5，线程1和线程2分别计算size=4，最后回写，不管哪个前哪个后，都会覆盖前面的结果，最后结果是4，漏了一次计数。

### 10.3 线程不安全---扩容时的问题

扩容的过程中涉及大量的结构修改和数据计算，如果这个时候出现了多线程同时resize()，最后可能导致：

- 节点丢失
- 链表成环
- 数据指向错误

其中链表回环最为严重，比如jdk7时，使用的时头插法进行扩容。

例：table[1] -> node[A] -> node[B]，经过头插法后会变成：table[1] -> node[B] -> node[A]。也就是链表反转。

假设此时两个线程在扩容，都在迁移这个链表，那么可能会发生：A.next = B，B.next = A。就成了循环。到时候一旦get到这个元素，就会死循环。

jdk8改为尾插法，保证了链表的顺序不反转，而且使用了高低位拆分链表的方式迁移，降低了回环的风险。

注：但 jdk8的HashMap 仍然不是线程安全的，仍可能出现数据覆盖、size 不准确、扩容期间数据丢失或结构异常。

## 11.HashMap的常见面试题

### 11.1.HashMap是如何解决哈希冲突的

如果出现了哈希冲突，HashMap的做法是把元素通过链表的方式串起来，如果链表的长度达到了8，且容量大于等于64，就会把链表树话，转为红黑树。

### 11.2jdk7和8中的hashMap有哪些不同

| 比较点 | jdk7          | jdk8   | 优化点 |
| :---| :-------------: | :------: | :---: |
| 表结构 | 数组+链表 |数组+链表+红黑树|如果是Jdk7的情况下，元素很多的时候，链表会特别长，遍历就很慢。jdk8转为红黑树就是提高查速度|
| 链表插入方式 | 头插 |尾插|头插法在并发resize下，容易形成链表环，尾插降低了这个风险|
| 扩容方式 | 暴力迁移 |高低位拆分迁移|速度速度更快，而且迁移更安全，多线程下降低了结构出错的风险|
| 扰动计算方式 | 复杂 |(h = key.hashCode()) ^ (h >>> 16)|更简单，且让高位也参与了运算，使得数组下标分配更平均|
| 查找速度 | 链表O(n) |红黑树 O(log n)|冲突严重时性能下降|

​			













