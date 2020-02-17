LRUCache算法解读：

1.LRUCache底层数据结构是LinkedHashMap来保存数据。

LinkedHashMap有个构造方法如下:

```java
public LinkedHashMap(int initialCapacity,
    loat loadFactor,
    boolean accessOrder) {
    super(initialCapacity, loadFactor);
    //是否要按照访问排序
    this.accessOrder = accessOrder;
}
```

当 accessOrder 为 true 时，这个集合的元素顺序就会是访问顺序，也就是访问了之后就会将这个元素放到集合的最后面。

```java
LinkedHashMap < Integer, Integer > map = new LinkedHashMap < > (0, 0.75f, true);
map.put(0, 0);
map.put(1, 1);
map.put(2, 2);
map.put(3, 3);
map.get(1);
map.get(2);

for (Map.Entry < Integer, Integer > entry: map.entrySet()) {
    System.out.println(entry.getKey() + ":" + entry.getValue());

}
```

打印结果：

```
0:0
3:3
1:1
2:2
```

LRUCache 主要方法有两个put（插入元素）和get（读取元素）

①put()方法重点在于trimToSize()方法， 这个方法是用来判断加入元素后是否超过最大缓存数， 如果超过就清除掉最少使用元素；

②get()方法访问元素后会将该元素移动到LinkedHashMap的尾部