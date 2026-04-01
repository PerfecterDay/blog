# Set
{docsify-updated}
- [Set](#set)
  - [HashSet 类](#hashset-类)
    - [`equals()` 与 `hashCode()` 方法](#equals-与-hashcode-方法)
  - [LinkedHashSet](#linkedhashset)
  - [TreeSet](#treeset)
    - [TreeSet 排序规则](#treeset-排序规则)
  - [各个 Set 实现类的性能分析](#各个-set-实现类的性能分析)
  - [BitSet](#bitset)


Set 不允许包含“相同”的元素，如果试图把两个相同的元素加入到同一个 Set 集合，则添加操作失败， add() 方法返回 false，且新元素也不会被加入。    
但是，不同的实现类对“相同元素”的定义略有不同。

### HashSet 类
`HashSet`底层是用散列表数据结构来实现。散列表为每个对象计算一个整数的散列码（hash code），散列码只和对象的状态有关，能够快速地被计算出来。Java的散列表使用链表数组实现，每个链表被称为一个桶，要想查找元素在散列表中的位置，先计算散列码，然后对桶的数目取余，所得到的结果就是保存这个元素的桶的位置索引。如果桶中没有其它元素，就保存在桶中，如果桶中已经保存了其它元素，我们称发生了散列冲突。此时，需要将新对象与桶中的所有元素进行比较，查看这个对象是否已经存在。

如果想要更好的控制散列表的性能，就要指定一个初始桶数。标准类库使用的桶数是2的幂，默认值为16（如果提供了桶参数，就会被自动地转换成2的幂）。如果散列表装得太满了，超过了**装填因子**，就需要进行再散列。这需要创建一个桶数更多得表，并将所有元素重新装填到新表中，然后丢弃原来的表。装填因子默认值为0.75，即表中超过75%的桶被占用时，就会进行再散列，默认会创建一个双倍桶数目的表。

`HashSet` 使用散列表来存储集合元素，因此具有良好的存储和查找性能，具有以下特点：
+ 不能保证元素的排列顺序，顺序可能与添加顺序不同，而且可能发生变化
+ 不是线程安全的
+ 元素值可以为 null

当向 `Hashset` 中存入一个元素时， `HashSet` 首先会调用该对象的 `hashCode()` 方法计算 hashCode 值，然后根据该 hashCode 值决定对象的存储位置。如果两个元素的 `equals()` 方法返回 true，但是 `hashCode()` 方法返回值不相等，HashSet 会将它们视为不同的元素存储在不同的位置。  
也就是说， `HashSet` 判断两个元素相同的标准是： **`equals()` 方法返回 true 且 `hashCode()` 方法返回值相等**。

#### `equals()` 与 `hashCode()` 方法
如果需要重写类的 `equals()` 方法，则也应该重写其 `hashCode()` 方法。规则是：如果两个对象通过 `equals()` 方法返回 true，则 `hashCode()` 方法返回值也应该相等。看以下反例：
1. `equals()`返回true，`hashCode()`返回值不等
这将导致两个元素存储到不同的位置，从而使两个相同的（`equals()` 为true）的元素都可以保存到同一个 Set，与Set集合的规则冲突了。
2. `equals()`返回false, `hashCode()`返回相等
这将导致不同的对象将存储在同一个位置（`hashCode()`相等），实际上 HashSet 以链表保存这些元素，这会导致性能的下降。

### LinkedHashSet
`LinkedHashSet `是 `HashSet` 的子类，它也是根据元素的 `hashCode()`值来决定元素的存储位置，但是它同时使用链表维护元素的**位置次序或者说添加次序**，使得元素看起来是以插入的顺序保存的的。也就是说，当遍历 `LinkedHashSet `集合里的元素时，会按照元素添加的顺序来访问集合里的元素。

`LinkedHashSet `需要维护元素的插入顺序，因此性能略低于 `HashSet` 的性能，但是在迭代访问集合中的全部元素时将有很好的性能，因为它以链表来维护内部顺序。

### TreeSet
`TreeSet` 是 `SortedSet` 接口的实现类，它可以确保集合元素处于**有序（不是位置有序）状态**。 `TreeSet` 提供了如下额外方法：
+ `Comparator<? super E> comparator()`：如果 `TreeSet` 使用了定制排序，则返回定制的 `Comparator`, 如果 `TreeSet` 使用了自然排序，则返回 null
+ `E first()`：返回集合中第一个元素
+ `E last()`：返回集合中最后一个元素
+ `E lower(E e)`：返回集合中位于指定元素e的前一个元素，e不需要是 `TreeSet` 中的元素
+ `E higher(E e)`：返回集合中位于指定元素e的后一个元素，e不需要是 `TreeSet` 中的元素
+ `SortedSet<E> subSet(E fromElement, E toElement)`：返回子集合Set，范围是[fromElement,toElement)
+ `SortedSet<E> headSet(E toElement)`：返回小于 toElement 元素组成的子集合
+ `SortedSet<E> tailSet(E fromElement)`：返回大于或等于 fromElement 的元素组成的子集合

当前，`TreeSet` 底层使用**红黑树而非散列表**结构来存储集合元素。

#### TreeSet 排序规则
`TreeSet` 支持两种排序方法：自然排序和定制排序，默认使用自然排序。

1. 自然排序  
    `TreeSet` 会调用集合元素的 `int compareTo(T o)` 方法来比较元素之间的大小关系，然后将集合元素按升序排列。因此待插入的元素必须实现 `Comparable` 接口，否则会引发`ClassCastException` 。 
    当调用 `obj1.compareTo(obj2)` 时，如果返回值大于0，代表 obj1 大于 obj2 ，等于0代表两者相等，小于0代表 obj1 小于 obj2 。    
    在这种情况下，对于 `TreeSet` 集合而言，判断两个对象是否相等的唯一标准是：**两个对象通过 `compareTo()` 方法比较是否返回0**。即使 `equals()` 方法返回true，而 `compareTo()` 方法返回不为0， `TreeSet` 也将两者视为不相元素。因此，如果要把对象插入到 `TreeSet` ，且重写了 `equals()` 方法，则应该保证 `equals()` 相等的两个对象在调用 `compareTo()` 方法时返回0。

2. 定制排序  
    自然排序默认使用升序排列，如果想要实现如降序排列等其它特性，则可以通过 `Comparator` 接口的帮助来实现。该接口里包含一个 `int compare(T o1, T o2)` 方法用于比较 o1 和 o2 的大小： 
    + 如果该方法返回正整数，则表明 o1 大于 o2；
    + 如果该方法返回负整数，则表明 o1 小于 o2；
    + 如果该方法返回0，则表明 o1 等于 o2；
    如果需要实现定制排序，则要在创建 `TreeSet` 集合时，提供一个 `Comparator` 给 `TreeSet` 的构造器，由该 `Comparator` 负责元素的排序逻辑。

    在这种情况下，对于 `TreeSet` 集合而言，判断两个对象是否相等的唯一标准是：**两个对象通过 `compare()` 方法比较是否返回0** 。即使 `equals()` 方法返回true，而 `compare()` 方法返回不为0， `TreeSet` 也将两者视为不相元素。因此，如果要把对象插入到 `TreeSet` ，且重写了 `equals()` 方法，则应该保证 `equals()` 相等的两个对象在调用 `compare()` 方法时返回0。

### 各个 Set 实现类的性能分析
`HashSet` 和 `TreeSet` 是 Set 的两个典型实现， `HashSet` 总是比 `TreeSet` 的性能好，因为 `TreeSet` 需要额外的红黑树来维护集合元素次序，只有当需要保持排序的 Set 时，才应该考虑使用 `TreeSet` 。  
`LinkedHashSet `的插入、删除操作要比 `HashSet` 略慢，这是由于要维护链表所带来的额外开销所致，但是由于有了链表，在遍历集合元素时会更快。 
`EnumSet` 是所有 Set 中性能最好的，但它只能保存同一个枚举类的枚举值作为集合元素。

### BitSet
`BitSet` 并不是集合中的一份子，事实上，它没有实现 Collections 或 Map 接口，但是因为它的功能与集合类似，是用来存储位序列的，所以把它放在这里。 `BitSet`将位包装在字节中，所以非常高效。它提供了一些便于读取、设置或清除各个位的接口：
1. `boolean get(int i)`:返回第i位的状态（true或false）
2. `void set(int i)`:将第 i位设置 true
3. `void clear(int i)`:将第 i位设置 false
4. `void and(BitSet set)`:与另一个 BitSet 进行逻辑 AND 操作
5. `void or(BitSet set)`:与另一个 BitSet 进行逻辑 OR 操作
6. `void xor(BitSet set)`:与另一个 BitSet 进行逻辑 XOR 操作
7. `void andNot(BitSet set)`:清除这个位集中对应另一个位集中设置的所有位
8. `int length(int i)`:返回长度