## 1 引言

​		CountDownLatch和Semaphore同样是基于AQS实现的同步工具，可以控制线程等待，二者都是共享式获取共享资源的方式。

## 2 CountDownLatch

### 2.1 基本结构

CountDownLatch主要包含了内部类Sync，和ReentrantLock实现类似，主要方法有await 和countDown 方法。

```java
/**
     * Synchronization control For CountDownLatch.
     * Uses AQS state to represent count.  -- 基于AQS实现同步控制
     */
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero  --计数减1，等0时唤醒
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

    private final Sync sync;

    /**
     * Constructs a {@code CountDownLatch} initialized with the given count. --构造函数需要传个计数count，当count减为0时线程被唤醒
     *
     * @param count the number of times {@link #countDown} must be invoked
     *        before threads can pass through {@link #await}
     * @throws IllegalArgumentException if {@code count} is negative
     */
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

```



### 2.2 获取资源-await

```java
/**
     * Causes the current thread to wait until the latch has counted down to  -- 阻塞当前线程直到count被减为0
     * zero, unless the thread is {@linkplain Thread#interrupt interrupted}.  -- 除非被中断
     *
     * <p>If the current count is zero then this method returns immediately. -- 如果当前为0则直接返回
     *
     * <p>If the current count is greater than zero then the current
     * thread becomes disabled for thread scheduling purposes and lies
     * dormant until one of two things happen:
     * <ul>
     * <li>The count reaches zero due to invocations of the
     * {@link #countDown} method; or
     * <li>Some other thread {@linkplain Thread#interrupt interrupts}
     * the current thread.
     * </ul>
     *
     * <p>If the current thread:
     * <ul>
     * <li>has its interrupted status set on entry to this method; or
     * <li>is {@linkplain Thread#interrupt interrupted} while waiting,
     * </ul>
     * then {@link InterruptedException} is thrown and the current thread's
     * interrupted status is cleared.
     *
     * @throws InterruptedException if the current thread is interrupted
     *         while waiting
     */
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1); // AQS可中断共享获取资源
    }
```

```java
protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1; // state == 0则获取成功
        }
```

### 2.3 释放资源- countDown

