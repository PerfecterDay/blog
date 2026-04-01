# Java基础-显式锁
{docsify-updated}

- [Java基础-显式锁](#java基础-显式锁)
	- [Lock API](#lock-api)
	- [重入锁-ReentrantLock](#重入锁-reentrantlock)
	- [读写锁-ReentrantReadWriteLock](#读写锁-reentrantreadwritelock)
	- [LockSupport 工具](#locksupport-工具)


### Lock API
<center><img src="pics/lock-api.jpg" width="70%"></center>


### 重入锁-ReentrantLock
重入锁 `ReentrantLock` ，顾名思义，就是支持重进入的锁，它表示该锁能够支持一个线程对资源的重复加锁。除此之外，该锁的还支持获取锁时的公平和非公平性选择。

```
myLock.lock(); // a ReentrantLock object
try{
	critical section
}finally{
	myLock.unlock: // make sure the lock is unlocked even if an exception is thrown 
}
```

### 读写锁-ReentrantReadWriteLock
之前提到锁(如 `Mutex` 和 `ReentrantLock` )基本都是排他锁，这些锁在同一时刻只允许**一个线程**进行访问，而读写锁在同一时刻可以允许多个读线程访问，但是在写线程访问时，所有的读线程和其他写线程均被阻塞。读写锁维护了一对锁，一个读锁和一个写锁，通过分离读锁和写锁，使得并发性相比一般的排他锁有了很大提升。

下面是使用读/写锁的必要步骤:
1. 构造 一个RcentrantReadWriteL.ock对象:
```
private ReentrantReadWriteLock rwl = new ReentrantReadwriteLock; 
```

2. 抽取读锁和写锁:
```
private Lock readLock = rwl. readLock;
private Lock writeLock = rwl.writeLock;
```

3. 对所有的获取方法加读锁: 
```
public double getTotalBalance(){
	readLock.lock();
	try {
		...
	}finally { 
		readLock.unlock();
	}
}
```

4. 对所有的修改方法加写锁:
```
public void transfer(. . .) {
	writeLock.lock();
	try {
		. . . 
	}finally { 
		writeLock.unlock(); 
	}
}
```

### LockSupport 工具
当需要阻塞或唤醒一个线程的时候，都会使用LockSupport工具类来完成相应工作。 `LockSupport` 定义了一组的公共静态方法，这些方法提供了最基本的线程阻塞和唤醒功能，而 `LockSupport` 也成为构建同步组件的基础工具。

`LockSupport` 定义了一组以 `park` 开头的方法用来阻塞当前线程，以及 `unpark(Thread thread)` 方法来唤醒一个被阻塞的线程。

<center><img src="pics/lockSupport.jpg" width="70%"></center>