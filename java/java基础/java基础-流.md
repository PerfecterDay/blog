# 流
{docsify-updated}

- [流](#流)
  - [Java 流概述](#java-流概述)
  - [创建Stream](#创建stream)
  - [Collectors API](#collectors-api)


### Java 流概述
流遵循了“做什么而非怎么做”的原则 。 在流的示例中，我们描述了需要做什么:获取长单词，并对它们计数 。我们没有指定该操作应该以什么顺序或者在哪个线程中执行 。相比之下，本节开头处的循环要确切地指定计算应该如何工作，因此也就丧失了进行优化的机会。
流表面上看起来和集合很类似，都可以让我们转换和获取数据。但是，它们之间存在着显著的差异:
1. 流并不存储其元素。 这些元素可能存储在底层的集合中，或者是按需生成的。
2. 流的操作不会修改其数据源 。 例如， filter 方法不会从新的流中移除元素，而是会
生成一个新的流，其中不包含被过滤掉的元素 。
3. 流的操作是尽可能惰性执行的。 这意味着直至需要其结果时，操作才会执行。 例如，如果我们只想查找前 5 个长单词而不是所有长单词，那么 filter 方法就会在匹配到第 5 个 单词后停止过滤 。 因此，我们甚至可以操作无限流 。

Java 8 中增加了 Stream API，简化了串行或并行的大批量操作。这个 API 提供了两个关键抽象：
1. Stream（流）

   代表数据元素的有限或无限顺序，这些元素可能来自任何位置，常见的来源包括集合、数组、文件、正则表达式模式匹配器、伪随机数生成器，以及其他 Stream 。Stream 中的数据元素可以是对象引用、或者基本类型值。它支持三种基本类型：int、long 和 double 。
   有同学会问：一个顺序输出的Java对象序列，不就是一个List容器吗？
   **这个Stream和List也不一样，List存储的每个元素都是已经存储在内存中的某个Java对象，而Stream输出的元素可能并没有预先存储在内存中，而是实时计算出来的。换句话说，List的用途是操作一组已存在的Java对象，而Stream实现的是惰性计算，参看[这个例子](#demo)**

2. Strean pipeline（流管道）

   代表这些元素的一个多级计算。一个 Strean pipeline 中包含一个源 Stream ，接着是 0 或者多个中间操作和一个终止操作。每个中间操作都会通过某种方式对 Stream 进行转换，例如将每个元素通过一个函数映射到另一元素，或者过滤掉某些不满足条件的元素。所有的中间操作都会将 Stream 转换成另一个 Stream， 其元素类型可能与输入的 Stream 一样，也可能不同。终止操作会在最后一个中间操作产生的 Stream 上执行一个最终计算，例如将其元素保存到一个集合中，并返回某一个元素，或打印出所有元素等。
   Stream pipeline 通常是 lazy 的：直到调用终止操作时才会开始计算，对于完成终止操作不需要的数据元素，将永远都不会被计算。正是这种 lazy 计算，使无限 Stream 成为可能。没有终止操作的 Stream pipeline 将是一个静默的无操作指令，因此千万不能忘记终止操作。
   Stream API 是流式的：所有的 pipeline 的调用可以链接成一个表达式。事实上，多个 pipeline 也可以链接在一起，成为一个表达式。
   默认情况下，Stream pipeline 是按顺序运行的。要使 pipeline 并发执行，只需在该 pipeline 的任何 Stream 上调用 parallel 方法即可，但是通常不建议这么做。

### 创建Stream
1. Stream.of()

    创建Stream最简单的方式是直接用Stream.of()静态方法，传入可变参数即创建了一个能输出确定元素的 Stream.
    ```
    Stream<String> stream = Stream.of("A", "B", "C", "D");
            // forEach()方法相当于内部循环调用，
            // 可传入符合Consumer接口的void accept(T t)的方法引用：
            stream.forEach(System.out::println);
    ```

2. 基于数组或Collection
	可以用 Collection 接口的 `stream` 方法将任何集合转换为一个流,或者调用 `Arrays.stream(arr)` 将数组转换为流
    ```
    Stream<String> stream1 = Arrays.stream(new String[] { "A", "B", "C" });
    Stream<String> stream2 = List.of("X", "Y", "Z").stream();
    stream1.forEach(System.out::println);
    stream2.forEach(System.out::println);
    ```

3. 基于Supplier的无限流

    创建Stream还可以通过 `Stream.generate()`方法，它需要传入一个`Supplier`对象：
    `Stream<String> s = Stream.generate(Supplier<String> sp);`
    基于`Supplier`创建的Stream会不断调用`Supplier.get()`方法来不断产生下一个元素，这种Stream保存的不是元素，而是算法，它可以用来表示无限序列。
    例如，我们编写一个能不断生成自然数的Supplier，它的代码非常简单，每次调用get()方法，就生成下一个自然数：
    
    ```
    public class Main {
        public static void main(String[] args) {
            Stream<Integer> natual = Stream.generate(new NatualSupplier());
            // 注意：无限序列必须先变成有限序列再打印:
            natual.limit(20).forEach(System.out::println);
        }
    }
    class NatualSupplier implements Supplier<Integer> {
        int n = 0;
        public Integer get() {
            n++;
            return n;
        }
    }
    ```

4. 其他方法

    创建Stream的第三种方法是通过一些API提供的接口，直接获得Stream。
    `Files`类的`lines()`方法可以把一个文件变成一个Stream，每个元素代表文件的一行内容
    正则表达式的`Pattern`对象有一个`splitAsStream()`方法，可以直接把一个长字符串分割成Stream序列而不是数组。

### Collectors API 
Collectors API 又叫收集器，它的作用是将 Stream 的元素合并到单个对象中去，收集器产生的对象一般是一个集合。
将 Stream 的元素集中到一个真正的 Collection 里去的收集器比较简单，它有三个这样的收集器： `toList()`、`toSet()`、`toCollection(collectionFactory)`，它们分别返回一个列表、一个集合和程序员指定的集合类型。`toMap(keyMapper,valueMapper)` 将 Stream 元素集合到 Map 中，keyMapper 是一个将 Stream 元素映射到 Map 中键的函数，而 valueMapper 是将 Stream 元素映射到 Map 中值的函数。