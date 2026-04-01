# 集合类概述
{docsify-updated}

- [集合类概述](#集合类概述)
	- [概述](#概述)
	- [Collection接口](#collection接口)
		- [遍历操作 Collection 的三种方法](#遍历操作-collection-的三种方法)
		- [集合与数组之间的转换](#集合与数组之间的转换)
		- [算法](#算法)


Java 集合大致可分为 Set 、List 、 Queue 和 Map 四种体系，其中
+ Set 代表 **无序的、不可重复的集合**；
+ List 代表**位置有序、可重复的集**
+ Map 代表具有**key->value映射关系**的集合
+ Queue 代表**队列**集合的实现，是 Java 5 新增的

## 概述
Java 集合主要由两个接口派生而出： `Collection` 和 `Map` ，它们是整个集合框架的根接口。
<center>
<img src="pics/java-collection.jpg" alt="" width=45% >
<img src="pics/java-collection-2.jpg" width="50%">
</center>

## Collection接口
Collection的常用方法：
+ `Iterator<E> iterator()`：返回一个 Iterator 迭代器用于便利元素
+ `int size()`：返回当前集合中元素的数量
+ `boolean isEmpty()`：判断集合是否为空
+ `boolean contains(Object o)`：查找集合中是否有指定的对象
+ `boolean containsAll(Collection c)`：查找集合中是否有集合c中的所有元素
+ `boolean add(Object o)`：添加对象到集合，添加成功返回 true
+ `boolean addAll(Collection<? extends E> c)`:把集合c里所有的元素都添加到集合里，如果集合被改变了则返回 true
+ `boolean remove(Object o)`：删除指定的对象
+ `boolean removeAll(Collection c)`：从集合中删除c集合中也有的元素，如果有一个或多个元素被删除，返回 true
+ `void clear()`：清空集合里的元素，将集合长度变为0
+ `void retainAll(Collection c)`：从集合中删除集合c中不包含的元素，如果调用改变了集合，返回 true
+ `Object[] toArray()`：把集合转换成一个Object数组，所有元素成为数组的元素。
+ `<T> T[] toArray(T[] a)`：把集合转换成一个**指定类型**数组，所有元素成为数组的元素。

### 遍历操作 Collection 的三种方法
1. 使用 `Iterator` 迭代器遍历元素  
	`Iterator` 必须依附于某个 `Collection` 对象,`Iterator` 接口定义了如下4个方法：  
	+ `boolean hasNext()`：如果被迭代的集合元素没有遍历完则返回 true
	+ `E next()`：返回集合里的下一个元素
	+ `void remove()`：删除集合中上一次 next 方法返回的元素
	+ `void forEachRemaining(Consumer<? super E> action)`：Java 8 新增方法，可以使用 Lambda 表达式来遍历集合

	<b>当使用for循环或者foreach遍历集合元素时，需要注意：
	`Collection` 集合里的元素不能被改变，增删都不行，但是可以更新设置某个元素的值，只有通过 `Iterator` 的 `remove()` 方法删除集合元素才可以；假如此时修改了集合（遍历的时候或者另外一个线程改变了集合元素的个数），将会引发 `ConcurrentModificationException` 异常。但是，也不能有多个 `Iterator` 在遍历的时候，修改集合，否则也会引发 `ConcurrentModificationException` 异常，多个迭代器遍历时，只能有一个迭代器能修改集合。</b>

1. 使用 `Iterable` 接口  
	使用 `Iterable` 的 `void forEach(Consumer<? super T> action)` 方法， `Iterable` 是 `Collection` 的父接口，因此可以直接调用此方法。
	同样，使用此方法遍历元素时，`Collection` 集合也不能增删，否则将会引发 `ConcurrentModificationException` 异常。

3. 使用 `foreach` 循环遍历集合元素
	```
	List list = new ArrayList():
	list.add("adass"):
	for (Object o : list) {
		.....
	}
	```
	同样，使用此方法遍历元素时，`Collection` 集合也不能增删，否则将会引发 `ConcurrentModificationException` 异常。

### 集合与数组之间的转换
1. 数组转集合
   使用 `Arrays.asList(Object[])`方法能将数组转换为 List，继而可以作为构造器参数用来构造各种集合（Map,Set,Queue等）。注意，这样得到的list 是不能执行 `add()/remove()`等操作的。
2. 集合转数组
   使用 `Collection` 接口的`toArray()`方法转换为 `Object[]`,或者提供指定类型数组参数来转换成指定类型的数组，如：
   `staff.toArray(new String[staff.size()])`


### 算法
集合上的算法主要由 `Collections` 的静态方法来提供：
1. `void sort(List list)`:排序
2. `void sort(List list,Comparator comparator)`:使用指定的比较器排序
3. `void shuffle(List list)`:随机打乱
4. `void shuffle(List list,Random r)`:随机打乱
5. `int binarySearch(List<T> list, T key)`：二分查找一个值，返回它的位置索引
