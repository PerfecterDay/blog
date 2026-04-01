# java线程
{docsify-updated}

## 线程的三种创建方式
1. 继承自 `Thread` 类，重写 `run()`方法，并调用 `start()` 方法启动线程
2. 实现 `Runnable` 接口，并在实例化 `Thread` 对象时，将接口实现对象作为构造参数传递进去
3. 创建 `Callable` 接口的实现类，实现 `call()` 方法，使用 `FutureTask` 对象来包装  `Callable` 对象，构造 `Thread` 时，将 `FutureTask`对象作为参数传递进去。`FutureTask` 实现了 `Runnable` 接口。

三种方式的对比：
1. 继承 `Thread` 类后无法继承其它类，实现 `Runnable` 或 `Callable` 还可以继承其它类
2. `Callable` 接口与 `Runnable`接口相比，可以有返回值且可以抛出异常。

## 线程的一些属性
线程的常见属性包括：**线程所属的线程组**、**线程优先级**、**是否是Daemon线程**等。  

1. **优先级**  
	每个线程都有一个优先级，默认情况下，一个线程继承它的父线程的优先级。可以调用 `Thread.setPriority()` 方法为线程设置优先级。可以将优先级设置为 MIN_PRIORITY（1） 和 MAX_PRIORITY（10） 之间的
	任何值。但是线程的优先级高度依赖于操作系统和JVM平台实现的，不要把程序的正确性依赖于优先级。  

2. **守护线程**  
	Daemon线程是一种支持型线程，因为它主要被用作程序中**后台调度以及支持性工作**。这意味着，当一个Java虚拟机中**不存在非Daemon线程的时候，Java虚拟机将会退出**。可以通过调用`Thread.setDaemon(true)`将线程设置为Daemon线程，必须在**启动（调用 start）线程之前**调用。  
	Daemon线程被用作完成支持性工作，但是在Java虚拟机退出时,Daemon线程中的finally块并不一定会执行。

3. **未捕获异常处理器** 
	线程的run方法不能抛出任何被检测的异常，但是，不被检测的异常会导致线程终止。在这种情况下，线程就死亡了。
	Java线程提供了为捕获异常的处理器，该处理器必须实现一个 `Thread.UncaughtExceptionHandler` 接口，该接口只有一个 `void uncaughtException(Thread t, Throwable e)` 接口，可以使用线程实例对象的`setUncaughtExceptionHandler` 方法为线程安装处理器，也可以调用 Thread 类的静态方法 `setDefaultUncaughtExceptionHandler` 为所有线程安装一个默认的未捕获异常处理器。  
	如果没有调用上述方法为线程安装异常处理器，此时的处理器就是线程所属的 `ThreadGroup` 对象。

4. **`ThreadGroup`**  
   `ThreadGroup` 类实现 `Thread.UncaughtExceptionHandler` 接口。它的 `uncaughtException` 方法操作如下：
   1. 如果该线程组有父线程组，那么父线程组的 `uncaughtException` 方法被调用。
   2. 否则，如果 `Thread.getDefaultExceptionHandler` 方法返回一个非空的处理器，则调用该处理器
   3. 否则，如果 `Throwable` 是一个 `ThreadDeath` 实例，则什么也不做
   4. 否则，线程的名字以及 `Throwable` 的栈踪迹被输出到 `System.err` 上 


## 线程的状态
<center><img src="pics/thread-state.jpg" width=60% heght=60% /></center>

1. 阻塞  
	当一个线程获取一个内部的对象锁（而不是java.util.concurrent库中的锁），而该锁被其他线程持有，则该线程进入阻塞状态。当其他线程释放锁，且调度器允许本线程获取锁时，该线程变为非阻塞状态。
2. 等待  
	当线程等待另一个线程通知调度器一个条件时，它自己进入等待状态（主动进入）。被阻塞状态与等待状态是有很大的不同的。

<center><img src="pics/thread-state-transform.png" width=60% heght=60% /></center>

注意：上图中的 WAITING/TIME_WAITING 状态不能直接转换到 RUNNABLE 状态，而是先转换到 BLOCKED 状态，获取到锁后才能转换到 RUNNABLE 状态。如果这时候正好没有其它线程在争取锁，那么被唤醒的线程可以直接获取锁进入 RUNNABLE 状态。

