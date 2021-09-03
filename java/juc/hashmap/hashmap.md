
# 一、数据结构

![案例图](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84JDK1.8.jpg?raw=true)

HashMap是java的util包中一个高效的查询容器，内部是哈希表的数据结构，哈希表以键值对为基础存储和查询数据，查询效率是O(1)。

**数据结构**
- 数组：基于数组下标随机访问，查询速度快。
- 链表：通过链地址法解决Hash冲突。
- 红黑树：使用红黑树logn(n指的是链上的数量)级别的查询效率，解决链表过长查询效率低的问题。

## 1. 数组
**哈希表数组**
```
transient Node<K,V>[] table;
```
**描述**
HashMap实际存储键值数据的是名为talbe的字段，table是Node结点类型的数组。

## 2. 链表
**哈希表元素的数据结构**
```
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
        // ... ...
}
```
**描述**
每一个结点都有一个hash、key、value和next，结点是构成单链表的基础数据结构。

## 3. 红黑树
**哈希表树结点的数据结构**
```
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
        // ... ...
}
```

**描述**
HashMap在1.8之后引入了红黑树的概念，红黑树是二叉搜索树的一种，常规的二叉搜索树只需要满足结点的值大于做孩子的值小于右孩子的值即可。但是红黑树不仅仅是这样，红黑树需要满足以下的性质。

### 红黑树的性质
（来自算法导论第三版 p174）
1. 每个结点或是红色的，或是黑色的。
2. 根结点是黑色的。
3. 每个叶结点（NIL）是黑色的。
4. 如果一个结点是红色的，则它的两个子结点都是黑色的。
5. 对每个结点，从该结点到其所有后代叶结点的简单路径上，均包含相同数目的黑色结点。

### 为什么使用红黑树？
**二叉搜索树**
传统的二叉搜索树最坏的情况下树的高度等于结点的数目，以至于查询数据达到了O(n)。

**AVL树**
平衡二叉树是一种平衡了树高的结构，使得任意一个结点的左右子树高差值不超过1。虽然AVL树可以弥补传统的二叉搜索树的缺点，但是它又引来了新的问题，当数据量大了的时候维护AVL树性质的自旋操作会很影响插入性能。

**红黑树**
通过红黑节点性质来维护树的结构，通过节点的颜色变换，简单的旋转赋值提高红黑树的插入性能。

红黑树的最长路径不会超过最短路径的两倍，此时全黑10 + 黑10红9，不会超过两倍。**max <= 2\*min**

**数据结构小结**
红黑树的是许多“平衡搜索树”中的一种，它是一种似平衡的状态，它可以保证在最坏的情况下基本动态集合的操作时间复杂度为O(logn)。


# 二、哈希寻址

![案例图](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E5%93%88%E5%B8%8C%E5%AF%BB%E5%9D%80.jpg?raw=true)

**寻址流程**
1. 使用key执行**hash算法**，用key的hashCode方法返回值当作**扰动函数**的基础数据，并计算最终哈希值。
2. 根据返回的哈希值与表长减一的值进行**与**运算hash & (n-1)，得到取模的结果。
3. 把取模的结果当作目标下标进行相应的操作。

**备注**
表达式[hash & (n-1)] 和 [hash % n] 在表长为2的幂次的情况下，得到的结果是一致的，都可以作为取模操作。