```java
/**
     * Decrements the count of the latch, releasing all waiting threads if
     * the count reaches zero.
     *
     * <p>If the current count is greater than zero then it is decremented.
     * If the new count is zero then all waiting threads are re-enabled for
     * thread scheduling purposes.
     *
     * <p>If the current count equals zero then nothing happens.
     */
    public void countDown() {
        sync.releaseShared(1);
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
    public final boolean releaseShared(int arg) {  // AQS的共享模式释放资源
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

```java
protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
```



### 2.4 特点及用法

又到了学习大神英文注释的时候~

```java
/**
 * A synchronization aid that allows one or more threads to wait until -- 一种同步机制可以允许
 * a set of operations being performed in other threads completes.  --某个或多个线程线程等待别的多个线程完成xx操作
 *
 * <p>A {@code CountDownLatch} is initialized with a given <em>count</em>. -- 通过给定count初始化
 * The {@link #await await} methods block until the current count reaches -- 调用await会被阻塞，直到别的线程通过countDown方法使count
 * zero due to invocations of the {@link #countDown} method, after which  -- 减到0
 * all waiting threads are released and any subsequent invocations of
 * {@link #await await} return immediately.  This is a one-shot phenomenon  -- 这个是一次性的，count不能被重置，如果需要重置可以用
 * -- the count cannot be reset.  If you need a version that resets the   -- CyclicBarrier
 * count, consider using a {@link CyclicBarrier}.
 *
 * <p>A {@code CountDownLatch} is a versatile synchronization tool -- CountDownLatch可作为同步工具可以做以下几件事
 * and can be used for a number of purposes.  A
 * {@code CountDownLatch} initialized with a count of one serves as a   -- 1.简单的作为控制大门的一把锁(1/0)，所有线程都在等待大门打开，
 * simple on/off latch, or gate: all threads invoking {@link #await await} --直到某个线程调用countDown方法(1-->0)
 * wait at the gate until it is opened by a thread invoking {@link
 * #countDown}.  A {@code CountDownLatch} initialized to <em>N</em> -- 2.初始化为N，用于某一个线程等待另外的N个线程 完成某些action，
 * can be used to make one thread wait until <em>N</em> threads have 
 * completed some action, or some action has been completed N times. -- 或者某个action 被执行N次
 *
 * <p>A useful property of a {@code CountDownLatch} is that it  -- 一个比较有用的是，不需要调用countDown的线程会阻塞等待，直到count减到0
 * doesn't require that threads calling {@code countDown} wait for
 * the count to reach zero before proceeding, it simply prevents any
 * thread from proceeding past an {@link #await await} until all
 * threads could pass.
 *
 * <p><b>Sample usage:</b> Here is a pair of classes in which a group   -- 使用样例展示，值得好好学习
 * of worker threads use two countdown latches:
 * <ul>
 * <li>The first is a start signal that prevents any worker from proceeding
 * until the driver is ready for them to proceed;
 * <li>The second is a completion signal that allows the driver to wait
 * until all workers have completed.
 * </ul>
 *
 *  <pre> {@code
 * class Driver { // ...
 *   void main() throws InterruptedException {
 *     CountDownLatch startSignal = new CountDownLatch(1);
 *     CountDownLatch doneSignal = new CountDownLatch(N);
 *
 *     for (int i = 0; i < N; ++i) // create and start threads
 *       new Thread(new Worker(startSignal, doneSignal)).start();
 *
 *     doSomethingElse();            // don't let run yet
 *     startSignal.countDown();      // let all threads proceed
 *     doSomethingElse();
 *     doneSignal.await();           // wait for all to finish
 *   }
 * }
 *
 * class Worker implements Runnable {
 *   private final CountDownLatch startSignal;
 *   private final CountDownLatch doneSignal;
 *   Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
 *     this.startSignal = startSignal;
 *     this.doneSignal = doneSignal;
 *   }
 *   public void run() {
 *     try {
 *       startSignal.await();
 *       doWork();
 *       doneSignal.countDown();
 *     } catch (InterruptedException ex) {} // return;
 *   }
 *
 *   void doWork() { ... }
 * }}</pre>
 *
 * <p>Another typical usage would be to divide a problem into N parts,
 * describe each part with a Runnable that executes that portion and
 * counts down on the latch, and queue all the Runnables to an
 * Executor.  When all sub-parts are complete, the coordinating thread
 * will be able to pass through await. (When threads must repeatedly
 * count down in this way, instead use a {@link CyclicBarrier}.)
 *
 *  <pre> {@code
 * class Driver2 { // ...
 *   void main() throws InterruptedException {
 *     CountDownLatch doneSignal = new CountDownLatch(N);
 *     Executor e = ...
 *
 *     for (int i = 0; i < N; ++i) // create and start threads
 *       e.execute(new WorkerRunnable(doneSignal, i));
 *
 *     doneSignal.await();           // wait for all to finish
 *   }
 * }
 *
 * class WorkerRunnable implements Runnable {
 *   private final CountDownLatch doneSignal;
 *   private final int i;
 *   WorkerRunnable(CountDownLatch doneSignal, int i) {
 *     this.doneSignal = doneSignal;
 *     this.i = i;
 *   }
 *   public void run() {
 *     try {
 *       doWork(i);
 *       doneSignal.countDown();
 *     } catch (InterruptedException ex) {} // return;
 *   }
 *
 *   void doWork() { ... }
 * }}</pre>
 *
 * <p>Memory consistency effects: Until the count reaches
 * zero, actions in a thread prior to calling
 * {@code countDown()}
 * <a href="package-summary.html#MemoryVisibility"><i>happen-before</i></a>
 * actions following a successful return from a corresponding
 * {@code await()} in another thread.
 *
 * @since 1.5
 * @author Doug Lea
 */
public class CountDownLatch {...}
```



## 3 Semaphore

### 3.1 基本结构

```java
/**
     * Creates a {@code Semaphore} with the given number of
     * permits and nonfair fairness setting.
     *
     * @param permits the initial number of permits available.
     *        This value may be negative, in which case releases
     *        must occur before any acquires will be granted.
     */
    public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }

    /**
     * Creates a {@code Semaphore} with the given number of
     * permits and the given fairness setting.
     *
     * @param permits the initial number of permits available.
     *        This value may be negative, in which case releases
     *        must occur before any acquires will be granted.
     * @param fair {@code true} if this semaphore will guarantee
     *        first-in first-out granting of permits under contention,
     *        else {@code false}
     */
    public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }
