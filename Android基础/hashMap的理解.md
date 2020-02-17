#### hashMap的理解

hashMap底层是通过数组和链表组成的， 默认容量是16， 当达到阈值后， 会先申请两倍原来阈值的容量， 并将原来的值复制进去， 这样子会导致，如果存储大容量数据的话， 会导致内存消耗严重。

**JDK8之前的hashMap是完全通过数组 和链表组成的；内部保存了Entry[],**

  **put（存储）的时候是：**

**int hash = hash(key)**

**int index= hash(key) % length;**

**Entry[index] = value;**

**get(取值)的时候是：**

**int hash = hash(key)**

**int index= hash(key) % length;**

**return Entry[index];**

初始化容量是16， 那么我们通过hash(key)%length 取到的值很有可能会相同，底层是通过**扰动函数**来处理，减少hash碰撞几率；

**上面我的们提到Entry  实际上类似于链表结构，同一个key  当put 的时候一直更新的是链表头  之前如果有值  那么直接往这个链表的后面排一位；这样子 我们取得时候直接取得也是链表头；**