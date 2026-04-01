# Meters 种类
{docsify-updated}

## Counters
计数器只报告一个指标：计数。 计数器界面允许你以一个固定值递增，这个值必须是正数。

```
Normal rand = ...; // a random generator

MeterRegistry registry = ...
Counter counter = registry.counter("counter");

Flux.interval(Duration.ofMillis(10))
        .doOnEach(d -> {
            if (rand.nextDouble() + 0.1 > 0) {
                counter.increment();
            }
        })
        .blockLast();


Counter counter = Counter
    .builder("counter")
    .baseUnit("beans") // optional
    .description("a description of what this counter does") // optional
    .tags("region", "test") // optional
    .register(registry);
```

操作 Counters:
```
counter.increment();
counter.increment(10);
```

## Gauges
Gauges是一个用于获取当前值的句柄。 Gauges的典型示例包括集合或地图的大小，或处于运行状态的线程数量。将Gauges视为 "heisen-gauge"：一种只有在被观察到时才会发生变化的Gauges。 所有其他类型的仪表都会累积中间计数，然后将数据发送到度量后台。
```
List<String> list = registry.gauge("listGauge", Collections.emptyList(), new ArrayList<>(), List::size);
List<String> list2 = registry.gaugeCollectionSize("listSize2", Tags.empty(), new ArrayList<>());
Map<String, Integer> map = registry.gaugeMapSize("mapGauge", Tags.empty(), new HashMap<>());

Gauge gauge = Gauge
    .builder("gauge", myObj, myObj::gaugeValue)
    .description("a description of what this gauge does") // optional
    .tags("region", "test") // optional
    .register(registry);
```

一般来说，Gauge是自动获取的属性值，但是特殊情况下也可以手动操作 Gauge:
```
myGauge.set(27);
myGauge.set(11);
```

## Timers
计时器用于测量耗时和此类事件的频率。 定时器的所有实现都至少将耗时和事件计数报告为单独的时间序列，但也可以报告其他时间序列，具体取决于后端支持什么（最大值、百分位数、直方图）。   
典型的应用就是用来统计请求的处理时间。
```
Timer timer = Timer
    .builder("my.timer")
    .description("a description of what this timer does") // optional
    .tags("region", "test") // optional
    .register(registry);


SimpleMeterRegistry registry = new SimpleMeterRegistry();
Timer timer = registry.timer("app.event");
timer.record(() -> {
    try {
        TimeUnit.MILLISECONDS.sleep(15);
    } catch (InterruptedException ignored) {
    }
    });

timer.record(30, TimeUnit.MILLISECONDS);


Timer.Sample sample = Timer.start(registry);

// do stuff
Response response = ...

sample.stop(registry.timer("my.timer", "response", response.status()));
```

### `@Timed`注解
micrometer-core 模块包含一个 @Timed 注解，框架可使用该注解为特定类型的方法（例如为网络请求端点提供服务的方法）或所有方法添加定时支持。
首先需要注册一个 AspectJ 切面：
```
@Configuration
public class TimedConfiguration {
   @Bean
   public TimedAspect timedAspect(MeterRegistry registry) {
      return new TimedAspect(registry);
   }
}
```

然后在需要统计耗时的方法上加上 `@Timed` 注解即可：
```
@Service
public class ExampleService {

  @Timed
  public void sync() {
    // @Timed will record the execution time of this method,
    // from the start and until it exits normally or exceptionally.
    ...
  }

  @Async
  @Timed
  public CompletableFuture<?> async() {
    // @Timed will record the execution time of this method,
    // from the start and until the returned CompletableFuture
    // completes normally or exceptionally.
    return CompletableFuture.supplyAsync(...);
  }

}
```

## Distribution Summaries
分布汇总跟踪事件的分布。 它在结构上类似于计时器，但记录的值不代表时间单位。 例如，您可以使用分布汇总来测量访问服务器的请求的有效载荷大小。

```
DistributionSummary summary = registry.summary("response.size");

DistributionSummary summary = DistributionSummary
    .builder("response.size")
    .description("a description of what this summary does") // optional
    .baseUnit("bytes") // optional
    .tags("region", "test") // optional
    .scale(100) // optional
    .register(registry);

summary.record(responseSize);
```

可以选择提供一个缩放因子(scale)，在记录每个样本时将其乘以该因子。

