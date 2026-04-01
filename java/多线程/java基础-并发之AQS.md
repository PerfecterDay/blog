# AQS
{docsify-updated}

## <a href="/java/多线程/aqs.pdf" target="_blank">设计与实现</a>
acquire：
```
while (synchronization state does not allow acquire) {
    enqueue current thread if not already queued;
    possibly block current thread;
}
dequeue current thread if it was queued;
```

release：
```
update synchronization state;
if (state may permit a blocked thread to acquire)
    unblock one or more queued threads;
```


Support for these operations requires the coordination of three
basic components:
+ Atomically managing synchronization state
+ Blocking and unblocking threads
+ Maintaining queues



`AbstractQueuedSynchronizer` 队列同步器是用来构建各种锁或者其他同步组件的基础。它使用一个 int 型变量来表示同步状态，通过内置的FIFO队列来完成线程的排队工作，Doug Lea 期望它成为实现大部分同步需求的基础。

## 对比PV操作
信号量是 Dijkstra 提出的用于解决进程同步的有效工具。信号量是一个数据结构以及对其的操作。除初始化外，仅能通过两个标准的原子操作wait(S)和 signal(S)来访问。**两个语句对信号量状态的改变是安全的**。

信号量值的含义是表示系统值某类资源的数目。

1. P(S)/wait(S): 每次 P(S) 操作，意味着进程请求一个单位的该类资源，使系统可供分配的该类资源数减少一个。 
   1. 将信号量S的值减1，即S.value:=S.value-1；
   2. 当S.value<0时，表示该类资源分配完毕，进程调用block原语，进行自我阻塞，放弃处理机，并插入到信号量链表中。 
2. V(S)/signal(S): 每次 V(S) 操作，表示执行进程释放一个单位资源，使系统中可供分配的该类资源数增加一个.
   1. 将信号量S的值加1，即S.value:=S.value+1； 
   2. 如果S.value<=0，表示在该信号量链表中，仍有等待该资源的进程被阻塞，故还应调用wakeup原语，将链表中的第一个等待进程唤醒。

初始设置的 S.value值，代表系统含有某类资源的数目，也限制了并发获取该资源的进程的数量。
如果 S.value > 0, 表示该资源还有，可以被使用。
如果 S.value = 0, 表示该资源正好被进程用完，也没有进程阻塞你等待在该资源上。
如果 S.value < 0, 表示该资源用完了，且有进程在阻塞等待该资源。

可以把 `AbstractQueuedSynchronizer` 类比成信号量数据结构的一种实现。但是比信号量更灵活。

## AQS原理

### 同步队列
同步器依赖内部的同步队列(双向FIFO队列)来完成同步状态的管理。

独占式同步状态获取流程如下：
<center><img src="pics/AQS-Exclusive.png" height=40% width=40%></center>

首先调用自定义同步器的 `tryAcquire(int arg)` 方法，该方法应该保证线程安全地获取同步状态，如果获取失败，则构造同步节点（独占式 `NODE.EXCLUSIVE` ,同一时刻只能有一个线程成功获取同步状态），并通过 `addWaiter(Node node)` 方法将该节点加入到同步队列的尾部，最后调用 `acquireQueued(final Node node, int arg)` 方法，使得该节点（线程）以“死循环”的方式获取同步状态。如果获取不到则阻塞节点中的线程，被阻塞线程的唤醒主要依靠前驱节点的出队或阻塞线程被中断来实现。
```
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
    
    private Node enq(final Node node) {
        //死循环方式将节点加入队尾
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```
节点进入同步队列之后，就进入了一个自旋的过程，每个节点（或者说每个线程）都在自省地观察，当条件满足，获取到了同步状态，就可以从这个自旋过程中退出，否则依旧留在这个自旋过程中（并会阻塞节点的线程）：
``` 
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            //死循环方式获取同步状态，失败则阻塞
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

锁的释放：
```
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
因为只有一个线程获取到了锁，所以，释放锁不需要同步。释放锁后，将会唤醒队列中的下一个节点。