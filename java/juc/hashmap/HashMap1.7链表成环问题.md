
# HashMap环链表问题

## 环链表问题描述

**JDK 1.7 扩容核心代码**
```
    /**
     * 将老的数组转移到新的数组
     * Transfers all entries from current table to newTable.
     */
    void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;//新的数组大小
        for (Entry<K, V> e : table) {//双重循环
            while (null != e) {//加入链表顺序为1->2->3
                Entry<K, V> next = e.next;//next=2
                if (rehash) {//是否计算hash,一般是false
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);//计算数组下标,两种结果,要么在原有位置,要么原有的下标+oldTable.length
                e.next = newTable[i];//= null 或=2
                newTable[i] = e; // =1
                e = next;
            }
        }
    }
```

**问题概述**

HashMap是线程不安全的，多线程同时操作同一个HashMap，在执行扩容操作时会发生多个线程都在执行扩容业务的情况。有时在数据和执行点比较赶巧的情况下甚至会发生链表成环的问题，当然并非是一旦发生多线程扩容就一定成环。

环链表的威胁在于一旦HashMap出现环状链表，就可能引发出CPU占用100%的情况。

## 深入浅出环的形成

### 初始准备

**初始HashMap结构**

![案例图-初始hashMap](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E7%8E%AF%E9%93%BE%E8%A1%A8-%E6%89%A9%E5%AE%B9%E5%89%8D%E6%83%85%E5%86%B5.jpg?raw=true)

如图描述，有这样一个HashMap对象，它的数组长度是16，在下标为1的位置上有一个单链表，链表包含3个结点，其Hash值依次为1、33、17。

假设此时HashMap对象需要扩容，而线程1和线程2都执行到的transfer扩容的方法。

### 扩容开始

**操作步骤01**

![案例图-操作01](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E7%8E%AF%E9%93%BE%E8%A1%A8-%E6%93%8D%E4%BD%9C01.jpg?raw=true)

如图中间虚线左边是线程1，虚线右边是线程2，线程1先执行到"next = e.next"，线程1的时间片结束。线程2得到时间片并执行完毕整个的扩容操作。因为线程1和线程2创建的newTable是不同的对象，其表现大致如上图描述。

线程1的变量e指向实际的hash为1的对象，变量next指向实际的hash为33的对象。

**操作步骤02**

![案例图-操作02](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E7%8E%AF%E9%93%BE%E8%A1%A8-%E6%93%8D%E4%BD%9C02.jpg?raw=true)

步骤01线程2执行结束，线程1被调度起来继续执行，"e.next=newTable[i]"所以hash为1的结点的next区域为空。

**操作步骤03**

![案例图-操作03](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E7%8E%AF%E9%93%BE%E8%A1%A8-%E6%93%8D%E4%BD%9C03.jpg?raw=true)

线程1执行"newTable[i] = e"，此时线程1的哈希表数组下标为1的位置上的对象是hash:1。 

**操作步骤04**

![案例图-操作04](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E7%8E%AF%E9%93%BE%E8%A1%A8%E6%93%8D%E4%BD%9C04.jpg?raw=true)

线程1执行"e = next"，此时线程1的e和next的变量都指向hash:33的结点。

**操作步骤05**

![案例图-操作05](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E7%8E%AF%E9%93%BE%E8%A1%A8%E6%93%8D%E4%BD%9C05.jpg?raw=true)

线程1执行"next = e.next"， next当前指向hash:1的结点。

**操作步骤06**

![案例图-操作06](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E7%8E%AF%E9%93%BE%E8%A1%A8%E6%93%8D%E4%BD%9C06.jpg?raw=true)

线程1执行"e.next=newTable[i]" ，参考步骤05此时hash:33结点的next会指向hash:1结点，再执行"newTable[i] = e"数组第一个元素既为hash:33结点。

**操作步骤07**

![案例图-操作07](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E7%8E%AF%E9%93%BE%E8%A1%A8%E6%93%8D%E4%BD%9C07.jpg?raw=true)

线程1执行"e = next"，此时线程1的e和next的变量都指向hash:1的结点。

**操作步骤08**

![案例图-操作08](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E7%8E%AF%E9%93%BE%E8%A1%A8%E6%93%8D%E4%BD%9C08.jpg?raw=true)

最后一次循环，线程1执行"next = e.next"，此时线程1的变量e指向null。执行"e.next = newTable[i]"后hash:1的结点的next指针区域指向hash:33的结点。此时环形产生。

**操作步骤09**

![案例图-操作09](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E7%8E%AF%E9%93%BE%E8%A1%A8%E6%93%8D%E4%BD%9C09.jpg?raw=true)

最后线程1执行"newTable[i] = e"，此时HashMap哈希表下标为1的位置上存的是hash:1（相当于是hash:1和hash:33的结点交换了位置）。执行"e = next"后变量e和变量next都指向null。线程1扩容结束。

## 结论

JDK1.7的HashMap存在链表成环的隐患，错误的在并发情况下使用HashMap很危险，一旦在线上出现这个问题，就会是一次严重的线上事故。
