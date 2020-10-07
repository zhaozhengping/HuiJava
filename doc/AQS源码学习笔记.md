## 1 前言

​    	AQS（全称AbstractQueuedSynchronizer，位于java.util.concurrent.locks）提供了原子式管理同步状态、阻塞和唤醒线程功能以及队列简单模型框架。Java中的大部分同步类（CountDownLatch、Semaphore、ReentrantLock、ReadWriteLock）都是基于AQS实现的，也是备受面试官青睐的题目。



## 2 AQS数据结构

### 2.1 属性和方法

* 同步状态：**state**，由volatile修饰，标识当前临界资源状态。

```java
//The synchronization state
private volatile int state;

private transient volatile Node head;
private transient volatile Node tail;

```

访问state的几个方法，由final修饰。

| 方法名称                                                     | 方法注释                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| protected final int getState()                               | Returns the current value of synchronization state.          |
| protected final void setState(int newState)                  | Sets the value of synchronization state.                     |
| protected final boolean compareAndSetState(int expect, int update) | Atomically sets synchronization state to the given updated value if the current state value equals the expected value. |

* 基于**Node**(内部类)的同步队列，这是个**FIFO**的双向队列

  ```java
  static final class Node {
          /** Marker to indicate a node is waiting in shared mode  */
      	//共享模式
          static final Node SHARED = new Node();
          /** Marker to indicate a node is waiting in exclusive mode */
      	//独占模式
          static final Node EXCLUSIVE = null;
  
          /** waitStatus value to indicate thread has cancelled */
          static final int CANCELLED =  1;
          /** waitStatus value to indicate successor's thread needs unparking */
          static final int SIGNAL    = -1;
          /** waitStatus value to indicate thread is waiting on condition */
          static final int CONDITION = -2;
          /**
           * waitStatus value to indicate the next acquireShared should
           * unconditionally propagate
           */
          static final int PROPAGATE = -3;
  
          /**
           * Status field, taking on only the values:
           *
           * The values are arranged numerically to simplify use.
           * Non-negative values mean that a node doesn't need to
           * signal. So, most code doesn't need to check for particular
           * values, just for sign.
           *
           * The field is initialized to 0 for normal sync nodes, and
           * CONDITION for condition nodes.  It is modified using CAS
           * (or when possible, unconditional volatile writes).
           */
      	//当前节点在队列中的状态
          volatile int waitStatus;
  
          //前驱
          volatile Node prev;
          //后继
          volatile Node next;
          //表示处于该节点的线程
          volatile Thread thread;
          //指向下一个处于CONDITION状态的节点
          Node nextWaiter;
  
          /**
           * Returns true if node is waiting in shared mode.
           */
          final boolean isShared() {
              return nextWaiter == SHARED;
          }
  
          /**
           * Returns previous node, or throws NullPointerException if null.
           * Use when predecessor cannot be null.  The null check could
           * be elided, but is present to help the VM.
           *
           * @return the predecessor of this node
           */
          final Node predecessor() throws NullPointerException {
              Node p = prev;
              if (p == null)
                  throw new NullPointerException();
              else
                  return p;
          }
  
          Node() {    // Used to establish initial head or SHARED marker
          }
  
          Node(Thread thread, Node mode) {     // Used by addWaiter
              this.nextWaiter = mode;
              this.thread = thread;
          }
  
          Node(Thread thread, int waitStatus) { // Used by Condition
              this.waitStatus = waitStatus;
              this.thread = thread;
          }
      }
  ```
  
  
  
  翻译waitStatus注释可以得到下面几个枚举值（英文不错的朋友建议直接读代码注释）。
  
  | 枚举值    | 注释                                            |
  | --------- | ----------------------------------------------- |
  | 0         | Node被初始化默认值                              |
  | CANCELLED | 取值1，表示因超时或中断，获取资源的请求已取消。 |
  | CONDITION | 取值-2，表示节点在等待队列中，节点线程等待唤醒  |
  | PROPAGATE | 取值-3，共享模式                                |
  | SIGNAL    | 取值-1，等待被唤醒                              |
  
  

### 2.2 原理概述

​		基于以上基本属性和方法，AQS工作原理通过修改state的值，来获取同步状态。如果某个线程成功的将state的值从0修改为1，表示成功的获取了同步状态。 这个修改的过程是通过CAS完成的，所以可以保证线程安全。如果修改state失败，则会将当前线程加入到AQS的队列中，并阻塞线程。示意图如下：

