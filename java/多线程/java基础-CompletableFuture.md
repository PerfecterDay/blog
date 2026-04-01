# Future/FutureTask 与 CompletableFuture
{docsify-updated}
>https://www.baeldung.com/java-completablefuture  
>https://tech.meituan.com/2022/05/12/principles-and-practices-of-completablefuture.html

- [Future/FutureTask 与 CompletableFuture](#futurefuturetask-与-completablefuture)
    - [Future](#future)
    - [CompletableFuture](#completablefuture)
      - [将 CompletableFuture 当作简单Future来用](#将-completablefuture-当作简单future来用)
    - [相继执行多个任务](#相继执行多个任务)
    - [等待异步任务的完成](#等待异步任务的完成)
    - [编排任务](#编排任务)
      - [零依赖：CompletableFuture的创建](#零依赖completablefuture的创建)
      - [一元依赖：依赖一个CF](#一元依赖依赖一个cf)
      - [二元依赖：依赖两个CF](#二元依赖依赖两个cf)
      - [多元依赖：依赖多个CF](#多元依赖依赖多个cf)
  - [注意的问题](#注意的问题)


异步计算很难推理。通常，我们希望将任何计算视为一系列步骤，但在异步计算中，表示为回调的操作往往分散在代码中，或者相互嵌套。当我们需要处理某个步骤中可能出现的错误时，情况就会变得更糟。

Java 5 中添加了 `Future` 接口来作为异步计算的结果，但它没有任何方法来组合这些计算或处理可能出现的错误。

Java 8 引入了 `CompletableFuture` 类。除了 `Future` 接口，它还实现了 `CompletionStage` 接口。该接口定义了异步计算步骤的契约，我们可以将其与其他步骤结合起来。

`CompletableFuture` 同时是一个构件和框架，拥有约 50 种不同的方法，用于组成、组合和执行异步计算步骤以及处理错误。

### Future
Future是为了配合Callable/Runnable而产生的，既然有返回值，那么返回什么？什么时候返回？这些问题其实都可以算在Future机制里。  
简单来讲我认为Future是一个句柄，即Callable任务返回给调用方这么一个句柄，通过这个句柄我们可以跟这个异步任务联系起来，我们可以通过future来对任务查询、取消、执行结果的获取，是调用方与异步执行方之间沟通的桥梁。

```
ExecutorService threadpool = Executors.newCachedThreadPool();
Future<Long> futureTask = threadpool.submit(() -> factorial(number));

while (!futureTask.isDone()) {
    System.out.println("FutureTask is not finished yet..."); 
} 
long result = futureTask.get(); 

threadpool.shutdown();
```

### CompletableFuture

#### 将 CompletableFuture 当作简单Future来用
首先，CompletableFuture 类实现了 Future 接口，因此我们可以将其用作 Future 实现，但需要附加完成逻辑。
例如，我们可以用一个无参数构造函数创建该类的一个实例来表示未来的某个结果，然后将其交给消费者，并在未来某个时间使用 complete 方法完成它。消费者可以使用 get 方法阻塞当前线程，直到该结果被提供。

```
public Future<String> calculateAsync() throws InterruptedException {
    CompletableFuture<String> completableFuture = new CompletableFuture<>();

    Executors.newCachedThreadPool().submit(() -> {
        Thread.sleep(500);
        completableFuture.complete("Hello");
        return null;
    });

    return completableFuture;
}
```

### 相继执行多个任务
假设我们需要调用两个远程 API：`firstApiCall()` 和 `secondApiCall()`。第一个 API 的结果将是第二个 API 的输入。如果使用 Future 接口，就无法异步地将这两个操作结合起来：
```
class Demo {
 public static void main(String[] args) throws ExecutionException, InterruptedException {
   ExecutorService executor = Executors.newSingleThreadExecutor();
   Future<String> firstApiCallResult = executor.submit(
           () -> firstApiCall(someValue)
   );
   
   String stringResult = firstApiCallResult.get();
   Future<String> secondApiCallResult = executor.submit(
           () -> secondApiCall(stringResult)
   );

   doOtherThings();
 }
}
```
在上面的代码示例中，我们通过在 `ExecutorService` 上提交一个返回 Future 的任务来调用第一个 API。我们需要将该值传递给第二个 API，但获取该值的唯一方法是使用我们之前讨论过的 Future 方法的 `get()`，而使用该方法会**阻塞主线程**。现在我们必须等到第一个 API 返回结果后再做其他事情。

```
class Demo {
  public static void main(String[] args) {

    var finalResult = CompletableFuture.supplyAsync(
         () -> firstApiCall(someValue)
    )
    .thenApply(firstApiResult -> secondApiCall(firstApiResult));

	doOtherThings();
  }
}
```

### 等待异步任务的完成
```java
class Demo {
  public static void main(String[] args) {

  CompletableFuture cf1 = CompletableFuture.supplyAsync(
         () -> firstApiCall(someValue)
    ).thenApply(firstApiResult -> secondApiCall(firstApiResult));

  CompletableFuture cf2 = CompletableFuture.supplyAsync(
         () -> firstApiCall(someValue)
    ).thenApply(firstApiResult -> secondApiCall(firstApiResult)); 

  // cf3....
  cf1.join();
  cf2.join();
  ...

  CompletableFuture.allOf(cf1,cf2..).join(); //等待所有 CompletableFuture 完成
  CompletableFuture.anyOf(cf1,cf2..).join(); //等待任意一个 CompletableFuture 完成
	doOtherThings();
  }
}
```

### 编排任务
使用CompletableFuture也是构建依赖树的过程。一个CompletableFuture的完成会触发另外一系列依赖它的CompletableFuture的执行：
<center><img src="pics/completeablefuture-1.png" alt=""></center>

这里描绘的是一个业务接口的流程，其中包括CF1\CF2\CF3\CF4\CF5共5个步骤，并描绘了这些步骤之间的依赖关系，每个步骤可以是一次RPC调用、一次数据库操作或者是一次本地方法调用等，在使用CompletableFuture进行异步化编程时，图中的每个步骤都会产生一个CompletableFuture对象，最终结果也会用一个CompletableFuture来进行表示。

根据CompletableFuture依赖数量，可以分为以下几类：零依赖、一元依赖、二元依赖和多元依赖。

#### 零依赖：CompletableFuture的创建
<center><img src="pics/completeablefuture-2.png" alt=""></center>

如上图红色链路所示，接口接收到请求后，首先发起两个异步调用CF1、CF2，主要有三种方式：
```
ExecutorService executor = Executors.newFixedThreadPool(5);
//1、使用runAsync或supplyAsync发起异步调用
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
  return "result1";
}, executor);
//2、CompletableFuture.completedFuture()直接创建一个已完成状态的CompletableFuture
CompletableFuture<String> cf2 = CompletableFuture.completedFuture("result2");
//3、先初始化一个未完成的CompletableFuture，然后通过complete()、completeExceptionally()，完成该CompletableFuture
CompletableFuture<String> cf = new CompletableFuture<>();
cf.complete("success");
```

#### 一元依赖：依赖一个CF
<center><img src="pics/completeablefuture-3.png" alt=""></center>

如上图红色链路所示，CF3，CF5分别依赖于CF1和CF2，这种对于单个CompletableFuture的依赖可以通过 `thenApply、thenAccept、thenCompose` 等方法来实现，代码如下所示：
```
CompletableFuture<String> cf3 = cf1.thenApply(result1 -> {
  //result1为CF1的结果
  //......
  return "result3";
});
CompletableFuture<String> cf5 = cf2.thenApply(result2 -> {
  //result2为CF2的结果
  //......
  return "result5";
});
```

#### 二元依赖：依赖两个CF
<center><img src="pics/completeablefuture-4.png" alt=""></center>

如上图红色链路所示，CF4同时依赖于两个CF1和CF2，这种二元依赖可以通过 `thenCombine` 等回调来实现，如下代码所示：
```
CompletableFuture<String> cf4 = cf1.thenCombine(cf2, (result1, result2) -> {
  //result1和result2分别为cf1和cf2的结果
  return "result4";
});
```

#### 多元依赖：依赖多个CF
<center><img src="pics/completeablefuture-5.png" alt=""></center>

如上图红色链路所示，整个流程的结束依赖于三个步骤CF3、CF4、CF5，这种多元依赖可以通过 `allOf` 或 `anyOf` 方法来实现，区别是当需要多个依赖全部完成时使用 `allOf` ，当多个依赖中的任意一个完成即可时使用 `anyOf` ，如下代码所示：
```
CompletableFuture<Void> cf6 = CompletableFuture.allOf(cf3, cf4, cf5);
CompletableFuture<String> result = cf6.thenApply(v -> {
  //这里的join并不会阻塞，因为传给thenApply的函数是在CF3、CF4、CF5全部完成时，才会执行 。
  result3 = cf3.join();
  result4 = cf4.join();
  result5 = cf5.join();
  //根据result3、result4、result5组装最终result;
  return "result";
});
```

## 注意的问题
如果不手动 complete 一个 CompletableFuture ，那么它会一直在运行
```
CompletableFuture<String> cf = new CompletableFuture<>();
cf.join();
```