```
/*NEW- thread object created, but not started.
RUNNABLE- thread is executing.
BLOCKED- waiting for monitor after calling wait() method.
WAITING- when wait() if called & waiting for notify() to be called. Also when join() is called.
TIMED_WAITING- when below methods are called:
   Thread.sleep
   Object.wait with timeout
   Thread.join with timeout
TERMINATED- thread returned from run() method.*/
public class ThreadBlockingState {
	public static void main(String[] args) throws InterruptedException {
		Object obj= new Object();
		Object obj2 = new Object();
		Thread3 t3 = new Thread3(obj,obj2,"A");
		Thread.sleep(1000);
		System.out.println("nm:"+t3.getName()+",state:"+t3.getState().toString()+
				",when Wait() is called & waiting for notify() to be called.");
		Thread4 t4 = new Thread4(obj,obj2,"B");
		Thread.sleep(3000);
		System.out.println("nm:"+t3.getName()+",state:"+t3.getState().toString()+",After calling Wait() & waiting for monitor of obj2.");
		System.out.println("nm:"+t4.getName()+",state:"+t4.getState().toString()+",when sleep() is called.");
	}

}
class Thread3 extends Thread{
	Object obj,obj2;
	int cnt;
	Thread3(Object obj, Object obj2, String number){
		super(number);
		this.obj = obj;
		this.obj2 = obj2;
		this.start();
	}

	@Override
	public void run() {
		super.run();
		synchronized (obj) {
			try {
				System.out.println("nm:"+this.getName()+",state:"+this.getState().toString()+",Before Wait().");
				obj.wait();
				System.out.println("nm:"+this.getName()+",state:"+this.getState().toString()+",After Wait().");
				synchronized (obj2) {
					cnt++;
				}
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}
}
class Thread4 extends Thread{
	Object obj,obj2;
	Thread4(Object obj, Object obj2, String number){
		super(number);
		this.obj = obj;
		this.obj2 = obj2;
		this.start();
	}

	@Override
	public void run() {
		super.run();
		synchronized (obj) {
			System.out.println("nm:"+this.getName()+",state:"+this.getState().toString()+",Before notify().");
			obj.notify();
			System.out.println("nm:"+this.getName()+",state:"+this.getState().toString()+",After notify().");
		}
		synchronized (obj2) {
			try {
				Thread.sleep(15000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}
}
```

### JVM 线程状态不=OS线程状态
测试程序：
```
public class NetDemo {
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(10000, 1);
        Socket clientSocket = serverSocket.accept();
        if (clientSocket.isConnected()) {
            System.out.println(clientSocket.getInetAddress().getHostAddress() + ":" + clientSocket.getPort());
        }
    }
}
```
Thread-dump 输出线程处于 `RUNNABLE` 状态：
<center><img src="pics/thread_dump_accept.png" alt=""></center>

但是，操作系统显示处于 Sleeping 状态：
<center><img src="pics/os_state.png" alt=""></center>

+ R:Running（正在运行）
+ S:Sleeping（可中断休眠，等待事件，如 I/O）
+ D:Uninterruptible sleep（通常等待 I/O，不可中断）
+ T:Stopped（暂停，可能被 SIGSTOP）
+ Z:Zombie（僵尸进程）

### BLOCKED 和 WAITING 区别
0. BLOCKED 的线程等待的是一个**锁**，往往是在请求锁的时候，因为锁已经被别的线程持有，而**被动**(一般是由调度器将其设置为BLOCKED)的进入 `BLOCKED` 状态。
1. 线程通过调用`wait()`方法进入 `WAITING` 状态是一种主动行为，此时会释放持有的锁，一般是等待另一个线程的信号，用来完成线程的同步；而 `BLOCKED` 状态是被动的，线程希望继续执行，但是锁被别的线程获取，必须等待别的线程释放锁。
2. 站在调度器的角度上，假如一个线程释放了锁，调度器调度是需要考虑 BLOCKED 队列中的线程让它们争用锁，但是不需要考虑 WAITING 队列中的线程。
<center><img src="pics/wait-blocked.png" width=40% heght=40%></center>

### wait 和 sleep 的区别
简单地说， `wait()` 是一个用于线程同步的实例方法。
它可以在任何对象上调用，因为它就定义在 `java.lang.Object` 中，但只能在**同步代码块中调用。它会释放对象上的锁**，以便另一个线程可以跳入并获取锁。
另一方面， `Thread.sleep()` 是一个静态方法，可以在任何上下文中调用。`Thread.sleep()` 会暂停当前线程，**但不会释放任何锁**。

