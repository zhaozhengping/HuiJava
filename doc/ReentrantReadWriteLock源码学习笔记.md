## 1 引言

​		ReadWriteLock维护了一对相互关联的锁，用于读和写，也是基于AQS实现的，ReadLock是共享式的，WriteLock是抢占式的。

```java
public interface ReadWriteLock {
    /**
     * Returns the lock used for reading.
     *
     * @return the lock used for reading
     */
    Lock readLock();

    /**
     * Returns the lock used for writing.
     *
     * @return the lock used for writing
     */
    Lock writeLock();
```

通过调用readLock方法返回的就是读锁，writeLock返回的就是写锁。读锁与读锁共享，读写互斥，写写互斥。因此读写锁适用于读多写少的场景，性能优于ReentrantLock

## 2 构造方法

```java
public class ReentrantReadWriteLock
        implements ReadWriteLock, java.io.Serializable {
    private static final long serialVersionUID = -6992448646407690164L;
    /** Inner class providing readlock */
    private final ReentrantReadWriteLock.ReadLock readerLock;
    /** Inner class providing writelock */
    private final ReentrantReadWriteLock.WriteLock writerLock;
    /** Performs all synchronization mechanics */
    final Sync sync;

    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * default (nonfair) ordering properties.
     */
    public ReentrantReadWriteLock() {
        this(false);
    }

    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * the given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }

    public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
    public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }

```

ReentrantReadWriteLock有readerLock、writeLock两个对象，由内部类实现。对象创建和ReentrantLock一样，可以分公平锁和非公平锁。



## 2 ReadLock

* lock

```java
/**
         * Acquires the read lock.
         *
         * <p>Acquires the read lock if the write lock is not held by
         * another thread and returns immediately.
         *
         * <p>If the write lock is held by another thread then
         * the current thread becomes disabled for thread scheduling
         * purposes and lies dormant until the read lock has been acquired.
         */
        public void lock() {
            sync.acquireShared(1); //以共享模式获取锁
        }
```

```java
protected final int tryAcquireShared(int unused) {
            /*
             * Walkthrough:
             * 1. If write lock held by another thread, fail. -- 如果有写锁被别的线程持有，则获取读锁失败
             * 2. Otherwise, this thread is eligible for  -- 
             *    lock wrt state, so ask if it should block
             *    because of queue policy. If not, try
             *    to grant by CASing state and updating count. -- 可以增加到65535次
             *    Note that step does not check for reentrant
             *    acquires, which is postponed to full version
             *    to avoid having to check hold count in
             *    the more typical non-reentrant case.
             * 3. If step 2 fails either because thread
             *    apparently not eligible or CAS fails or count
             *    saturated, chain to version with full retry loop.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            int r = sharedCount(c);
    	
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&            // MAX_COUNT = 65535
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }
```



* unlock

  ```java
  /**
           * Attempts to release this lock.
           *
           * <p>If the number of readers is now zero then the lock
           * is made available for write lock attempts.
           */
          public void unlock() {
              sync.releaseShared(1); // 以共享的方式释放锁
          }
  ```
  
  ```java
  protected final boolean tryReleaseShared(int unused) {
              Thread current = Thread.currentThread();
              if (firstReader == current) {
                  // assert firstReaderHoldCount > 0;
                  if (firstReaderHoldCount == 1)
                      firstReader = null;
                  else
                      firstReaderHoldCount--;
              } else {
                  HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                      rh = readHolds.get();
                  int count = rh.count;
                  if (count <= 1) {
                      readHolds.remove();
                      if (count <= 0)
                          throw unmatchedUnlockException();
                  }
                  --rh.count;
              }
              for (;;) {
                  int c = getState();
                  int nextc = c - SHARED_UNIT;
                  if (compareAndSetState(c, nextc))
                      // Releasing the read lock has no effect on readers,
                      // but it may allow waiting writers to proceed if
                      // both read and write locks are now free.
                      return nextc == 0;
              }
          }
  ```
  
  
  
* newCondition

  ```java
  /**
           * Throws {@code UnsupportedOperationException} because
           * {@code ReadLocks} do not support conditions.
           *
           * @throws UnsupportedOperationException always
           */
          public Condition newCondition() {
              throw new UnsupportedOperationException(); //读锁不支持条件队列
          }
  ```

  



## 3 WriteLock

* lock