```

Semaphore和ReentrantLock类似，有公平方式和非公平方式两种实现。



### 3.2 获取资源-acquire

```java
/**
     * Acquires a permit from this semaphore, blocking until one is  -- 从信号量获取资源，会阻塞直到 有1个信号量可用
     * available, or the thread is {@linkplain Thread#interrupt interrupted}. -- 或者 别中断
     *
     * <p>Acquires a permit, if one is available and returns immediately, -- 如果有1个信号可用，则立即返回，并permits减1
     * reducing the number of available permits by one.
     *
     * <p>If no permit is available then the current thread becomes  -- 如果没有信号可用，则线程休眠，直到出现以下2件事发生
     * disabled for thread scheduling purposes and lies dormant until
     * one of two things happens:
     * <ul>
     * <li>Some other thread invokes the {@link #release} method for this  -- 1.其他线程释放了资源，并且当前线程接下来就被唤醒获取资源
     * semaphore and the current thread is next to be assigned a permit; or  -- 2.被其他线程中断
     * <li>Some other thread {@linkplain Thread#interrupt interrupts}
     * the current thread.
     * </ul>
     *
     * <p>If the current thread:
     * <ul>
     * <li>has its interrupted status set on entry to this method; or
     * <li>is {@linkplain Thread#interrupt interrupted} while waiting
     * for a permit,
     * </ul>
     * then {@link InterruptedException} is thrown and the current thread's
     * interrupted status is cleared.
     *
     * @throws InterruptedException if the current thread is interrupted
     */
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1); // 可中断的共享模式获取资源
    }
```

```java
/**
     * Acquires in shared mode, aborting if interrupted.  Implemented
     * by first checking interrupt status, then invoking at least once
     * {@link #tryAcquireShared}, returning on success.  Otherwise the
     * thread is queued, possibly repeatedly blocking and unblocking,
     * invoking {@link #tryAcquireShared} until success or the thread
     * is interrupted.
     * @param arg the acquire argument.
     * This value is conveyed to {@link #tryAcquireShared} but is
     * otherwise uninterpreted and can represent anything
     * you like.
     * @throws InterruptedException if the current thread is interrupted
     */
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```

```java
static final class FairSync extends Sync {
        private static final long serialVersionUID = 2014338818796000944L;

        FairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) { //公平模式
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }
```

```java
protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires); //非公平模式获取资源
        }