![image-20201007000101064](C:\Users\zhaozp\AppData\Roaming\Typora\typora-user-images\image-20201007000101064.png)



## 3 获取和释放同步状态

AQS实现获取和释放同步状态，是通过修改state的值实现的，主要有以下几个方法。

| 方法名称                                                     | 注释                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| protected boolean isHeldExclusively()                        | Returns true if synchronization is held exclusively with  respect to the current (calling) thread. |
| protected boolean tryAcquire(int arg)                        | Attempts to acquire in exclusive mode. This method should query  if the state of the object permits it to be acquired in the  exclusive mode, and if so to acquire it.   --独占模式 获取 |
| protected boolean tryRelease(int arg)                        | This method is always invoked by the thread performing release.  --独占模式 释放 |
| protected int tryAcquireShared(int arg)                      | Attempts to acquire in shared mode. This method should query if  the state of the object permits it to be acquired in the shared  mode, and if so to acquire it.  -- 共享模式 获取 |
| protected boolean tryReleaseShared(int arg)                  | Attempts to set the state to reflect a release in shared mode.  -- 共享模式 释放 |
| public final boolean tryAcquireNanos(int arg, long nanosTimeout) | Attempts to acquire in exclusive mode, aborting if interrupted, and failing if the given timeout elapses. --超时 独占模式 获取 |
| public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout) | Attempts to acquire in shared mode, aborting if interrupted, and  failing if the given timeout elapses. --超时机制 共享模式 获取 |



> 代码中有两套方法，tryAcquire 和acquire 有什么区别（笔者经常傻傻分不清楚）？

**tryAcquire**是尝试获取，AQS直接抛出*UnsupportedOperationException*，留给子类（例如：ReentrantLock）扩展，而**acquire**才是真正获取资源的方法实现。



### 3.1 抢占式获取和释放资源

