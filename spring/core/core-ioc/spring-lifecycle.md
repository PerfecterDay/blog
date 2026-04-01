# Spring Lifecycle 与 SmartLifecycle
{docsify-updated}

在使用Spring的过程中，我们通常会用 `@PostConstruct` 和 `@PreDestroy` 在Bean初始化或销毁时执行一些操作，这些操作属于Bean生命周期级别的。也就是每次 bean 创建或销毁的时候都会执行这些指定操作。

那么，就存在一些遗漏的场景，比如我们想在**容器本身的生命周期**（比如容器启动、停止）的事件上做一些工作，很典型的就是Spring Boot中启动内嵌的Web容器。该怎么办？请查看 `WebServerStartStopLifecycle` 类

这就需要用到Spring提供的另外一个接口 `Lifecycle` 。这篇文件就介绍一下 `Lifecycle` 接口，以及比它更聪明（Smart）的 `SmartLifecycle` 。


## Lifecycel
```
public interface Lifecycle {
	void start();
	void stop();
	boolean isRunning();
}
```


## SmartLifecycle
```
public interface SmartLifecycle extends Lifecycle, Phased {
	int DEFAULT_PHASE = Integer.MAX_VALUE;

	default boolean isAutoStartup() {
		return true;
	}

	default void stop(Runnable callback) {
		stop();
		callback.run();
	}

	@Override
	default int getPhase() {
		return DEFAULT_PHASE;
	}

}
```

## 原理
<center><img src="pics/lifecycle2.png"></center>

从调用帧可以看出，`AbstractApplicationContext` 的 `finishRefresh()` 方法中会调用 `DefaultLifecycleProcessor` 的 `onRefresh()` 方法。

```
public class DefaultLifecycleProcessor implements LifecycleProcessor, BeanFactoryAware {
	...
	public void onRefresh() {
		startBeans(true);
		this.running = true;
	}

	public void start() {
		startBeans(false);
		this.running = true;
	}
	...
}
```

`onRefresh()` 方法调用 `startBeans()` 方法时，传入的是 `true` ；而 `start()` 调用 `startBeans()` 方法时，传入的是 `false` 。我们再看 `startBeans` 方法：

<center><img src="pics/lifecycle.png" width="80%"></center>

可以发现只有传入 `false` 参数或者 bean 是 `SmartLifecycle` 类型实例且是自动启动时，才会调用 `Lifecycle` 的 `start()` 方法。所以，通过上面的 `DefaultLifecycleProcessor` 的 `start` 方法调用时(通过 `AbstractApplicationContext` 的 `start()`方法会调用)， `Lifecycle/SmartLifecycle` bean 会被同时调用；而 `onRefresh()` 方法只会调用 `SmartLifecycle` bean 的 `start` 方法。

这也就是为什么说需要显示调用 `ApplicationContext` 的 `start()` 方法， `Lifecycle` bean 才能生效的原因。

## 与spring 事件机制的一些区别
用 Spring 的事件机制（比如监听 ContextRefreshedEvent）也可以实现类似的“启动时做点事”的功能。但 `Lifecycle/SmartLifecycle` 和事件监听是关注点不同的机制，各有适用场景。
| 能力 | Lifecycle/SmartLifecycle | 事件监听 |
|------|--------------------------|----------|
| 容器启动时运行 | ✅ 是 | ✅ 是 |
| 容器关闭时运行 | ✅ 自动触发 stop() | ❌ 需要另写 ContextClosedEvent，没有生命周期感知，容器关闭时无法自动做清理 |
| 组件状态控制 | ✅ 有 isRunning() 等 | ❌ 没有 |
| 启动顺序控制 | ✅ getPhase() 控制 | ❌ 不行，不能控制执行顺序 |
| 自动启动控制 | ✅ isAutoStartup() | ❌ 不行，没有自动 “start/stop” 的语义，更多是事件通知 |
| 生命周期集成（如 DefaultLifecycleProcessor） | ✅ 是 | ❌ 不是 |

监听 Spring 事件是“响应容器事件的方式”，而 `Lifecycle/SmartLifecycle` 是“参与到 Spring 生命周期管理中的一等公民”。  
如果你有组件需要控制启动/关闭过程、顺序和状态，用 `SmartLifecycle` 会更合适、更专业。

如果你只是一次性做点初始化逻辑，用事件监听就够了；但如果你写的是服务组件、后台任务、连接器、MQ、Socket 之类的 —— 用 SmartLifecycle 更稳、更易控。