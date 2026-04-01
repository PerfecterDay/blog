# 定时任务
{docsify-updated}


### 定时任务/延时任务
`ScheduIedExecutorService` 接口为预定执行（Scheduled Execution ）或重复执行任务而设计的方法 。 它是一种允许使用线程池机制的 `java.util.Timer` 的泛化 。 `Executors`类的 `newScheduledThreadPool()` 和 `newSingleThreadScheduledExecutor()` 方法将返回实现了 `ScheduledExecutorService` 接口的对象，可以预定 `Runnable` 或 `Callable` 在初始的延迟之后只运行一次,也可以预定一个 `Runnable` 对象周期性地运行 。具体API如下：

1. `public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit);`： 预定在指定的时间之后执行任务
2. `public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit);`: 同上，只是提交 Callable 任务
3. `public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit);`: 预定在初始的延迟结束后，周期性地运行给定的任务 ，周期长度是固定 period
4. `public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit);`: 预定在初始的延迟结束后周期性地给定的任务，在一次调用完成和下一次调用开始之间有长度为 delay 的延迟

3和4的区别主要在于间隔时间的计算方式不同，3是固定以指定时间间隔运行而不管上一个任务是否执行完毕，也就是说时间间隔到了就执行。而4是从上一个任务执行完毕后计算时间间隔。

```
public interface ScheduledExecutorService extends ExecutorService 
{   
    @Test
    public void givenUsingExecutorService_whenSchedulingRepeatedTask_thenCorrect() 
        throws InterruptedException {
        TimerTask repeatedTask = new TimerTask() {
            public void run() {
                System.out.println("Task performed on " + new Date());
            }
        };
        ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor();
        long delay  = 1000L;
        long period = 1000L;
        executor.scheduleAtFixedRate(repeatedTask, delay, period, TimeUnit.MILLISECONDS);
        Thread.sleep(delay + period * 3);
        executor.shutdown();
    }
}
```

`Timer` 和 `ExecutorService` 解决方案的主要区别:
+ `Timer` 对系统时钟的变化很敏感；而 `ScheduledThreadPoolExecutor` 则不然。
+ `Timer　只有一个执行线程；而` `ScheduledThreadPoolExecutor` 可以配置任意数量的线程。
+ 在 `TimerTask` 中抛出的运行时异常会杀死线程，因此后续的计划任务不会继续运行；而在 `ScheduledThreadExecutor` 中，当前任务会被取消，但其他任务会继续运行。