* 抢占式获取同步状态：**tryAcquire** 和 **acquire**

  ```java
  /**
       * Attempts to acquire in exclusive mode. This method should query
       * if the state of the object permits it to be acquired in the
       * exclusive mode, and if so to acquire it.
       *
       * <p>This method is always invoked by the thread performing
       * acquire.  If this method reports failure, the acquire method
       * may queue the thread, if it is not already queued, until it is
       * signalled by a release from some other thread. This can be used
       * to implement method {@link Lock#tryLock()}.                    -- 可用于实现tryLock
       *
       * <p>The default
       * implementation throws {@link UnsupportedOperationException}.
       *
       * @param arg the acquire argument. This value is always the one
       *        passed to an acquire method, or is the value saved on entry
       *        to a condition wait.  The value is otherwise uninterpreted
       *        and can represent anything you like.
       * @return {@code true} if successful. Upon success, this object has
       *         been acquired.
       * @throws IllegalMonitorStateException if acquiring would place this
       *         synchronizer in an illegal state. This exception must be
       *         thrown in a consistent fashion for synchronization to work
       *         correctly.
       * @throws UnsupportedOperationException if exclusive mode is not supported
       */
      protected boolean tryAcquire(int arg) {
          throw new UnsupportedOperationException();
      }
  ```

  ``` java
  /**
       * Acquires in exclusive mode, ignoring interrupts.  Implemented  -- 独占模式，忽略中断，
       * by invoking at least once {@link #tryAcquire},  -- 先至少调用一次tryAcquire 成功返回后，再调用该方法
       * returning on success.  Otherwise the thread is queued, possibly -- 否则线程在排队，会出现反复阻塞、取消阻塞
       * repeatedly blocking and unblocking, invoking {@link
       * #tryAcquire} until success.  This method can be used
       * to implement method {@link Lock#lock}. -- 可用于实现lock方法
       *
       * @param arg the acquire argument.  This value is conveyed to
       *        {@link #tryAcquire} but is otherwise uninterpreted and
       *        can represent anything you like.
       */
      public final void acquire(int arg) {
          if (!tryAcquire(arg) &&
              acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
              selfInterrupt();
      }
  ```

  > acquire源码中看出可以看出，先调用tryAcquire方法如果无法成功获取到该锁，然后进行排队等待。acquireQueued和addWaiter方法就是去排队等待的，那具体怎么排队的呢？直接贴源码~

  ```java
      /**
       * Acquires in exclusive uninterruptible mode for thread already in
       * queue. Used by condition wait methods as well as acquire.
       *
       * @param node the node
       * @param arg the acquire argument
       * @return {@code true} if interrupted while waiting
       */
      final boolean acquireQueued(final Node node, int arg) {
          boolean failed = true;
          try {
              boolean interrupted = false;
              for (;;) {
                  final Node p = node.predecessor();
                  if (p == head && tryAcquire(arg)) {   // 把放入队列中的这个线程不断进行“获锁”,直到它“成功获锁”或者“不再需要锁（如被中断）
                      setHead(node);
                      p.next = null; // help GC
                      failed = false;
                      return interrupted;
                  }
                  if (shouldParkAfterFailedAcquire(p, node) &&  // 判断当前线程是不是需要被阻塞，如果被阻塞就不会无限循环消耗CPU
                      parkAndCheckInterrupt())
                      interrupted = true;
              }
          } finally {
              if (failed)
                  cancelAcquire(node);
          }
      }
  ```

  ```java
  /**
       * Creates and enqueues node for current thread and given mode. -- 把获取失败的node放进队列尾部
       *
       * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
       * @return the new node
       */
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
          enq(node);//入队-FIFO
          return node;
      }
  ```

  ```java
  /**
       * Inserts node into queue, initializing if necessary. See picture above. -- 入队 FIFO
       * @param node the node to insert
       * @return node's predecessor
       */
      private Node enq(final Node node) {
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

  > 什么时候需要阻塞线程呢？

  ``` java
  /**
       * Checks and updates status for a node that failed to acquire.
       * Returns true if thread should block. This is the main signal
       * control in all acquire loops.  Requires that pred == node.prev.
       *
       * @param pred node's predecessor holding status
       * @param node the node
       * @return {@code true} if thread should block
       */
      private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
          int ws = pred.waitStatus;
          if (ws == Node.SIGNAL)
              /*
               * This node has already set status asking a release
               * to signal it, so it can safely park.
               */
              return true;
          if (ws > 0) {
              /*
               * Predecessor was cancelled. Skip over predecessors and
               * indicate retry.
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
              compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
          }
          return false;
      }
  ```

  ```java
  /**
       * Convenience method to park and then check if interrupted -- 阻塞当前线程，并返回是否被中断
       *
       * @return {@code true} if interrupted
       */
      private final boolean parkAndCheckInterrupt() {
          LockSupport.park(this); //阻塞当前线程
          return Thread.interrupted();
      }
  ```

  

* 抢占式释放同步状态：**tryRelease**和**release**

  ```java
  /**
       * Attempts to set the state to reflect a release in exclusive
       * mode.
       *
       * <p>This method is always invoked by the thread performing release.
       *
       * <p>The default implementation throws
       * {@link UnsupportedOperationException}.
       *
       * @param arg the release argument. This value is always the one
       *        passed to a release method, or the current state value upon
       *        entry to a condition wait.  The value is otherwise
       *        uninterpreted and can represent anything you like.
       * @return {@code true} if this object is now in a fully released
       *         state, so that any waiting threads may attempt to acquire;
       *         and {@code false} otherwise.
       * @throws IllegalMonitorStateException if releasing would place this
       *         synchronizer in an illegal state. This exception must be
       *         thrown in a consistent fashion for synchronization to work
       *         correctly.
       * @throws UnsupportedOperationException if exclusive mode is not supported
       */
      protected boolean tryRelease(int arg) {
          throw new UnsupportedOperationException();
      }
  ```

  ```java
  /**
       * Releases in exclusive mode.  Implemented by unblocking one or
       * more threads if {@link #tryRelease} returns true.
       * This method can be used to implement method {@link Lock#unlock}.
       *
       * @param arg the release argument.  This value is conveyed to
       *        {@link #tryRelease} but is otherwise uninterpreted and
       *        can represent anything you like.
       * @return the value returned from {@link #tryRelease}
       */
      public final boolean release(int arg) {
          if (tryRelease(arg)) {
              Node h = head;
              if (h != null && h.waitStatus != 0)
                  unparkSuccessor(h);//唤醒线程
              return true;
          }
          return false;
      }
  ```

  ```java
  /**
       * Wakes up node's successor, if one exists.
       *
       * @param node the node
       */
      private void unparkSuccessor(Node node) {
          /*
           * If status is negative (i.e., possibly needing signal) try
           * to clear in anticipation of signalling.  It is OK if this
           * fails or if status is changed by waiting thread.
           */
          int ws = node.waitStatus;
          if (ws < 0)
              compareAndSetWaitStatus(node, ws, 0);
  
          /*
           * Thread to unpark is held in successor, which is normally
           * just the next node.  But if cancelled or apparently null,
           * traverse backwards from tail to find the actual
           * non-cancelled successor.
           */
          Node s = node.next;
          if (s == null || s.waitStatus > 0) {
              s = null;
              for (Node t = tail; t != null && t != node; t = t.prev)
                  if (t.waitStatus <= 0)
                      s = t;
          }
          if (s != null)
              LockSupport.unpark(s.thread); //唤醒满足条件的线程
      }
  ```

  先通过tryAcquire尝试获取资源，如果获取失败则把当前线程阻塞后并放入队尾。释放资源时会先调用tryRelease，如果成功则唤醒等待队列中第一个满足条件线程参与竞争资源。

​	

### 3.2 共享式获取和释放资源

* 共享式获取同步状态：**tryAcquireShared** 和 **acquireShared**

  ```java
  /**
       * Attempts to acquire in shared mode. This method should query if
       * the state of the object permits it to be acquired in the shared
       * mode, and if so to acquire it.
       *
       * <p>This method is always invoked by the thread performing
       * acquire.  If this method reports failure, the acquire method
       * may queue the thread, if it is not already queued, until it is
       * signalled by a release from some other thread.
       *
       * <p>The default implementation throws {@link
       * UnsupportedOperationException}.
       *
       * @param arg the acquire argument. This value is always the one
       *        passed to an acquire method, or is the value saved on entry
       *        to a condition wait.  The value is otherwise uninterpreted
       *        and can represent anything you like.
       * @return a negative value on failure; zero if acquisition in shared
       *         mode succeeded but no subsequent shared-mode acquire can
       *         succeed; and a positive value if acquisition in shared
       *         mode succeeded and subsequent shared-mode acquires might
       *         also succeed, in which case a subsequent waiting thread
       *         must check availability. (Support for three different
       *         return values enables this method to be used in contexts
       *         where acquires only sometimes act exclusively.)  Upon
       *         success, this object has been acquired.
       * @throws IllegalMonitorStateException if acquiring would place this
       *         synchronizer in an illegal state. This exception must be
       *         thrown in a consistent fashion for synchronization to work
       *         correctly.
       * @throws UnsupportedOperationException if shared mode is not supported
       */
      protected int tryAcquireShared(int arg) {
          throw new UnsupportedOperationException(); //由子类扩展实现，比如 ReadLock、CountDownLatch、Semaphore
      }
  ```

  ```java
      /**
       * Acquires in shared mode, ignoring interrupts.  Implemented by
       * first invoking at least once {@link #tryAcquireShared},
       * returning on success.  Otherwise the thread is queued, possibly
       * repeatedly blocking and unblocking, invoking {@link
       * #tryAcquireShared} until success.
       *
       * @param arg the acquire argument.  This value is conveyed to
       *        {@link #tryAcquireShared} but is otherwise uninterpreted
       *        and can represent anything you like.
       */
      public final void acquireShared(int arg) {
          if (tryAcquireShared(arg) < 0)  // 共享模式 用小于0判断是否还有空余资源，决定是否阻塞线程
              doAcquireShared(arg);
      }
  ```

  ```java
  /**
       * Acquires in shared uninterruptible mode.
       * @param arg the acquire argument
       */
      private void doAcquireShared(int arg) {
          final Node node = addWaiter(Node.SHARED);
          boolean failed = true;
          try {
              boolean interrupted = false;
              for (;;) {
                  final Node p = node.predecessor();
                  if (p == head) {
                      int r = tryAcquireShared(arg);
                      if (r >= 0) {
                          setHeadAndPropagate(node, r);
                          p.next = null; // help GC
                          if (interrupted)
                              selfInterrupt();
                          failed = false;
                          return;
                      }
                  }
                  if (shouldParkAfterFailedAcquire(p, node) &&
                      parkAndCheckInterrupt()) // 阻塞线程的逻辑与抢占模式一样
                      interrupted = true;
              }
          } finally {
              if (failed)
                  cancelAcquire(node);
          }
      }
  ```

  共享式获取资源的逻辑与抢占式基本一致，只是判断是否有空闲资源时，抢占式是判断tryXXX返回true，而共享式是判断资源数是否小于0。因为抢占式只允许有一个线程获得资源，而共享式允许有多个线程获得资源。

* 共享式释放同步状态：**tryReleaseShared**和**releaseShared**

  ```java
  /**
       * Attempts to set the state to reflect a release in shared mode.
       *
       * <p>This method is always invoked by the thread performing release.
       *
       * <p>The default implementation throws
       * {@link UnsupportedOperationException}.
       *
       * @param arg the release argument. This value is always the one
       *        passed to a release method, or the current state value upon
       *        entry to a condition wait.  The value is otherwise
       *        uninterpreted and can represent anything you like.
       * @return {@code true} if this release of shared mode may permit a
       *         waiting acquire (shared or exclusive) to succeed; and
       *         {@code false} otherwise
       * @throws IllegalMonitorStateException if releasing would place this
       *         synchronizer in an illegal state. This exception must be
       *         thrown in a consistent fashion for synchronization to work
       *         correctly.
       * @throws UnsupportedOperationException if shared mode is not supported
       */
      protected boolean tryReleaseShared(int arg) {
          throw new UnsupportedOperationException();
      }
  ```

  ```java
  /**
       * Releases in shared mode.  Implemented by unblocking one or more
       * threads if {@link #tryReleaseShared} returns true.
       *
       * @param arg the release argument.  This value is conveyed to
       *        {@link #tryReleaseShared} but is otherwise uninterpreted
       *        and can represent anything you like.
       * @return the value returned from {@link #tryReleaseShared}
       */
      public final boolean releaseShared(int arg) {
          if (tryReleaseShared(arg)) {
              doReleaseShared();
              return true;
          }
          return false;
      }
  ```

  ```java
  /**
       * Release action for shared mode -- signals successor and ensures -- 共享模式需要处理signals 和 propagation 状态
       * propagation. (Note: For exclusive mode, release just amounts  -- 抢占模式 直接唤醒合适的线程
       * to calling unparkSuccessor of head if it needs signal.)
       */
      private void doReleaseShared() {
          /*
           * Ensure that a release propagates, even if there are other
           * in-progress acquires/releases.  This proceeds in the usual
           * way of trying to unparkSuccessor of head if it needs
           * signal. But if it does not, status is set to PROPAGATE to
           * ensure that upon release, propagation continues.
           * Additionally, we must loop in case a new node is added
           * while we are doing this. Also, unlike other uses of
           * unparkSuccessor, we need to know if CAS to reset status
           * fails, if so rechecking.
           */
          for (;;) {
              Node h = head;
              if (h != null && h != tail) {
                  int ws = h.waitStatus;
                  if (ws == Node.SIGNAL) {
                      if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                          continue;            // loop to recheck cases
                      unparkSuccessor(h);
                  }
                  else if (ws == 0 &&
                           !compareAndSetWaitStatus(h, 0, Node.PROPAGATE)) //PROPAGATE状态特殊处理？？？
                      continue;                // loop on failed CAS
              }
              if (h == head)                   // loop if head changed
                  break;
          }
      }
  ```

  共享式释放共享资源的时候，需要考虑SIGNAL 和PROPAGATE状态，这里是和抢占式不一样。

  

## 4 AQS应用

​	    Java中几大同步工具中，ReentrantLock、WriteLock属于抢占模式，同时只允许一个线程获取同步资源，而CountDownLatch、Semaphore、ReadLock是属于共享模式，同时可以允许多个线程获取同步资源，具体的线程个数可以由参数指定。



## 5 总结

​         AQS要了解其数据结构，弄清楚了抢占模式和共享模式（其实还有可中断模式、带有超时机制）就能够顺着读源码了解大概的流程，而且源码注释都写的很清楚，如果英语不错的朋友可以直接读英文注释，相信能事半功倍。不得不佩服Doug Lea大神设计的AQS及其应用的各大同步工具实在是精彩绝伦，向大神致敬~



## 6 参考资料

* 美团技术团队       李卓            [从ReentrantLock的实现看AQS的原理及应用](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)
* 知乎        养兔子的大叔             [(JDK)ReentrantLock手撕AQS](https://zhuanlan.zhihu.com/p/54297968)
* 实体书    方腾飞&魏鹏&程晓明        《Java并发编程的艺术》 



**感谢以上平台及各位大佬的分享和付出~ 小弟拜谢！！**

