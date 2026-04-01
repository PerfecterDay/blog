# Java实现 EventLoop
{docsify-updated}
> https://hackernoon.com/implementing-an-event-loop-in-java-for-fun-and-profit

- [Java实现 EventLoop](#java实现-eventloop)

```
interface EventLoop {
  void start();

  void stop();
}

class BusyWaitingEventLoop implements EventLoop {
  private final Events events;
  private final Script script;
  private final AtomicBoolean alive;

  BusyWaitingEventLoop(final Events events, final Script script) {
    this(events, script, new AtomicBoolean(true));
  }
  
  BusyWaitingEventLoop(final Events events, final Script script, final AtomicBoolean alive) {
    this.events = events;
    this.script = script;
    this.alive = alive;
  }

  @Override
  public void start() {
    while (alive.get()) {
      events.next().ifPresent(event -> event.trigger(script));
    }
  }

  @Override
  public void stop() {
    alive.set(false);
  }
}

class MultithreadedEventLoop implements EventLoop {
  private final EventLoop origin;
  private final Integer nThreads;
  private final ExecutorService executorService;

  MultithreadedEventLoop(final EventLoop origin, final Integer nThreads, final ExecutorService executorService) {
    this.origin = origin;
    this.nThreads = nThreads;
    this.executorService = executorService;
  }

  @Override
  public void start() {
    for (var i = 0; i < nThreads; i++) {
      executorService.execute(origin::start);
    }
  }

  @Override
  public void stop() {
    origin.stop();
    shutdownExecutorService();
  }

  private void shutdownExecutorService() {
    // Java specific code
  }
}


interface Events {
  Optional<Event> next();
}

public class EventsContainer implements Events {

	BlockingQueue<Event> queue = new ArrayBlockingQueue(100);
	@Override
	public Optional<Event> next() {
        try {
            return Optional.ofNullable(queue.take());
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }

	public void addEvent(Event e) throws InterruptedException {
		queue.put(e);
	}
}

interface Event {
  void trigger(Script script);
}

public class EventImpl implements Event {
	@Override
	public void trigger(Script script) {
		script.run(null, (e)->{
			System.out.println("aaa");
		},(e)->{
			System.out.println("bbb");
		});
	}
}

interface Script {
  void run(JsonObject properties, Consumer<Instant> onSuccess, Consumer<Throwable> onFailure);
}

class AsyncScript implements Script {
  private final Script origin;
  private final ExecutorService executorService;

  AsyncScript(final Script origin, final ExecutorService executorService) {
    this.origin = origin;
    this.executorService = executorService;
  }

  @Override
  public void run(final JsonObject properties, final Consumer<Instant> onSuccess, final Consumer<Throwable> onFailure) {
    if (!executorService.isShutdown()) {
      executorService.execute(() -> origin.run(properties, onSuccess, onFailure));
    }
  }
}
```