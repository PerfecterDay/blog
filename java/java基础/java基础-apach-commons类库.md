# Apache commons 类库

> https://commons.apache.org/

Apache Commons 是一个 Apache 项目，重点关注可重用 Java 组件的各个方面。
Commons Proper 致力于实现一个主要目标：创建和维护可重复使用的 Java 组件。 在这里，整个 Apache 社区的开发人员可以共同开发项目，供 Apache 项目和 Apache 用户共享。Commons 开发人员将努力确保其组件对其他库的依赖性降到最低，以便这些组件可以轻松部署。 此外，Commons 组件还将尽可能保持其接口的稳定性，以便 Apache 用户（包括其他 Apache 项目）可以实施这些组件，而不必担心将来会有变化。

```
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-collections4</artifactId>
    <version>4.5.0-M2</version>
</dependency>

<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.17.0</version>
</dependency>
```

Apche commons 常见工具库的使用教程： https://www.baeldung.com/apache-commons-series

## Commons-configuration 自动重载配置
> https://commons.apache.org/proper/commons-configuration/userguide/howto_reloading.html

如果应用程序对可用性有特殊要求，则可能希望无需重启即可更改配置文件。 应用程序应能自动检测到此类更改并做出相应的反应。 为自动重载提供支持非常困难，因为应用程序可能对如何以及何时执行重载有非常具体的需求。 此外，重载不应仅限于基于文件的配置，还应适用于其他配置源，例如数据库中保存的配置设置。 因此，Commons Configuration 提供了一些处理重载的通用类和接口。

Commons Configuration 定义的重载机制涉及多个组件，这些组件共同检测配置源的变化，并触发实际的重载操作。

### ReloadingDetector
一个基本组件是重载检测器，由 `ReloadingDetector` 接口定义。 该对象负责检测外部配置源的变化。 示例实现可以检查特定文件的最后修改数据是否发生变化。 请注意，重载检测器不会主动监控配置源的变化；它只需在触发时能检测到变化即可。 这一点在重载检测器接口定义的方法中有所体现：
+ `boolean isReloadingRequired()`: 调用该方法可触发检查。 检测器必须确定被监控源是否发生了变化，并返回一个布尔标志作为结果。
+ `void reloadingPerformed()`: 该方法会在执行重载操作后被调用。 该方法让检测器有机会重置自己，以便检测到相关配置源的新变化。

### ReloadingController
下一个参与重新加载的组件是 `ReloadingController` 类的实例。 这是一个功能齐全的类，执行通用重载协议，用于执行重载检查（基于外部触发）并做出相应反应。 是否需要重新加载的实际检查工作委托给了与控制器关联的重新加载检测器（`ReloadingDetector`）。 当检测器报告发生变化时，就会向已注册的重载监听器发送相应的通知。 与重载检测器一样，重载控制器不会主动监控特定资源；它有一个  `checkForReloading()` 方法，必须调用该方法才能触发重载检查。 如果该方法返回 true，控制器就会进入所谓的重新加载状态。 这意味着已检测到需要重新加载，现在必须进行重新加载。 通常情况下，这是由在控制器上注册的 `ReloadingListener` 对象完成的。 只要控制器处于重新加载状态，就不会检测到相关 `ReloadingDetector` 监测到的配置源的进一步更改。 必须手动调用`resetReloadingState()` 方法才能终止该状态并检测到进一步的变化。

目前讨论的组件只能按需执行重载检查。 为了实现自动重新加载，必须确保定期调用 `ReloadingController` 的 `checkForReloading()` 方法，或者至少在发生可能影响受监控配置源的事件时调用 `checkForReloading()` 方法。 重载机制的这一部分很难以通用形式提供；在这一领域，需求和用例往往非常具体。 因此，Commons Configuration 只提供了一个非常简单的、基于计时器的解决方案；在简单的情况下，这可能就足够了，而对于更复杂的需求，可能需要创建一个自定义组件来触发重载检查。