```java
/**
         * Acquires the write lock.
         *
         * <p>Acquires the write lock if neither the read nor write lock
         * are held by another thread
         * and returns immediately, setting the write lock hold count to
         * one.
         *
         * <p>If the current thread already holds the write lock then the
         * hold count is incremented by one and the method returns
         * immediately.
         *
         * <p>If the lock is held by another thread then the current
         * thread becomes disabled for thread scheduling purposes and
         * lies dormant until the write lock has been acquired, at which
         * time the write lock hold count is set to one.
         */
        public void lock() {
            sync.acquire(1); //以抢占式的方式获取锁，如果没有读锁既没有写锁时获取成功
        }

```

```java
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

```java
protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. If read count nonzero or write count nonzero  --存在持有读锁 或存在持有写锁的线程，则获取锁失败
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            int w = exclusiveCount(c);
            if (c != 0) {
                // (Note: if c != 0 and w == 0 then shared count != 0)
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                setState(c + acquires);
                return true;
            }
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```

* unlock

  ```java
  /**
           * Attempts to release this lock.
           *
           * <p>If the current thread is the holder of this lock then
           * the hold count is decremented. If the hold count is now
           * zero then the lock is released.  If the current thread is
           * not the holder of this lock then {@link
           * IllegalMonitorStateException} is thrown.
           *
           * @throws IllegalMonitorStateException if the current thread does not
           * hold this lock
           */
          public void unlock() {
              sync.release(1);
          }
  ```

  ```java
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

  ```java
  /*
           * Note that tryRelease and tryAcquire can be called by
           * Conditions. So it is possible that their arguments contain
           * both read and write holds that are all released during a
           * condition wait and re-established in tryAcquire.
           */
  
          protected final boolean tryRelease(int releases) {
              if (!isHeldExclusively())
                  throw new IllegalMonitorStateException();
              int nextc = getState() - releases;
              boolean free = exclusiveCount(nextc) == 0;
              if (free)
                  setExclusiveOwnerThread(null);
              setState(nextc);
              return free;
          }
  ```



* newCondition

  ```java
  /**
           * Returns a {@link Condition} instance for use with this
           * {@link Lock} instance.
           * <p>The returned {@link Condition} instance supports the same
           * usages as do the {@link Object} monitor methods ({@link
           * Object#wait() wait}, {@link Object#notify notify}, and {@link
           * Object#notifyAll notifyAll}) when used with the built-in
           * monitor lock.
           *
           * <ul>
           *
           * <li>If this write lock is not held when any {@link
           * Condition} method is called then an {@link
           * IllegalMonitorStateException} is thrown.  (Read locks are
           * held independently of write locks, so are not checked or
           * affected. However it is essentially always an error to
           * invoke a condition waiting method when the current thread
           * has also acquired read locks, since other threads that
           * could unblock it will not be able to acquire the write
           * lock.)
           *
           * <li>When the condition {@linkplain Condition#await() waiting}
           * methods are called the write lock is released and, before
           * they return, the write lock is reacquired and the lock hold
           * count restored to what it was when the method was called.
           *
           * <li>If a thread is {@linkplain Thread#interrupt interrupted} while
           * waiting then the wait will terminate, an {@link
           * InterruptedException} will be thrown, and the thread's
           * interrupted status will be cleared.
           *
           * <li> Waiting threads are signalled in FIFO order.
           *
           * <li>The ordering of lock reacquisition for threads returning
           * from waiting methods is the same as for threads initially
           * acquiring the lock, which is in the default case not specified,
           * but for <em>fair</em> locks favors those threads that have been
           * waiting the longest.
           *
           * </ul>
           *
           * @return the Condition object
           */
          public Condition newCondition() {
              return sync.newCondition(); //写锁支持条件队列
          }
  ```

  

## 4 特点

类的注释详细记录了读写锁的各大特征

