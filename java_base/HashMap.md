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
在jd<font style="color:#000000;">k1.8之前，hashMap的底层结构是</font>`数组+链表`，jdk1.8之后就改为了`数组+链表+红黑树`<font style="color:#000000;">，在后面会详细解释为什么引入红黑树。</font>

<font style="color:#000000;">数组也叫做table数组 </font>

```java
    /**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     */
    transient Node<K,V>[] table;
```

每个数组里放的可能是null，链表，红黑树，结构如下：

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



## 3.HashMap中的细节
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
也就是HashMap的扩容判断条件，比如长度是16。那么阈值的计算如下：16 * 0.75 = 12。那么数组长度超过阈值就会扩容

### 3.2转红黑树的条件
当数组长度大于等于64，且链表节点大于等于8时，就会转成红黑树。

### 3.3为什么数组的长度是2的幂等

详细参考这一节：[4.2第二步获取数组下标](#4.2第二步获取数组下标)

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
if (first.hash == hash && 
    (first.key == key || key.equals(first.key))) {
    return first.value;
}
// 遍历，寻找key和hash都相等的结果
Node<K,V> node = first.next;
while (node != null) {
    if (node.hash == hash &&
        (node.key == key || key.equals(node.key))) {
        return node.value;
    }
    node = node.next;
}

return null;
```

## 6.HashMap为什么快

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
}

```

## 7.什么是哈希冲突

当两个不同的key，被映射到同一个桶里时，且两个key进行equals比较不相等，就发生了哈希冲突。如果两个key的equals为true，就会进行元素替换，新value覆盖就value。

这里需要延申着重讲下，为什么一定要重写equals和hashCode方法，几个点如下：

- 如果两个值的equals方法相等，那么hash值也一定相等。如果hash值相等，equals不一定相等。
- 如果只重写了equals方法，那就会出现两个key的hash值不同，放到了不同的桶里。那么执行get的时候，可能就找到别的桶了（复习：找桶是通过hash计算数组下标的）。
- 如果只重写了hashCode方法，就会出现本该被替换的旧值，却因为equals方法不相等，导致新值和旧值放在一起了，没有替换掉。

HashMap解决哈希冲突的方式是`链地址法`，如果当前桶里的数据结构还没树化，就尾插法插在链表后面。如果树化了，就根据大小放在合适的位置。

## 8.HashMap在实战中的应用

一般用来快速通过一个key获取指定对象

```java
Map<Long, FileRecord> fileRecordMap = new HashMap<>();
FileRecord fileRecord = fileRecordMap.get(1L);
```

## 9.HashMap是线程安全的吗

HashMap不是线程安全你的，在多线程的情况下写入可能会出现问题。比如put操作，它们可能同时修改数组容量，size大小，节点指针等情况，所以不安全

如果想要线程安全，可以使用ConcurrentHashMap来实现。后续会出一篇关于ConcurrentHashMap的文章。

## 10.HashMap的常见面试题

### 10.1.HashMap是如何解决哈希冲突的

如果出现了哈希冲突，HashMap的做法是把元素通过链表的方式串起来，如果链表的长度达到了8，且容量超过了64，就会把链表树话，转为红黑树。

### 10.2jdk7和8中的hashMap有哪些不同

| 比较点 | jdk7          | jdk8   | 优化点 |
| :---| :-------------: | :------: | :---: |
| 表结构 | 数组+链表 |数组+链表+红黑树|如果是Jdk7的情况下，元素很多的时候，链表会特别长，遍历就很慢。jdk8转为红黑树就是提高查询速读|
| 链表插入方式 | 头插 |尾插|头插法在并发resize下，容易形成链表环，尾插降低了这个风险|
















