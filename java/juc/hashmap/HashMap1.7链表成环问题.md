
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

![案例图-未树化](https://github.com/muzi-code/image-collection/blob/main/java/juc/hashmap/HashMap%E8%BF%9D%E8%83%8C%E9%99%A4%E6%B3%95%E6%95%A3%E5%88%97%E6%B3%95%E8%A7%84%E5%88%99.png?raw=true)