```java
/**
 * An implementation of {@link ReadWriteLock} supporting similar
 * semantics to {@link ReentrantLock}.
 * <p>This class has the following properties:
 *
 * <ul>
 * <li><b>Acquisition order</b>  --锁获取顺序
 *
 * <p>This class does not impose a reader or writer preference   --ReentrantLock不会将读取者优先或写入者优先强加给锁访问的排序。
 * ordering for lock access.  However, it does support an optional  -- 但是，它确实支持可选的公平 策略
 * <em>fairness</em> policy.
 *
 * <dl>
 * <dt><b><i>Non-fair mode (default)</i></b>  -- 默认模式，非公平模式
 * <dd>When constructed as non-fair (the default), the order of entry  --当非公平策略构造时，
 * to the read and write lock is unspecified, subject to reentrancy   -- 未指定进入读写锁的顺序，受到 reentrancy 约束
 * constraints.  A nonfair lock that is continuously contended may  --连续竞争的非公平锁可能无限期地推迟一个或多个 reader 或 writer 线程
 * indefinitely postpone one or more reader or writer threads, but  
 * will normally have higher throughput than a fair lock.  --但吞吐量通常要高于公平锁
 *
 * <dt><b><i>Fair mode</i></b>   -- 公平模式
 * <dd>When constructed as fair, threads contend for entry using an  -- 当使用公平策略是，线程使用一种近似同步到达的顺序争夺资源
 * approximately arrival-order policy. When the currently held lock --  当线程释放当前持有锁时，等待时间最长的write线程获取到写入锁
 * be assigned the write lock, or if there is a group of reader threads -- 如果有一组等待时间大于所有正在等待的 writer 线程 
 * is released, either the longest-waiting single writer thread will  
 * waiting longer than all waiting writer threads, that group will be   -- 将为该组分配读锁
 * assigned the read lock.
 *
 * <p>A thread that tries to acquire a fair read lock (non-reentrantly)  --当一个线程尝试获取公平策略的read-lock时
 * will block if either the write lock is held, or there is a waiting  -- 如果write-lock被持有或者有等待的write线程，则当前线程会阻塞
 * writer thread. The thread will not acquire the read lock until  -- 直到当前最久等待writer线程获得并释放了写入锁之后，该线程才会获得读取锁
 * after the oldest currently waiting writer thread has acquired and 
 * released the write lock. Of course, if a waiting writer abandons  -- 如果等待 writer 放弃其等待
 * its wait, leaving one or more reader threads as the longest waiters -- 队列中一个或更多reader线程 为带有写入锁自由的时间最长的 waiter，
 * in the queue with the write lock free, then those readers will be 
 * assigned the read lock.                                                -- 则将为那些 reader 分配读取锁
 *
 * <p>A thread that tries to acquire a fair write lock (non-reentrantly)  -- 当一个线程尝试获取公平策略的write-lock时
 * will block unless both the read lock and write lock are free (which  -- 除非read-lock 和write-lock都free 
 * implies there are no waiting threads).  (Note that the non-blocking ----没有线程在等待 ，否则被阻塞
 * {@link ReadLock#tryLock()} and {@link WriteLock#tryLock()} methods    -- 但是trylock方法 不管队列中是否有线程等待，
 * do not honor this fair setting and will immediately acquire the lock  --会立即获取write-lock
 * if it is possible, regardless of waiting threads.)
 * <p>
 * </dl>
 *
 * <li><b>Reentrancy</b>  -- 可重入
 *
 * <p>This lock allows both readers and writers to reacquire read or -- 读锁可以重入读锁，写锁可以重入写锁
 * write locks in the style of a {@link ReentrantLock}. Non-reentrant  --读锁不能重入写锁，直到写锁被释放
 * readers are not allowed until all write locks held by the writing
 * thread have been released.
 *
 * <p>Additionally, a writer can acquire the read lock, but not
 * vice-versa.  Among other applications, reentrancy can be useful
 * when write locks are held during calls or callbacks to methods that
 * perform reads under read locks.  If a reader tries to acquire the
 * write lock it will never succeed.
 *
 * <li><b>Lock downgrading</b>   --锁降级
 * <p>Reentrancy also allows downgrading from the write lock to a read lock, --重入还允许从写入锁降级为读取锁
 * by acquiring the write lock, then the read lock and then releasing the  -- 先获取写入锁，然后获取读取锁，最后释放写入锁
 * write lock. However, upgrading from a read lock to the write lock is  -- 从读取锁升级到写入锁是不可能的
 * <b>not</b> possible.
 *
 * <li><b>Interruption of lock acquisition</b> -- 锁获取的中断
 * <p>The read lock and write lock both support interruption during lock  --读取锁和写入锁都支持锁获取期间的中断。
 * acquisition.
 *
 * <li><b>{@link Condition} support</b>  --条件队列支持
 * <p>The write lock provides a {@link Condition} implementation that
 * behaves in the same way, with respect to the write lock, as the
 * {@link Condition} implementation provided by
 * {@link ReentrantLock#newCondition} does for {@link ReentrantLock}.
 * This {@link Condition} can, of course, only be used with the write lock.
 *
 * <p>The read lock does not support a {@link Condition} and  -- 读锁不支持Condition
 * {@code readLock().newCondition()} throws
 * {@code UnsupportedOperationException}.
 *
 * <li><b>Instrumentation</b>
 * <p>This class supports methods to determine whether locks
 * are held or contended. These methods are designed for monitoring
 * system state, not for synchronization control.
 * </ul>
 *
 * <p>Serialization of this class behaves in the same way as built-in
 * locks: a deserialized lock is in the unlocked state, regardless of
 * its state when serialized.
 *
 * <p><b>Sample usages</b>. Here is a code sketch showing how to perform -- 样例很多时候就是最佳实践
 * lock downgrading after updating a cache (exception handling is
 * particularly tricky when handling multiple locks in a non-nested
 * fashion):
 *
 * <pre> {@code
 * class CachedData { 
 *   Object data;
 *   volatile boolean cacheValid;
 *   final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
 *
 *   void processCachedData() {
 *     rwl.readLock().lock();
 *     if (!cacheValid) {
 *       // Must release read lock before acquiring write lock
 *       rwl.readLock().unlock();
 *       rwl.writeLock().lock();
 *       try {
 *         // Recheck state because another thread might have
 *         // acquired write lock and changed state before we did.
 *         if (!cacheValid) {
 *           data = ...
 *           cacheValid = true;
 *         }
 *         // Downgrade by acquiring read lock before releasing write lock -- 获取写锁后 再获取读锁 
 *         rwl.readLock().lock();
 *       } finally {
 *         rwl.writeLock().unlock(); // Unlock write, still hold read   -- 释放写锁，但是还持有读锁
 *       }
 *     }
 *
 *     try {
 *       use(data);
 *     } finally {
 *       rwl.readLock().unlock();  -- 释放读锁
 *     }
 *   }
 * }}</pre>
 *
 * ReentrantReadWriteLocks can be used to improve concurrency in some
 * uses of some kinds of Collections. This is typically worthwhile
 * only when the collections are expected to be large, accessed by
 * more reader threads than writer threads, and entail operations with
 * overhead that outweighs synchronization overhead. For example, here
 * is a class using a TreeMap that is expected to be large and
 * concurrently accessed.
 *
 *  <pre> {@code
 * class RWDictionary {
 *   private final Map<String, Data> m = new TreeMap<String, Data>();
 *   private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
 *   private final Lock r = rwl.readLock();
 *   private final Lock w = rwl.writeLock();
 *
 *   public Data get(String key) {
 *     r.lock();
 *     try { return m.get(key); }
 *     finally { r.unlock(); }
 *   }
 *   public String[] allKeys() {
 *     r.lock();
 *     try { return m.keySet().toArray(); }
 *     finally { r.unlock(); }
 *   }
 *   public Data put(String key, Data value) {
 *     w.lock();
 *     try { return m.put(key, value); }
 *     finally { w.unlock(); }
 *   }
 *   public void clear() {
 *     w.lock();
 *     try { m.clear(); }
 *     finally { w.unlock(); }
 *   }
 * }}</pre>
 *
 * <h3>Implementation Notes</h3>
 *
 * <p>This lock supports a maximum of 65535 recursive write locks  --最多支持 65535 个递归写入锁和 65535 个读取锁
 * and 65535 read locks. Attempts to exceed these limits result in
 * {@link Error} throws from locking methods.
 *
 * @since 1.5
 * @author Doug Lea
 */
public class ReentrantReadWriteLock
        implements ReadWriteLock, java.io.Serializable {...}
```





## 5 总结

​    	读写锁在ReentrantLock的基础上更细分了读锁和写锁的场景，允许多个线程持有读锁的情况，适合于读多写少的场景。其实我们也能够看出，没有修改就没有并发问题，反过来想不修改共享资源也就可以避免并发问题。      

​	



## 6 参考资料

* 知乎        欣然             [Java并发编程——ReentrantReadWriteLock](https://zhuanlan.zhihu.com/p/38012123)

  

**感谢以上平台及各位大佬的分享和付出~ 小弟拜谢！！**