final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```



### 3.3 释放资源-release

```java
/**
     * Releases a permit, returning it to the semaphore.
     *
     * <p>Releases a permit, increasing the number of available permits by
     * one.  If any threads are trying to acquire a permit, then one is
     * selected and given the permit that was just released.  That thread
     * is (re)enabled for thread scheduling purposes.
     *
     * <p>There is no requirement that a thread that releases a permit must
     * have acquired that permit by calling {@link #acquire}.
     * Correct usage of a semaphore is established by programming convention
     * in the application.
     */
    public void release() {
        sync.releaseShared(1); //共享的方式释放资源
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
protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }
```

### 3.4 特点及用法

又到了学习大神英文注释的时候~

```java
/**
 * A counting semaphore.  Conceptually, a semaphore maintains a set of --一个计数的信号，概念上讲，一个信号维持了一组许可
 * permits.  Each {@link #acquire} blocks if necessary until a permit is  -- 线程必须获取到这个许可才能运行
 * available, and then takes it.  Each {@link #release} adds a permit,  -- 线程释放一个许可，就会唤醒一个被阻塞的线程
 * potentially releasing a blocking acquirer.
 * However, no actual permit objects are used; the {@code Semaphore} just -- 并没有实际的许可对象，只是维护一个数量，保证线程和谐运行
 * keeps a count of the number available and acts accordingly.
 *
 * <p>Semaphores are often used to restrict the number of threads than can  --Semaphore经常用于限制线程获取物理上或逻辑上的资源。 
 * access some (physical or logical) resource. For example, here is  -- 比如这里有个例子，展示了 用于控制进入对象池获取对象
 * a class that uses a semaphore to control access to a pool of items:  
 *  <pre> {@code
 * class Pool {   -- 可能就是最佳实践用法
 *   private static final int MAX_AVAILABLE = 100;
 *   private final Semaphore available = new Semaphore(MAX_AVAILABLE, true);
 *
 *   public Object getItem() throws InterruptedException {
 *     available.acquire();
 *     return getNextAvailableItem();
 *   }
 *
 *   public void putItem(Object x) {
 *     if (markAsUnused(x))
 *       available.release();
 *   }
 *
 *   // Not a particularly efficient data structure; just for demo
 *
 *   protected Object[] items = ... whatever kinds of items being managed
 *   protected boolean[] used = new boolean[MAX_AVAILABLE];
 *
 *   protected synchronized Object getNextAvailableItem() {
 *     for (int i = 0; i < MAX_AVAILABLE; ++i) {
 *       if (!used[i]) {
 *          used[i] = true;
 *          return items[i];
 *       }
 *     }
 *     return null; // not reached
 *   }
 *
 *   protected synchronized boolean markAsUnused(Object item) {
 *     for (int i = 0; i < MAX_AVAILABLE; ++i) {
 *       if (item == items[i]) {
 *          if (used[i]) {
 *            used[i] = false;
 *            return true;
 *          } else
 *            return false;
 *       }
 *     }
 *     return false;
 *   }
 * }}</pre>
 *
 * <p>Before obtaining an item each thread must acquire a permit from
 * the semaphore, guaranteeing that an item is available for use. When
 * the thread has finished with the item it is returned back to the
 * pool and a permit is returned to the semaphore, allowing another
 * thread to acquire that item.  Note that no synchronization lock is
 * held when {@link #acquire} is called as that would prevent an item
 * from being returned to the pool.  The semaphore encapsulates the
 * synchronization needed to restrict access to the pool, separately
 * from any synchronization needed to maintain the consistency of the
 * pool itself.
 *
 * <p>A semaphore initialized to one, and which is used such that it
 * only has at most one permit available, can serve as a mutual
 * exclusion lock.  This is more commonly known as a <em>binary
 * semaphore</em>, because it only has two states: one permit
 * available, or zero permits available.  When used in this way, the
 * binary semaphore has the property (unlike many {@link java.util.concurrent.locks.Lock}
 * implementations), that the &quot;lock&quot; can be released by a
 * thread other than the owner (as semaphores have no notion of
 * ownership).  This can be useful in some specialized contexts, such
 * as deadlock recovery.
 *
 * <p> The constructor for this class optionally accepts a
 * <em>fairness</em> parameter. When set false, this class makes no
 * guarantees about the order in which threads acquire permits. In
 * particular, <em>barging</em> is permitted, that is, a thread
 * invoking {@link #acquire} can be allocated a permit ahead of a
 * thread that has been waiting - logically the new thread places itself at
 * the head of the queue of waiting threads. When fairness is set true, the
 * semaphore guarantees that threads invoking any of the {@link
 * #acquire() acquire} methods are selected to obtain permits in the order in
 * which their invocation of those methods was processed
 * (first-in-first-out; FIFO). Note that FIFO ordering necessarily
 * applies to specific internal points of execution within these
 * methods.  So, it is possible for one thread to invoke
 * {@code acquire} before another, but reach the ordering point after
 * the other, and similarly upon return from the method.
 * Also note that the untimed {@link #tryAcquire() tryAcquire} methods do not
 * honor the fairness setting, but will take any permits that are
 * available.
 *
 * <p>Generally, semaphores used to control resource access should be
 * initialized as fair, to ensure that no thread is starved out from
 * accessing a resource. When using semaphores for other kinds of
 * synchronization control, the throughput advantages of non-fair
 * ordering often outweigh fairness considerations.
 *
 * <p>This class also provides convenience methods to {@link
 * #acquire(int) acquire} and {@link #release(int) release} multiple
 * permits at a time.  Beware of the increased risk of indefinite
 * postponement when these methods are used without fairness set true.
 *
 * <p>Memory consistency effects: Actions in a thread prior to calling
 * a "release" method such as {@code release()}
 * <a href="package-summary.html#MemoryVisibility"><i>happen-before</i></a>
 * actions following a successful "acquire" method such as {@code acquire()}
 * in another thread.
 *
 * @since 1.5
 * @author Doug Lea
 */
public class Semaphore implements java.io.Serializable {...}
```



## 4 总结

​    	CountDownLatch和Semaphore都是基于AQS的共享锁，Semaphore有公平和非公平两种实现方式。 CountDownLatch可以用于某些线程等待别的线程的场景，而Semaphore用于控制一组线程在资源足够的情况下才能允许的场景，有点类似流控拿到信号令牌才能通过。