### interrupt
中断标志或中断状态是线程的内部标志，在线程被中断时被设置。要设置它，只需在线程对象上调用 `thread.interrupt()`。

具体来说，当对一个线程，调用 `interrupt()` 时，
+ 如果线程处于被阻塞状态（例如处于sleep, wait, join 等状态），那么线程将立即退出被阻塞状态，并抛出一个InterruptedException异常。仅此而已。
+ 如果线程处于正常活动状态，那么会将该线程的中断标志设置为 true，仅此而已。被设置中断标志的线程将继续正常运行，不受影响。interrupt() 并不能真正的中断线程，需要被调用的线程自己进行配合才行。

线程有责任使用静态 `Thread.interrupted()` 或实例 `isInterrupted()` 方法定期检查中断状态。这些方法的区别在于，静态 `Thread.interrupted()` 会清除中断标志，而 `isInterrupted()` 不会。

## 线程之间的通信及同步

## volatile
`volatile` 基本可以做到两件事情：
1. 阻止编译器为了提高速度将一个变量缓存到寄存器内而不写回。
2. 阻止编译器调整操作volatile变量的指令顺序。

关键字 volatile 可以用来修饰字段（成员变量），就是告知程序**任何对该变量的访问均需要从共享内存中获取，而对它的改变必须同步刷新回共享内存，它能保证所有线程对变量访问的可见性。**

## ThreadLocal
[ThreadLocal](./java基础-线程安全的实现方法.md#线程本地存储-threadlocal)

## 等待通知机制
<center><img src="pics/wait-notify.jpg" width=50% heght=50%></center>

1. 调用 wait() 、notify() 或 notifyAll() 前需要先获取对象的锁
2. 调用 wait() 会立即释放对象锁（sleep()方法不会释放锁），线程由 RUNNING 变为 WAITING，并将当前线程放置到对象的等待队列
3. notify() 或 notifyAll() 调用后，等待线程依旧不会立即从 wait() 返回，需要调用 notify() 或 notifyAll()的线程释放锁之后，等待线程才有机会从 wait() 返回。
4. notify() 方法将等待队列中的一个等待线程从等待队列中移到同步队列中（阻塞于锁的队列），notifyAll() 则是将等待队列中的所有线程移到同步队列中，被移动的线程状态由 WAITING 变为 BLOCKED。
5. 从 wait() 方法返回的前提是获得对象的锁。

## 等待/通知的经典范式
等待方遵循如下原则：
1. 获取对象锁
2. 如果条件不满足，那么调用对象的 `wait()` 方法，被通知后仍要检查条件(`使用 while 循环`)，因为被通知后只是线程变成了可运行状态，在被调度运行之前不会立即获取到锁，其他线程有可能先得到调度获取锁并改变条件使得条件不再满足。
3. 条件满足则执行相应的逻辑。
对应的伪代码如下：
```
    synchronized(Object){
        while(条件不满足){
            Object.wait();
        }
        执行对应逻辑;
    }
```

通知方遵循如下原则：
1. 获取对象锁
2. 改变条件。
3. 通知所有等待在该对象上的线程。
对应的伪代码如下：
```
    synchronized(Object){
        改变条件;
        Object.notify();
    }
```

### 两个线程轮流打印0-100
```
public class UserCenterApp {

    private static int counter = 0;

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        Object lock = new Object();
        executorService.submit(() -> {
            while (true) {
                synchronized (lock) {
                    while (counter % 2 == 0) {
                        try {
                            lock.wait();
                        } catch (InterruptedException e) {
                            throw new RuntimeException(e);
                        }
                    }
                    if (counter > 100) return;
                    System.out.println("Thread 1" + ":" + counter++);
                    lock.notifyAll();
                }
            }
        });

        executorService.submit(() -> {
            while (true) {
                synchronized (lock) {
                    while (counter % 2 != 0) {
                        try {
                            lock.wait();
                        } catch (InterruptedException e) {
                            throw new RuntimeException(e);
                        }
                    }
                    if (counter > 100) return;
                    System.out.println("Thread 2" + ":" + counter++);
                    lock.notifyAll();
                }
            }
        });

        executorService.shutdown();
    }
}
```