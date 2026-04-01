# Map
{docsify-updated}

- [Map](#map)
	- [Map接口](#map接口)
	- [HashMap](#hashmap)
	- [TreeMap](#treemap)
	- [LinkedHashMap](#linkedhashmap)
- [HashMap 和 Hashtable 的区别](#hashmap-和-hashtable-的区别)


映射表数据结构能保存键值对信息，并且能快速地使用键查找到对应的值。Java类库为映射表提供了两个通用实现: `HashMap`和`TreeMap`。

### Map接口
+ `int size()`:返回映射表大小
+ `boolean isEmpty()`：返回是否为空
+ `boolean containsKey(Object key)`：判断某个键是否存在
+ `boolean containsValue(Object value)`：判断某个值是否存在
+ `V get(Object key)`：获取某个键对应的值
+ `V put(K key, V value)`：设置某个键值对，返回改键之前的值，之前不存在则返回null
+ `V remove(Object key)`：删除某个键值对
+ `void putAll(Map<? extends K, ? extends V> m)`：将一个 map 中的键值对全部放到当前 map 中
+ `void clear()`：清空键值对
+ `Set<K> keySet()`：获取键的集合
+ `Set<Map.Entry<K, V>> entrySet()`：获取键值对集合
+ `Collection<V> values()`：获取值集合

### HashMap
`HashMap`底层使用的技术与`HashSet`一致,都是使用散列表数据结构实现的。需要注意的是，`HashMap`中的散列和比较只发生在键上，与键关联的值不能进行散列或比较。同样， `HashMap`也可以设置初始桶数量和装填因子。

### TreeMap
`TreeMap`与`TreeSet`类似，也是有序的映射表，**可以指定在构造时指定 `Comparator` 对象对元素进行排序。**

### LinkedHashMap
`LinkedHashMap`**是插入位置有序的映射表，元素会按照加入映射表的顺序排序。**

# HashMap 和 Hashtable 的区别
1. `Hashtable` 是一个线程安全的 Map 实现，但 `HashMap` 不是线程安全的，因为没有锁的开销，所以 `HashMap` 性能会更好点
2. `Hashtable` 不允许使用 null 作为 key 和 value。如果试图把 null 放入 `Hashtable` 将会引发 NPE，但 `HashMap` 可以使用 null 作为 key 或 value。     
与 `HashSet` 类似， `HashMap` 、 `Hashtable` 判断两个 key 相等的标准也是： **两个 key 通过 `equals()` 方法返回 true ，且 `hashCode()` 返回值也相同。**    
另外， `HashMap` 、 `Hashtable` 中还有一个 `containsValue()` 方法，用于判断是否包含指定 value。此时，判断是否与 value 相等的标准更简单：**只需要 equals() 方法返回 true 即可。**