## 1. 哈希算法
```
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
hash算法的目的是为了让不同的key哈希的结果更加离散，防止链表过长或红黑树过高。
上述HashMap的算法是基于键的hashCode返回值，参与到扰动函数的计算中，从而得到具体的哈希结果。这里需要注意当key为null的时候，返回的哈希值为0。

## 2. 扰动函数
![案例图](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E5%93%88%E5%B8%8C%E5%AF%BB%E5%9D%80%E6%89%B0%E5%8A%A8%E5%87%BD%E6%95%B0.jpg?raw=true)

所谓的扰动函数是指，为了使得key尽可能的分散，我们需要使用到key的更多特征，而不仅仅是用其哈希值直接进行取模运算。上述扰动函数使用key哈希值和key哈希值无符号右移16位的值进行异或运算，引入了高16位的特征，增大哈希值的离散性。

## 3. 为什么异或更加分散？
![案例图](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E6%89%B0%E5%8A%A8%E5%87%BD%E6%95%B0%E4%BD%BF%E7%94%A8%E5%BC%82%E6%88%96%E7%9A%84%E4%BB%B7%E5%80%BC.jpg?raw=true)

与操作和或操作对位上0和1的概率是不均衡的，而异或是均衡的。哈希值右移16位，正好是32位的一半，高16位和低16位异或，混合了原始哈希值的高低位的特征，以此来加大低位的随机性。


# 三、HashMap的结点操作方法

## 1. 保存结点
![案例图](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E7%9A%84%E4%BF%9D%E5%AD%98%E6%B5%81%E7%A8%8B.png?raw=true)

**put方法会先进行hash，然后调用putVal方法进行实际处理。**
```
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```

**putVal描述**
1. 如果当前表为空或长度为0，重新构建一个默认容量的容器。
2. 如果哈希寻址到的下标的元素为空，创建一个Node放入。
3. 非以上情况
   i. 如果当前key等于待插入元素的key，hash值也相同，替换原值。
   ii. 如果当前结点是treeNode，使用红黑树的putTreeVal方法插入结点。
   iii. 否则就遍历当前链表依次比较key和hash，如果相同替换原值，不同就继续遍历，直到下一个结点是空的就插入。 插入会判断当前是否是第八个结点，是的话且表长等于64，链表就树化。
4. 返回该key的原值，若没有则返回为空。

```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            // 哈希表为空初始化
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            // 索引位置为空，放入当前结点
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                // 键相同值替换，后面判断替换值
                e = p;
            else if (p instanceof TreeNode)
                // 树结点，使用红黑树的插入方法
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                // 遍历链表
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    // 相同的键则跳出循环，判断替换原值
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            // 替换原值
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
```


## 2. 查询结点

**get方法会先hash然后getNode寻找值**

```
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
```

**getNode描述**
1. 根据hash与运算求得数组下标位置，并判断是否有值。
2. 有值的话，则判断查询的key，是否等于first结点的key，等于则返回。
3. 头结点不等于的话，判断是否是红黑树的结点，是红黑树的结点就走红黑树的查询逻辑。
4. 不是红黑树的结点就遍历单链表查询。

```
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            
            // 找hash值对应到数组中的元素下标，判断下标位置是否为空
            (first = tab[(n - 1) & hash]) != null) {
            
            // 判断头结点
            if (first.hash == hash && 
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
            
            // 判断是否是树结点
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                    
                // 遍历单链表
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }

```

## 3. 哈希表扩容
HashMap 的扩容在 put 操作中会触发扩容，主要是这个方法:
```
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        // ... ...
 		// 定义新哈希表
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
        	// 老哈希表内容复制到新的哈希表
            for (int j = 0; j < oldCap; ++j) {
                if ((e = oldTab[j]) != null) {
                        // ... ...
                    else {
                    	// 当前链表拆成两个链表，原位置表loHead，新位置表hiHead。
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
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
                        // 原位置表lohead设置到新表的原位置上。
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        // 新位置表hihead设置到新表的原位置+原表长的位置上。
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
		// ... ...
    }
```

HashMap一次扩容的过程:
1、取当前table的2倍作为新table的大小。
2、根据算出的新table的大小new出一个新的Entry数组来，名为newTab。
3、HashMap中的table指向newTable。
3、轮询原table的每一个位置，将每个位置上连接的Entry，算出在新table上的位置，并以链表形式连接。原table上的所有Entry全部轮询完毕之后，意味着原table上面的所有Entry已经移到了新的table上。

### 扩容案例

扩容前：

![案例图](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E5%93%88%E5%B8%8C%E8%A1%A8%E6%89%A9%E5%AE%B9%E5%89%8D.jpg?raw=true)

扩容后：
![案例图](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E5%93%88%E5%B8%8C%E8%A1%A8%E6%89%A9%E5%AE%B9%E5%90%8E.jpg?raw=true)

### 扩容后数组的定位
![案例图](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E6%89%A9%E5%AE%B9%E5%89%8D%E5%90%8E%E4%BD%8D%E7%BD%AE%E8%A7%84%E5%88%99.png?raw=true)

如图可知，扩容前的Hash值[A - F]都在同一个链表上，扩容后则链表就会拆开[ABC]在4位置，[DEF]在20位置。扩容后原来的链表元素仅可能出现在两个位置，如果链表元素原来在 x 位置上，那么扩容后的两个点位就是 x 和 x + 16的位置上。

故上图原来都在一个链表上，那么扩容就是4和20两个位置。


## 4. put和putIfAbsent区别

**put方法onlyIfAbsent传的是false**
```
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict)

```
**putIfAbsent方法onlyIfAbsent传的是true**
```
    public V putIfAbsent(K key, V value) {
        return putVal(hash(key), key, value, true, true);
    }
```
**putVal的onlyIfAbsent参数是干嘛的**
```
            // 键相同替换原值的逻辑
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                // 如果onlyIfAbsent是false则是替换原值的
                // onlyIfAbsent是true则是不会替换原值的
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }

```

### putIfAbsent小结
onlyIfAbsent参数为false，则新值会替换原值并且返回原值。
onlyIfAbsent参数为true，则原值不变并且返回原值。

# 其他

## 1. 澄清HashMap链表的树化条件

### 条件1:链表已有8个节点，第9个节点插入链表时树化。

#### 8个节点，未树化。
![案例图-未树化](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E9%95%BF64%E5%8D%95%E9%93%BE8%E6%9C%AA%E6%A0%91%E5%8C%96.jpg?raw=true)

#### 9个节点，树化。
![案例图-树化](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E9%95%BF64%E5%8D%95%E4%BD%8D%E7%BD%AE%E7%AC%AC9%E4%B8%AA%E7%BB%93%E7%82%B9%E6%A0%91%E5%8C%96.jpg?raw=true)

#### 插入结点源码解析
当前count值要大于等于 TREEIFY_THRESHOLD(8) - 1，计数器从0开始计数，到7刚好是8个结点。遍历时从第2个节点开始，故当前待插入节点是8 + 1，第九个节点进行树化。

```
            else {
                // 遍历单链表，判断相同，直到找到空值
                for (int binCount = 0; ; ++binCount) {
                // 重点这里e是第2个节点。
                    if ((e = p.next) == null) {
                        // 待插入结点已插入
                        p.next = newNode(hash, key, value, null);
                        // 树化条件1: 当前count值大于等于 8 - 1，当前待插入结点如果是第九个结点（[0-7]为8个，从第二个节点开始故加1，所以待插入是第九个节点）就树化。
                        if (binCount >= TREEIFY_THRESHOLD - 1)
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
```

### 条件2:表的长度大于或等于64

#### 树化方法源码解析
**treeifyBin(tab, hash)方法**
表的长度小于MIN_TREEIFY_CAPACITY(64)，就不进行树化操作，resize扩容即可。反之，表的长度大于或等于64，才可以进行链表树化的操作。
```
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        // 表长度小于 MIN_TREEIFY_CAPACITY 64，此时扩容即可不需要进行树化操作。
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        
        // 表长大于等于64，才需要进行树化操作
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
```

## 2.自定义类型作为HashMap的键，为什么要重写hashCode和equals方法？

equals方法是比较两个对象是否相同的方法，Object基础实现是比较hashCode，也就是对象的地址。
我们自定义类型如果想当作HashMap的键是需要重写equals方法的，否则两个对象的属性值相同，但是却不是同一个对象，地址不相同导致最终结果不相等。如果作为键的对象没有重写equals，这肯定是有问题的。

hashCode方法，是唯一标识一个对象的方法，Object默认实现时返回对象的地址。
**HashCode重写时需要注意以下几点：**
1. hashCode方法中不能包含equals方法中没有的字段。
2. String和其它包装类型已有的hashCode可以直接调用。
3. hash = 31 * hash + (field != null ? field.hashCode : 0)；可以应用于hashCode方法当中。因为任何数 n * 31都可以被JVM优化为 (n<<5)-n 这个表达式，移位和减法要比其它的操作快速的多。

**《Effective Java》中提出了一种简单通用的hashCode算法：**
1. 初始化一个整型的变量，并为此变量富裕一个非零的常数值，如 int code = 13;
2. 如果对象中有String或其它包装类型，则递归调用该属性的hashCode，如果属性为空则处理为0。

**案例**
```
    @Override
    public int hashCode() {
        int hash = 13;
        hash = hash * 31 + (name != null ? name.hashCode() : 0);
        hash = hash * 31 + (location != null ? location.hashCode() : 0);
        return hash;
    }
```

## 3. HashMap和HashTable有哪些不同？


1. 初始容量：HashMap是16，HashTable是11。

2. HashTable是线程安全的，HashMap是线程不安全的。HashTable在读写方法前使用了synchronized同步锁HashMap就没有这些安全机制，多线程环境下使用是有问题的。

3. HashTable没有树化的操作，就仅仅是数组加链表。HashMap由于被到处引用，为了避免Hash冲突导致链表过长的问题，就引入了红黑树树化操作。

### 3.1 为什么HashMap和HashTable容量规则不同？
![案例图-未树化](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E8%BF%9D%E8%83%8C%E9%99%A4%E6%B3%95%E6%95%A3%E5%88%97%E6%B3%95%E8%A7%84%E5%88%99.png?raw=true)
HashMap为了效率违背了算法导论的推荐，所以是有弊端的，HashMap为了弥补这个弊端，就重写了hash算法，加入了高位特征的扰动函数。使得h(k)结果足够分散。

### 3.2 HashMap红黑树可能出现的实际诱因？

**HashMap 1.7 导致的Tomcat的DoS问题**

URL：[http://mail-archives.apache.org/mod_mbox/www-announce/201112.mbox/%3C4EFB9800.5010106@apache.org%3E](http://mail-archives.apache.org/mod_mbox/www-announce/201112.mbox/%3C4EFB9800.5010106@apache.org%3E)

上面链接的页面是来自Tomcat邮件组的讨论。Tomcat参数是用HashMap存储的，故如果参数有5W个，并且有心人构造了hash冲突比较严重的参数，此时链表的长度很长，查询参数就占用了CPU很多资源，就可能出现Dos问题（DoS时CPU100%）。

## 4. HashMap为什么链表超过8个结点会树化？
```
    static final int TREEIFY_THRESHOLD = 8;
```

![案例图-未树化](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E6%B3%A8%E9%87%8A%E6%B3%8A%E6%9D%BE%E5%88%86%E5%B8%83.png?raw=true)

上图是JDK1.8HashMap源码中提供的泊松分布的注释，泊松分布链表中出现8个元素的概率是极低的，所以出现红黑树概率很低。并且树化条件并不只是链表长度超过8，数组长度也要是64及以上才行。