Micrometer 预选的百分位数直方图桶都是 1 到 `Long.MAX_VALUE` 之间的整数。 目前， `minimumExpectedValue` 和 `maximumExpectedValue` 用于控制水桶集的核心数量。 如果检测到您的最小/最大值范围较小，并根据您的摘要范围缩放预选水桶域，那么我们就没有其他控制水桶数量的工具了。

## Long Task Timers
长任务定时器是一种特殊的定时器，可以在被测事件仍在运行时测量时间。 普通定时器只记录任务完成后的持续时间。
```
@Timed(value = "aws.scrape", longTask = true)
@Scheduled(fixedDelay = 360000)
void scrapeResources() {
    // find instances, volumes, auto-scaling groups, etc...
}

LongTaskTimer scrapeTimer = registry.more().longTaskTimer("scrape");
void scrapeResources() {
    scrapeTimer.record(() => {
        // find instances, volumes, auto-scaling groups, etc...
    });
}

LongTaskTimer longTaskTimer = LongTaskTimer
    .builder("long.task.timer")
    .description("a description of what this timer does") // optional
    .tags("region", "test") // optional
    .register(registry);
```
## Histograms and Percentiles
计时器和分布汇总支持收集数据以观察其百分位数分布(95线/99线/中位数...)。 查看百分位数主要有两种方法：
+ 百分位数直方图： Micrometer 将数值累加到底层直方图中，并将一组预定的桶发送到监控系统。 监控系统的查询语言负责根据该直方图计算百分位数。 目前，只有 Prometheus、Atlas 和 Wavefront 分别通过 `histogram_quantile` 、`:percentile` 和 `hs()` 支持基于直方图的百分位数近似值。 如果您的目标是 Prometheus、Atlas 或 Wavefront，请优先选择这种方法，因为您可以跨维度汇总直方图（通过汇总一组维度的桶值），并从直方图中得出可汇总的百分位数。
+ 客户端百分位数： Micrometer 会计算每个仪表 ID（名称和标签集）的百分位近似值，并将百分位值发送到监控系统。 这种方法不如百分位数直方图灵活，因为它无法汇总各标签的百分位数近似值。 尽管如此，对于不支持基于直方图的服务器端百分位数计算的监控系统来说，它还是能在一定程度上提供对百分位数分布的了解。

```
Timer.builder("my.timer")
   .publishPercentiles(0.5, 0.95) // median and 95th percentile
   .publishPercentileHistogram()
   .serviceLevelObjectives(Duration.ofMillis(100))
   .minimumExpectedValue(Duration.ofMillis(1))
   .maximumExpectedValue(Duration.ofSeconds(10))
```
+ `publishPercentiles()` ： 用于发布应用程序中计算的百分位数值。 这些值不可跨维度聚合。
+ `publishPercentileHistogram()` ： 用于发布适合在 Prometheus（通过使用 `histogram_quantile`）、Atlas（通过使用 `:percentile`）和 Wavefront（通过使用 `hs()`）中计算可聚合（跨维度）百分位数近似值的直方图。 对于 Prometheus 和 Atlas，Micrometer 会根据 Netflix 根据经验确定的生成器预设直方图结果中的桶数，该生成器能产生大多数实际计时器和分布汇总的合理误差范围。 默认情况下，生成器会生成 276 个桶，但 Micrometer 只包含那些在 `minimumExpectedValue` 和 `maximumExpectedValue` （包括在内）所设范围内的桶。 Micrometer 默认将计时器箝位在 1 毫秒到 1 分钟的范围内，因此每个计时器维度会产生 73 个直方图桶。 `publishPercentileHistogram` 对不支持可聚合百分位数近似值的系统没有影响。 这些系统不会发布直方图。
+ `serviceLevelObjectives()` : 用于发布累积直方图，其中的桶由 SLO 定义。 在支持可汇总百分位数的监控系统上与 `publishPercentileHistogram` 配合使用时，此设置会在发布的直方图中添加额外的桶。 在不支持可聚合百分位数的系统上使用时，此设置会导致发布的直方图仅包含这些数据桶。这个选项用来统计特定桶范围的样本数据。
+ `minimumExpectedValue/maximumExpectedValue` ： 控制 `publishPercentileHistogram` 发送的数据桶数量，并控制底层 HdrHistogram 结构的准确性和内存占用。



