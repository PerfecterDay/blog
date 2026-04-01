# NIO 之 Buffer
{docsify-updated}

## 缓冲区
`Buffer` 是一个对象， 它包含一些要写入或者刚读出的数据。 在 NIO 中加入 `Buffer` 对象，体现了新库与原 I/O 的一个重要区别。在面向流的 I/O 中，您将数据直接写入或者将数据直接读到 Stream 对象中。

在 NIO 库中，所有数据都是用缓冲区处理的。在读取数据时，它是直接读到缓冲区中的。在写入数据时，它是写入到缓冲区中的。任何时候访问 NIO 中的数据，都是将它放到缓冲区中。

缓冲区实质上是一个数组。通常它是一个字节数组，但是也可以使用其他种类的数组。但是一个缓冲区不仅仅是一个数组。缓冲区提供了对数据的结构化访问，而且还可以跟踪系统的读/写进程。

### 缓冲区的状态变量
可以用三个值指定缓冲区在任意时刻的状态：
+ `capacity` -- Buffer 的容量
+ `position` -- 下一个读写的位置
+ `limit`  -- 读写的最后一个位置

这三个变量一起可以跟踪缓冲区的状态和它所包含的数据。

#### 写模式
```
ByteBuffer buf = ByteBuffer.allocate(10);

position = 0
limit    = 10
capacity = 10
----------------

buf.put((byte)1);
buf.put((byte)2);

[ 1 ][ 2 ][   ][   ][   ][   ][   ][   ][   ][   ]
          ↑
       position=2
limit=10
```

### 读模式
```
buf.flip();

position=0
limit=2
capacity=10
-------------------

buf.get(); // 1
buf.get(); // 2
```

#### flip方法
现在我们要将数据写到输出通道中。在这之前，我们必须调用 `flip()` 方法。这个方法做两件非常重要的事：
```
limit = position
position = 0
```
+ 它将 limit 设置为当前 position。
+ 它将 position 设置为 0。

#### clear方法
从通道读入数据到缓冲区之前，这个方法重设缓冲区以便接收更多的字节。 `Clear()` 做两件非常重要的事情：
```
limit = capaticy;
position = 0;
```
+ 它将 limit 设置为与 capacity 相同。
+ 它设置 position 为 0。

#### get() 方法
`ByteBuffer` 类中有四个 `get()` 方法：
+ `byte get()` : 获取单个字节
+ `ByteBuffer get(byte[] dst)` : 将一组字节读到一个数组中
+ `ByteBuffer get(byte[] dst, int offset, int length)` : 将一组字节读到一个数组中，从 `offset` 开始的 `length` 个字节
+ `byte get( int index)` : 从缓冲区中的特定位置获取字节

此外，我们认为前三个 `get()` 方法是相对的，而最后一个方法是绝对的。 相对意味着 `get()` 操作服从 `limit` 和 `position` 值 ― 更明确地说，字节是从当前 `position` 读取的，而 `position` 在 `get` 之后会增加。另一方面，一个 绝对方法会忽略 `limit` 和 `position` 值，也不会影响它们。事实上，它完全绕过了缓冲区的统计方法。  
上面列出的方法对应于 `ByteBuffer` 类。其他类有等价的 `get()` 方法，这些方法除了不是处理字节外，其它方面是是完全一样的，它们处理的是与该缓冲区类相适应的类型。

#### put()方法
`ByteBuffer` 类中有五个 `put()` 方法：
+ `ByteBuffer put( byte b );`
+ `ByteBuffer put( byte src[] );`
+ `ByteBuffer put( byte src[], int offset, int length );`
+ `ByteBuffer put( ByteBuffer src );`
+ `ByteBuffer put( int index, byte b );`

第一个方法写入（put） 单个字节。第二和第三个方法写入来自一个数组的一组字节。第四个方法将数据从一个给定的源 `ByteBuffer` 写入这个 `ByteBuffer` 。第五个方法将字节写入缓冲区中特定的 位置 。那些返回 `ByteBuffer` 的方法只是返回调用它们的缓冲区的 this 值。

与 `get()` 方法一样，我们将把 `put()` 方法划分为 相对 或者 绝对 的。前四个方法是相对的，而第五个方法是绝对的。

上面显示的方法对应于 `ByteBuffer` 类。其他类有等价的 `put()` 方法，这些方法除了不是处理字节之外，其它方面是完全一样的。它们处理的是与该缓冲区类相适应的类型。

#### 缓冲区分配和包装
在能够读和写之前，必须有一个缓冲区。要创建缓冲区;
1. 分配它。我们使用静态方法 allocate() 来分配缓冲区：
```
ByteBuffer buffer = ByteBuffer.allocate( 1024 );
```
2. 还可以将一个现有的数组转换为缓冲区，如下所示：
```
byte array[] = new byte[1024];
ByteBuffer buffer = ByteBuffer.wrap( array );
```