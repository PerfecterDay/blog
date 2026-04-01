# Queue
{docsify-updated}
- [Queue](#queue)
	- [PriorityQueue 实现类](#priorityqueue-实现类)
	- [Deque 接口与 ArrayDeque 实现类](#deque-接口与-arraydeque-实现类)
	- [DelayQueue](#delayqueue)


`Queue` 用于模拟队列这种数据结构，队列通常是先进先出（FIFO）的容器。队列的头部保存存放时间最长的元素，队尾保存存放时间最短的元素。通常人们可以快速地在队尾添加一个元素（入队），并且在对头移除一个元素（出队）。双端队列允许在队列两端同时添加或删除元素。但是队列通常不支持在中间添加或删除元素，也不允许随机访问。

`Queue` 提供了如下常用方法:
+ `boolean add(E e)`：将指定元素添加到队列的尾部并返回true，队列满了将抛出 IllegalStateException
+ `boolean offer(E e)`：将指定元素添加到队列的尾部，队列满了返回false
+ `E remove()`：获取队列的头部元素，并删除该元素。如果队列为空则返回 NoSuchElementException
+ `E poll()`：获取队列头部的元素，并删除该元素（出队），如果队列为空返回 null
+ `E element()`：获取队列头部的元素，但是不删除该元素（不出队），如果队列为空则返回 NoSuchElementException
+ `E peek()`：获取队列头部的元素，但是不删除该元素，如果队列为空，返回null

`Queue` 还有一个 `Deque` 子接口， `Deque` 代表一个 *双端队列* ，`双端队列可以同时从两端来添加、删除元素`。因此 `Deque` 的实现类既可以当成队列使用，也可以当成栈使用。JDK为 `Deque` 提供了 `ArrayDeque` 和 `LinkedList` 两个实现类。

### PriorityQueue 实现类
`PriorityQueue` 保存队列的顺序不是按元素加入的顺序，而是按照队列元素的大小进行重新排序的。出队时，取的不是最先进入队列的元素，而是队列中最小的元素（队尾到队头降序排列）。  
因为涉及到排序，因此也有自然排序和定制排序两种方式，与前文所述相同。

### Deque 接口与 ArrayDeque 实现类
+ `void addFirst(E e)`:将指定元素插入双端队列的队头，队列满了将抛出 IllegalStateException
+ `void addLast(E e)`:将指定元素插入到双端队列的队尾，队列满了将抛出 IllegalStateException
+ `boolean offerFirst(E e)`:将指定元素插入该双端队列的队头，队列满了返回false
+ `boolean offerLast(E e)`:将指定元素插入该双端队列的队尾，队列满了返回false
+ `E removeFirst()`:获取并删除队头元素，如果队列为空则返回 NoSuchElementException
+ `E removeLast()`:获取并删除队尾元素，如果队列为空则返回 NoSuchElementException
+ `E pollFirst()`:获取并删除队头元素，如果队列为空，返回 null
+ `E pollLast()`:获取并删除队尾元素，如果队列为空，返回 nul
+ `E getFirst()`:获取但并不删除队头元素，如果队列为空则返回 NoSuchElementException
+ `E getLast()`:获取但并不删除队尾元素，如果队列为空则返回 NoSuchElementException
+ `E peekFirst()`:获取但并不删除队头元素，如果队列为空，返回 nul
+ `E peekLast()`:获取但并不删除队尾元素，如果队列为空，返回 nul
+ `boolean removeFirstOccurrence(Object o)`:删除第一次出现的元素o
+ `boolean removeLastOccurrence(Object o)`:删除最后一次出现的元素o
+ `void push(E e)`:将元素e入栈
+ `E pop()`:pop处栈顶元素
+ `public int size()`:获取队列大小
+ `Iterator<E> iterator()`:获取队列迭代器

`Deque` 不仅可以当成双端队列来使用，而且可以被当成栈来使用，因为含有 `push()` 、 `pop()` 方法。  
`Deque` 接口有一个典型实现类 `ArrayDeque` ，基于数组实现的双端队列，可以在初始化时指定数组大小，如果不指定，则默认大小是 16。


### DelayQueue
```
public static void main(String[] args) throws InterruptedException {
	DelayObject objectA = new DelayObject("a",10000);
	DelayObject objectB = new DelayObject("b",2000);
	DelayObject objectC = new DelayObject("c",1000);

	DelayQueue<DelayObject> delayQueue = new DelayQueue();
	delayQueue.put(objectA);
	delayQueue.put(objectB);
	delayQueue.put(objectC);
	DelayObject object = delayQueue.take();
	while (object != null){
		System.out.println("Consumer take: " + object);
		object = delayQueue.take();
	}
}

static class DelayObject implements Delayed {
	private String data;
	private long startTime;

	DelayObject(String data, long delayInMilliseconds) {
		this.data = data;
		this.startTime = System.currentTimeMillis() + delayInMilliseconds;
	}

	//返回剩余的延时；零或负值表示延时已过，可以出队
	@Override
	public long getDelay(TimeUnit unit) {
		long diff = startTime - System.currentTimeMillis();
		return unit.convert(diff, TimeUnit.MILLISECONDS);
	}

	@Override
	public int compareTo(Delayed o) {
		return Ints.saturatedCast(this.startTime - ((DelayObject) o).startTime);
	}

	@Override
	public String toString() {
		return "{" + "data='" + data + '\'' + ", startTime=" + startTime + '}';
	}
}
```