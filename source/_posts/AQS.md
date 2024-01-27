---
title: AQS
date: 2024-01-27 17:36:47
tags:
- Java
- 锁
---
# CLH

CLH 锁是一种 **基于链表的自旋锁，每个线程自旋的变量都是不同的（没有争抢同一个变量）**。

**当前节点不断轮询前驱的状态**，如果发现前驱释放了锁就结束自旋。
<!-- more -->
关于 CLH 相关知识比较完整的可以参考这篇[文章](https://coderbee.net/index.php/concurrent/20131115/577)。

```Java
public class CLHLock {  
	public static class CLHNode {  
		private volatile boolean isLocked = true; // 默认下一个线程需要等待锁  
	}  
	  
	private volatile CLHNode tail;  
	private static final AtomicReferenceFieldUpdater<CLHLock, CLHNode> UPDATER = 
							AtomicReferenceFieldUpdater  
							. newUpdater(CLHLock.class, CLHNode.class , "tail" );  
	  
	public void lock(CLHNode currentThread) {  
		CLHNode preNode = UPDATER.getAndSet( this, currentThread);  
		if(preNode != null) {//已有线程占用了锁，进入自旋  
			//noinspection StatementWithEmptyBody  
			while(preNode.isLocked);  
		}  
	}  
	  
	public void unlock(CLHNode currentThread) {  
		// 这个判断似乎有点多余
		if (UPDATER.compareAndSet(this, currentThread, null)) {  
			// 如果队列里只有当前线程，则释放对当前线程的引用（for GC）。  
			return;  
		}  
		// 还有后续线程  
		currentThread.isLocked = false ;// 改变状态，让后续线程结束自旋  
	}  
}
```

# AQS（AbstractQueuedSynchronizer）

先整体过一遍 `AQS` 中比较重要的属性以及内部类，代码中为了保持简洁会删减一些无关的东西

```Java
// AQS 作为一个框架提供。其子类可以是独占/共享、公平/非公平，子类应该被定义为非公共内部类
// {@link ReentrantLock.Sync}
public abstract class AbstractQueuedSynchronizer {

	// 等待队列节点类，该队列是 “CLH” 锁定队列的变体, 虽然 CLH 锁通常用于自旋锁
	// 采用与 CLH 类似的策略，在前驱节点中保存当前节点的一些信息，每个线程会被包装在一个节点中
    // 如果要加入队列，只需要尝试将节点变成新的尾部
	//      @see AbstractQueuedSynchronizer#addWaiter Call compareAndSetTail
	// 如果节点位于队列中的第一个（不包括 head)，则可能会尝试获取锁，如果成功则成为新的head
	//      @see AbstractQueuedSynchronizer#acquireQueued Call setHead
	static final class Node {
		// SIGNAL 
		//        当前节点在获取锁时发现前驱节点为该值，当前节点会 park 当前线程
        //              @see AbstractQueuedSynchronizer#acquireQueued Call shouldParkAfterFailedAcquire
		//        当前节点在释放锁时发现当前节点为该值，当前节点会 unpark 后续节点的线程
        //              @see AbstractQueuedSynchronizer#release Call unparkSuccessor
        // CANCELLED
        //        当前节点被取消
        // CONDITION
        //        当前节点正在条件队列中，直到条件改变前都不会用作同步队列，暂时忽略
        // PROPAGATE 略
		volatile int waitStatus;

        volatile Node prev;
        volatile Node next;
        volatile Thread thread;
        // 条件队列，Condition 相关，暂时忽略
        Node nextWaiter; 
	}

	// 头结点
    // CLH 队列的头尾节点不会在初始就创建，而是在第一次发生争用时构造节点并设置
    // 对于头结点，存在两种情况：
    //      1.当前持有锁的线程对应的节点，这种情况比较显然，也就是在
    //              @see AbstractQueuedSynchronizer#setHead
    //      2.假设有 thread1 第一次成功获取锁，此时没有争用，不需要 CLH 队列；
    //        然后 thread2 获取锁，产生争用，发现 head/tail 为空，于是初始化队列
    //        并加入，等待 thread1 唤醒；
    //        此时的 head 的节点实际上只是个虚拟节点，并不代表 thread1
    //              @see AbstractQueuedSynchronizer#initializeSyncQueue
	private transient volatile Node head;
	
	// 阻塞的尾节点，每个新的节点进来，都成为新的tail
	private transient volatile Node tail;
	
	// 当前锁的状态，0代表没有被占用，大于 0 代表锁重入的次数
    // 注意与 Node#waitStatus 区分开来
	private volatile int state;
}
```

## ReentrantLock
> Lock取代了synchronized方法和语句的使用，而Condition则取代了对象监视方法的使用

对于具体的 `Lock` ，大多情况下的用法为：
```Java  
 l.lock();  
 try {    
    // access the resource protected by this lock  
 } finally {
    l.unlock();  
 }
```

`ReentrantLock` 在内部基于 `Sync` 实现同步控制，而 `Sync` 正是 `AQS` 的一种具体实现。

我们以 `ReentrantLock` 的非公平锁为例，讲述 `lock()` 和 `unlock()` 的整体流程

### lock()

![](../img/微信图片_20240127211035.png)

```Java
    public final void acquire(int arg) {
        // 尝试获取锁, 如果获取成功免去队列操作
        if (!tryAcquire(arg) &&
            // addWaiter加入阻塞队列  acquireQueued直到成功获取锁或者被中断时返回, 返回值为中断状态
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

 	...

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
    // 如果当前锁没被持有，尝试通过一次 CAS 更改状态为持有，整体操作轻量级 
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 初步判断没有锁
            // 当前线程尝试获取锁
            if (compareAndSetState(0, acquires)) {
                // 成功
                setExclusiveOwnerThread(current);
                return true;
            }
        } else if (current == getExclusiveOwnerThread()) {
            // 重入
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded"); // 溢出
            setState(nextc);
            return true;
        }
        // 失败
        return false;
    }

    ...

    // 将当前结点加入到阻塞队列的尾部，mode = Node.EXCLUSIVE
    private Node addWaiter(Node mode) {
        // node 后继指向 mode
        Node node = new Node(mode);

        for (;;) {
            // 不断尝试
            Node oldTail = tail;
            if (oldTail != null) {
                // node 前驱指向 oldTail
                node.setPrevRelaxed(oldTail);
                // 尝试将当前结点设置为 tail
                if (compareAndSetTail(oldTail, node)) {
                    // 成功
                    // oldTail 的后继指向 node
                    oldTail.next = node;
                    return node;
                }
            } else {
                // 队列为空
                initializeSyncQueue();
            }
        }
    }

    ...

    // 自旋
    // 前驱为头节点时尝试获取锁
    // 检查是否应该 park
    // 返回值为中断状态
    final boolean acquireQueued(final Node node, int arg) {
        boolean interrupted = false;
        try {
            for (;;) {
                // 前驱
                final Node p = node.predecessor();
                // 如果前驱是头节点，尝试获取锁
                if (p == head && tryAcquire(arg)) {
                    // 当前线程持有锁，为新的头节点
                    setHead(node);
                    p.next = null; // help GC
                    return interrupted;
                }
                // 检查是否应该阻塞 (park)
                if (shouldParkAfterFailedAcquire(p, node))
                    // 阻塞并且检测终端状态
                    interrupted |= parkAndCheckInterrupt();
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            if (interrupted)
                selfInterrupt();
            throw t;
        }
    }

    ...

    // 检查并更新获取失败的节点的状态。如果线程应该阻塞则返回 true。这是轮询中的主要信号控制
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            // 前驱已经设置状态为 SIGNAL，因此当前节点可以安全地停放。
            return true;
        if (ws > 0) {
            /*
             * 前驱被取消了，不断往前直到找到第一个未被取消的节点（头节点必然未被取消）
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            // 尝试将前驱设置状态为 SIGNAL
            // 前驱节点的 waitStatus=-1 依赖于后继节点设置
            pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
        }
        return false;
    }
```
### unlock()
`unlock` 的逻辑相对比较简单
```Java
public void unlock() {
	sync.release(1);
}
public final boolean release(int arg) {
	// 尝试释放锁，当计数器为 0 时成功
	if (tryRelease(arg)) {
		Node h = head;
		if (h != null && h.waitStatus != 0)
			// 唤醒后继
			unparkSuccessor(h);
		return true;
	}
	return false;
}

...

// 释放锁，不会失败
protected final boolean tryRelease(int releases) {
	int c = getState() - releases;
	if (Thread.currentThread() != getExclusiveOwnerThread())
		// 只有持有锁的线程才能释放锁
		throw new IllegalMonitorStateException();
	boolean free = false;
	if (c == 0) {
		// 计数器为 0 , 释放锁
		free = true;
		setExclusiveOwnerThread(null);
	}
	setState(c);
	return free;
}

...

// unpark 后继节点，如果 后继节点被取消了或者为空 就从尾节点向前找第一个未被取消的
private void unparkSuccessor(Node node) {
	/*
	* If status is negative (i.e., possibly needing signal) try
	* to clear in anticipation of signalling.  It is OK if this
	* fails or if status is changed by waiting thread.
	*/
	int ws = node.waitStatus;
	if (ws < 0)
		// 清除
		node.compareAndSetWaitStatus(ws, 0);

	/*
	* Thread to unpark is held in successor, which is normally
	* just the next node.  But if cancelled or apparently null,
	* traverse backwards from tail to find the actual
	* non-cancelled successor.
	*/
	// unpark 后继节点。但如果后继被取消或者 **后继为null** 
	// 则从 **尾部向前** 遍历以找到实际的未取消后继者。
	Node s = node.next;
	if (s == null || s.waitStatus > 0) {
		s = null;
		for (Node p = tail; p != node && p != null; p = p.prev)
			if (p.waitStatus <= 0)
				s = p;
	}
	if (s != null)
		// 唤醒后继节点
		LockSupport.unpark(s.thread);
}
```