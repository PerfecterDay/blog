# List
{docsify-updated}

- [List](#list)
	- [List 接口](#list-接口)
	- [ListIterator 接口](#listiterator-接口)
	- [ArrayList 和 Vector](#arraylist-和-vector)
	- [LinkedList 实现类](#linkedlist-实现类)
	- [Arrays.ArrayList](#arraysarraylist)
	- [各种线性表的性能分析](#各种线性表的性能分析)
	- [常用操作](#常用操作)


`List` 集合代表一个元素位置有序、可重复的集合，集合中每个元素都有其对应的**顺序索引（位置）**。访问元素一般有两种方式：
1. 使用迭代器顺序访问
2. 使用 `get` 和 `set` 方法随机访问元素

第二种方式不适合基于链表的实现。

### List 接口
`List` 作为 `Collection` 接口的子接口，当然可以使用 `Collection` 接口里的全部方法。而且因为 `List` 是位置有序集合，因此 `List` 里增加了一些根据位置索引来操作集合的方法：
+ `ListIterator listIterator()`：返回一个列表迭代器，用来遍历 List
+ `void add(int index, E element)`：将 element 添加到 index 处
+ `boolean addAll(int index,Collection<? extends E> c)`：将集合c所包含的所有元素都插入到 List 集合的 index 处
+ `E remove(int index)`：删除并返回 index 索引处的元素
+ `E set(int index, E element)`：将 index 索引处的元素替换成 element 对象，并返回被替换的旧元素
+ `E get(int index)`:返回集合 index 处的元素
+ `int indexOf(Object o)`：返回对象o在List集合中第一次出现的位置索引，没有返回-1
+ `int lastIndexOf(Object o)`：返回对象o在List集合中最后次出现的位置索引，没有返回-1
+ `List<E> subList(int fromIndex, int toIndex)`：返回从索引 [fromIndex,toIndex)处所有元素组成的子集合。
+ `void replaceAll(UnaryOperator<E> operator)`：根据 operator 指定的计算规则重新设置List集合所有元素
+ `void sort(Comparator<? super E> c)`：根据 Comparator 参数对 List 排序。

List 判断两个对象相等的依据是：**只要`equals()` 方法返回 true ，两者就视为相等。**

### ListIterator 接口
List 除了可以使用 `iterator()` 来获取 `Iterator` 迭代器以外，还提供了一个 `ListIterator listIterator()` 方法获取一个 `ListIterator` `接口，ListIterator` 继承了 `Iterator` ，并增加了如下方法：
+ `boolean hasPrevious()`：返回该迭代器关联的集合是否还有上一个元素
+ `E previous()`：返回上一个元素
+ `void add(E e)`：在指定位置插入一个元素
+ `void set(E e)`：更新指定位置元素值

### ArrayList 和 Vector
`ArrayList` 和 `Vector` 都是基于**数组**实现的 List 类，底层封装了一个动态的、允许再分配的 `Object[]` 动态数组（或泛型数组）。

如果创建 `ArrayList` 或者 `Vector` 时，没有指定数组长度，则默认大小是10，如果 List 长度增长到数组最大长度时，再添加元素就要重新分配数组并将原数组中的元素拷贝到新数组中，会有很大开销。所以，如果开始就知道 List 的大小，可以在创建时就指定，这样能避免数组重分配，提高性能。

此外，两者还提供了如下方法来重新分配数组：
+ `void ensureCapacity(int minCapacity)`：增加数组长度，数组长度 += minCapacity
+ `void trimToSize()`：调整数组长度为当前元素个数，释放空间

两者有一个显著的区别就是： `ArrayList` 是线程不安全的，而 `Vector` 是线程安全的。即使如此，也不推荐使用 `Vector` 类，如果想要线程安全可以使用 `Collections` 工具类，它可以将 `ArrayList` 变成线程安全的类。

### LinkedList 实现类
`LinkedList` 底层是由链表结构实现的，Java 种的链表都是由双向链表实现的。
`LinkedList` 既实现了 `List` 接口，同时也实现了 `Deque` 接口。因此，其功能非常强大，可以用来做 `List` , 也可以作为队列、栈、双端队列等数据结构使用。常用方法：
+ `addFirst(E element)` ： 将元素添加到队头
+ `addLast(E element)` ： 将元素添加到队尾
+ `getFirst(E element)` ： 获取队头元素
+ `getLast(E element)` ： 获取队尾元素
+ `removeFirst(E element)` ： 删除并返回队头元素
+ `removeLast(E element)` ： 删除并返回队尾元素


### Arrays.ArrayList
`Arrays` 工具类的 `asList(Object... os)` 方法可以把一个数组或指定个数的对象转换成一个 List 集合，这个 List 实际上是 `Arrays.ArrayList` 的实例。

`Arrays.ArrayList` 是一个固定长度的 List 集合，程序**只能遍历访问该集合里的元素，不可增加、删除其中的元素**。任何添加、删除操作将在运行时引发`UnsupportedOperationException` 异常。


### 各种线性表的性能分析
一般来说，由于数组以一块连续内存来保存数据，所以数组的随机访问性能最好，所有内部以数组作为底层实现的集合类在随机访问时都有较好的性能；内部以链表实现的集合在执行插入、删除操作的集合有较好的性能。但总体上说， `ArrayList` 的性能要优于 `LinkedList` ，大部分时候应该考虑使用 `ArrayList` 。

关于 List 集合的使用有如下建议：
1. 如果需要遍历 List 集合元素：对于数组类（`ArrayList、ArrayDeque、Vector`等）List应该使用随机访问形式(`get(index)`)来遍历；对于链表类（ `LinkedList` )，则应该采用迭代器（`Iterator`)来遍历集合元素
2. 如果需要经常执行插入、删除操作来改变包含大量数据的List集合，应该优先考虑链表类的 `LinkedList` 。使用 `ArrayList` 可能需要经常重新分配内存数组
3. 如果有多个线程需要同时访问List集合，应该使用 `Collections` 工具类将集合包装成线程安全的类。

### 常用操作
1. 拷贝list
   + `List<Plant> copy = new ArrayList<>(list);` 由于我们在这里复制的是引用，而不是克隆对象，因此对一个元素的每次修改都会影响两个列表，因此，最好使用构造函数来复制不可变对象。
   + `copy.addAll(list);`
   + `Collections.copy(dest, source);`: 必须保证 dest 的size 大于等于 src 的size
   + `List<T> copy = List.copyOf(list);`： 唯一的条件是给定的集合不能为null，也不能包含任何null元素。
