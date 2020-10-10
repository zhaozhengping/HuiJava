## 1 引言

​		我们知道用synchronized实现线程的同步，锁的获取和释放都是隐式的，通过编译后加上机器指令（monitorenter、monitorexit）实现。而ReentrantLock是基于AQS实现的**抢占式**、**可重入**锁，我们使用ReentrantLock的姿势经常是这样的。

```java
private ReentrantLock lock = new ReentrantLock();
    public void run() {
        lock.lock(); //
        try {
            //do bussiness

        } finally {
            lock.unlock();
        }
    }
```

与synchronized对比，使用ReentrantLock需要手动设置加锁和解锁，比较灵活，能够控制加锁的粒度，还能响应中断，实现公平锁。

## 2 构造方法

```java
/**
     * Creates an instance of {@code ReentrantLock}.
     * This is equivalent to using {@code ReentrantLock(false)}.
     */
    public ReentrantLock() {
        sync = new NonfairSync(); //默认是非公平锁
    }

    /**
     * Creates an instance of {@code ReentrantLock} with the
     * given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantLock(boolean fair) { //通过参数设置公平/非公平锁
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

从构造方法可看ReentrantLock对象创建需要依赖FairSync、NonfairSync来创建出公平/非公平锁，而FairSync继承于Sync类，Sync又继承于AQS，类图如下：

![image-20201008234104139](C:\Users\zhaozp\AppData\Roaming\Typora\typora-user-images\image-20201008234104139.png)



## 2 获取锁

* 公平锁**FairSync**

```java
/**
     * Sync object for fair locks
     */
    static final class FairSync extends Sync { //内部类FairSync
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1); //直接调用AQS的acquire方法控制线程的同步，具体的逻辑可以参考另一篇AQS源码分析的文章
        }

        /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
        protected final boolean tryAcquire(int acquires) {  //覆盖父类AQS的tryAcquire方法
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) { // c==0表示当前没有线程持有该锁
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) { //实现公平性 ---- hasQueuedPredecessors 如果当前等待线程久于当前线程
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) { // 实现可重入---判断当前持有锁的线程是否当前线程，如果是则获取锁成功
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```

```java
/**
     * Queries whether any threads have been waiting to acquire longer
     * than the current thread.
     *
     * <p>An invocation of this method is equivalent to (but may be
     * more efficient than):
     *  <pre> {@code
     * getFirstQueuedThread() != Thread.currentThread() &&
     * hasQueuedThreads()}</pre>
     *
     * <p>Note that because cancellations due to interrupts and
     * timeouts may occur at any time, a {@code true} return does not
     * guarantee that some other thread will acquire before the current
     * thread.  Likewise, it is possible for another thread to win a
     * race to enqueue after this method has returned {@code false},
     * due to the queue being empty.
     *
     * <p>This method is designed to be used by a fair synchronizer to
     * avoid <a href="AbstractQueuedSynchronizer#barging">barging</a>.
     * Such a synchronizer's {@link #tryAcquire} method should return
     * {@code false}, and its {@link #tryAcquireShared} method should
     * return a negative value, if this method returns {@code true}
     * (unless this is a reentrant acquire).  For example, the {@code
     * tryAcquire} method for a fair, reentrant, exclusive mode
     * synchronizer might look like this:
     *
     *  <pre> {@code
     * protected boolean tryAcquire(int arg) {
     *   if (isHeldExclusively()) {
     *     // A reentrant acquire; increment hold count
     *     return true;
     *   } else if (hasQueuedPredecessors()) {
     *     return false;
     *   } else {
     *     // try to acquire normally
     *   }
     * }}</pre>
     *
     * @return {@code true} if there is a queued thread preceding the
     *         current thread, and {@code false} if the current thread
     *         is at the head of the queue or the queue is empty
     * @since 1.7
     */
    public final boolean hasQueuedPredecessors() { //保证公平性，判断等待队列是否有别的线程
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```



* 非公平锁 **NonFairSync**

  ```java
  /**
       * Sync object for non-fair locks
       */
      static final class NonfairSync extends Sync {
          private static final long serialVersionUID = 7316153563782823691L;
  
          /**
           * Performs lock.  Try immediate barge, backing up to normal
           * acquire on failure. --直接立即获取锁
           */
          final void lock() {
              if (compareAndSetState(0, 1))
                  setExclusiveOwnerThread(Thread.currentThread()); // 和公平锁不一样，不判断等待队列，直接抢占式申请锁
              else
                  acquire(1);
          }
  
          protected final boolean tryAcquire(int acquires) {
              return nonfairTryAcquire(acquires); //覆盖AQS的tryAcquire方法，直接调用Sync的nonfairTryAcquire方法
          }
      }
  ```

  **FairSync**获取锁会判断当前是否有等待线程，而**NonFairSync**不判断等待队列直接申请锁，这是二者的不同，获取锁的机制都调用了AQS的acquire方法。



## 3 释放锁

```java
/**
     * Attempts to release this lock.
     *
     * <p>If the current thread is the holder of this lock then the hold
     * count is decremented.  If the hold count is now zero then the lock
     * is released.  If the current thread is not the holder of this
     * lock then {@link IllegalMonitorStateException} is thrown.
     *
     * @throws IllegalMonitorStateException if the current thread does not
     *         hold this lock
     */
    public void unlock() {
        sync.release(1); //调用AQS的release方法，公平锁和非公平锁都是一样的
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
    public final boolean release(int arg) { //AQS的release方法
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

```java
protected final boolean tryRelease(int releases) { //Sync的tryRelease方法
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread()) //判断是否当前线程持有锁
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

释放锁方面，公平锁和非公平锁的处理机制都是一样的，都依赖AQS的release方法。



## 4 Condition

### 4.1 基本结构

AQS中另一个队列ConditionObject，实现了Condition接口，需要通过Lock.newCondition()创建。

```java
final ConditionObject newCondition() {
            return new ConditionObject();
        }
```



Condition接口有以下方法：

![image-20201010232956008](C:\Users\zhaozp\AppData\Roaming\Typora\typora-user-images\image-20201010232956008.png)

Condition中基本就是await() 和signal()方法，调用await和signal方法需要在lock.lock()和lock.unlock()之间才可以用，使用姿势如下：

```java
private final ReentrantLock lock = new ReentrantLock();
    private final Condition condition = lock.newCondition();
    public void run() {
        lock.lock();
        try {
            //do business
            condition.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
```



### 4.2 实现原理

* await

```java
/**
         * Implements interruptible condition wait.
         * <ol>
         * <li> If current thread is interrupted, throw InterruptedException.
         * <li> Save lock state returned by {@link #getState}.
         * <li> Invoke {@link #release} with saved state as argument,
         *      throwing IllegalMonitorStateException if it fails.
         * <li> Block until signalled or interrupted. -- 阻塞直到被唤醒
         * <li> Reacquire by invoking specialized version of
         *      {@link #acquire} with saved state as argument.
         * <li> If interrupted while blocked in step 4, throw InterruptedException.
         * </ol>
         */
        public final void await() throws InterruptedException { //AQS的await方法
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter(); //先把当前线程放到condition条件队列里
            int savedState = fullyRelease(node); //调用await前，当前线程是占有锁的,先释放掉
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) { //释放完毕后，看当前节点是否在AQS队列中，如果不在就说明它还没有竞争锁的资格，所以继续将自己沉睡
                LockSupport.park(this); //阻塞当前线程
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            
            //被唤醒后，重新开始正式竞争锁，同样，如果竞争不到还是会将自己沉睡，等待唤醒重新开始竞争
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE) 
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

* signalAll

  ```java
  /**
           * Moves all threads from the wait queue for this condition to
           * the wait queue for the owning lock.
           *
           * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
           *         returns {@code false}
           */
          public final void signalAll() {
              if (!isHeldExclusively())
                  throw new IllegalMonitorStateException();
              Node first = firstWaiter; //firstWaiter为condition自己维护的一个链表的头结点
              if (first != null)
                  doSignalAll(first);
          }
  ```

  ```java
  /**
           * Removes and transfers all nodes.
           * @param first (non-null) the first node on condition queue
           */
          private void doSignalAll(Node first) {
              lastWaiter = firstWaiter = null;
              do {
                  Node next = first.nextWaiter;
                  first.nextWaiter = null;
                  transferForSignal(first); //取出全部线程唤醒
                  first = next;
              } while (first != null);
          }
  ```

  其实Condition内部维护了等待队列的头结点和尾节点，该队列的作用是存放等待signal信号的线程，该线程被封装为Node节点后存放于此。



## 5 总结

​       AQS维护了2个队列，一个是同步队列，存放的是等待获取锁的线程。另一个就是Condition条件队列，存放的是等待条件满足的signal信号的线程。调用await和signal方法需要在lock.lock()和lock.unlock()之间才可以用。



## 6 参考资料

* 知乎        欣然             [java condition使用及分析](https://zhuanlan.zhihu.com/p/109211584)

  

**感谢以上平台及各位大佬的分享和付出~ 小弟拜谢！！**