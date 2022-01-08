# Lock和Condition

> Concurrent包下的锁都是`可重入锁`，线程获得锁后，再次请求锁也可以获得该锁，锁被设计为可重入，避免死锁发生。



## 可重入锁的继承层次

![image-20220301160659361](..\..\图片\image-20220301160659361.png)

> Sync为AQS子类，它有2个子类`NofairSync`、`FairSync`



## AQS相关

> AQS是抽象队列同步器，它是`其他同步工具类的基础、框架`，它能够`高效构造出同步器`，许多同步工具类`基于它实现`
>
> 
>
> **核心要素**
>
> 1. 需要维护`锁（资源）状态`且状态的`修改需要线程安全`：volatile 修饰的`整型变量`，CAS方式进行修改
> 2. 需要记录`当前持有锁`的线程：exclusiveOwnerThread记录持有锁的线程
> 3. 需要有对一个线程进行`阻塞`、`唤醒`的功能：使用了`LockSupport类`完成阻塞、唤醒功能
> 4. 需要用队列来维护`阻塞`中的线程，队列需要是`线程安全`的`无锁`队列（需要CAS）：AQS有内部类Node封装线程，用链表结构实现队列



### AOS

```java
public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable {

    // 记录持有锁的线程
    private transient Thread exclusiveOwnerThread;

	...
}
```



> state的值可以分为3类
>
> 1. `state = 0`：没有线程持有锁，exclusiveOwnerThread为null
> 2. `state = 1`：有线程持有锁，exclusiveOwnerThread为持有锁的线程
> 3. `state > 1`：线程重入了锁

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {

    // volatile修饰的int类型，记录锁的状态，state的修改是通过CAS的方式
    private volatile int state;
    
    // 维护阻塞中的线程，需要使用Node封装线程，用节点来实现队列
    private transient volatile Node head;
    private transient volatile Node tail;
    
    // 内部类Node
	static final class Node {
       
        // 链表的头尾
        volatile Node prev;
        volatile Node next;
        
        // 使用Node来封装阻塞的线程
        volatile Thread thread;
		
        ...
    }
    
    ...
}
```



> AQS需要对线程进行阻塞、唤醒操作，它使用的是`LockSupport`，对应的方法都调用了`Unsafe`类的具体方法。
>
> `unpark`方法的参数是线程，可以实现对线程的`精确唤醒`。
>
> 在线程中调用park方法，当前线程会被阻塞

```java
public class LockSupport {

    ...

    public static void unpark(Thread thread) {
        if (thread != null)
            UNSAFE.unpark(thread);
    }
    
    public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        UNSAFE.park(false, 0L);
        setBlocker(t, null);
    }
}
```



## Lock接口

![image-20220228203328819](..\..\图片\image-20220228203328819.png)



## ReentrantLock结构

### 核心方法

> ReentrantLock本身没有逻辑，具体逻辑由内部类完成

```java
public class ReentrantLock implements Lock, java.io.Serializable {
   
	private final Sync sync;

    public void lock() {
        sync.lock();
    }
    
    public void unlock() {
        sync.release(1);
    }
    
    ...
}
```



### 构造函数

> `Sync`是一个抽象类，有2个子类`NonfairSync`、`FairSync`，默认情况下使用非公平锁，以提高性能，减少线程切换

```java
	public ReentrantLock() {
        sync = new NonfairSync();
    }


    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```



### 非公平锁与公平锁

#### 非公平锁

```java
    static final class NonfairSync extends Sync {
        
        final void lock() {
            
            // lock方法直接进行CAS操作，修改state，不考虑队列中的线程，非公平
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                
                // 失败，则调用
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            
            // 非公平锁的tryAcquire方法调用，这个方法在抽象类Sync中实现
            return nonfairTryAcquire(acquires);
        }
    }
```



> AQS的`模板方法`

```java
    public final void acquire(int arg) {
        
        // tryAcquire方法被公平锁、非公平锁实现
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```



> 抽象类Sync实现的nonfairTryAcquire方法

```java
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            
            // state为0，则没有线程持有锁
            if (c == 0) {
                
                //对state变量进行CAS操作
                if (compareAndSetState(0, acquires)) {
                    
                    //将持有锁的线程设置为当前线程
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            
            // 线程重入时，对state进行修改
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

```



#### 公平锁

```java
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            
            // tryAcquire方法尝试获得锁
            acquire(1);
        }

        // 与非公平锁类似
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                
                // 判断队列前是否有线程
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
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



### 再看acquire方法

```java
    public final void acquire(int arg) {
        
        /*
         * tryAcquire方法被公平锁、非公平锁实现
         * addWaiter方法是将当前线程加入到队列尾部，此时线程并未阻塞
         */ 
        if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            
            //当acquireQueued方法返回true时，会调用方法
            selfInterrupt();
    }
```



#### AQS的acquireQueued方法

```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                
                // 如果当前线程在队头，就尝试获得锁
                if (p == head && tryAcquire(arg)) {
                    
                    /*
                     * 获得锁后，setHead方法将头节点设置为当前节点，
                     * 并将当前节点的thread、prev设置为null
                     */
                    setHead(node);
                    
                    // 原头节点的next设置为null，help GC
                    p.next = null;
                    failed = false;
                    
                    // 返回是否被“打断”
                    return interrupted;
                }
                
                /*
                 * shouldParkAfterFailedAcquire方法返回true，
                 * 则执行parkAndCheckInterrupt方法，阻塞自己，
                 */
                if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                    
                    // parkAndCheckInterrupt方法返回true，则执行到此处
                    interrupted = true;
                
                // 被唤醒、打断后，继续执行for循环
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```



#### AQS的parkAndCheckInterrupt静态方法

> 加入同步队列后，如果不是队列第一个节点，线程将阻塞在这个方法

```java
    private final boolean parkAndCheckInterrupt() {
        
        // 调用park方法，即阻塞当前线程，park方法会响应中断
        LockSupport.park(this);
        
        // 其他线程调用t.interrupt方法，park响应中断则返回true
        return Thread.interrupted();
    }
```

> 线程被唤醒的情况有
>
> 1. 其他线程调用`LockSupport.unpark(Thread t)`精确唤醒
> 2. 阻塞的线程被“打断”



#### AQS的selfInterrupt静态方法

> 线程中断自己

```java
    static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }
```



### ReentrantLock释放锁——unlock方法

```java
    public void unlock() {
        sync.release(1);
    }
```



#### AQS的release方法

```java
    public final boolean release(int arg) {
        
        // tryRelease方法
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                
                // 唤醒队列中后一个线程
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```



#### Sync的tryRelease方法

```java
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            
            // 不需要CAS
            setState(c);
            return free;
        }
```



#### AQS的unparkSuccessor方法

```java
    private void unparkSuccessor(Node node) {

        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        // head节点的下一个节点，即下一个线程
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            
            // 唤醒下一个线程
            LockSupport.unpark(s.thread);
    }
```



## 读写锁

![image-20220301165003542](..\..\图片\image-20220301165003542.png)

> `ReadLock`、`WriteLock`关联`ReentrantReadWriteLock`中的内部类Sync



### ReadWriteLock接口

![image-20220301165536934](..\..\图片\image-20220301165536934.png)



### ReentrantReadWriteLock

```java
public class ReentrantReadWriteLock
        implements ReadWriteLock, java.io.Serializable {
    
    private final ReentrantReadWriteLock.ReadLock readerLock;
    private final ReentrantReadWriteLock.WriteLock writerLock;
    
    ...
        
    public ReentrantReadWriteLock() {
        this(false);
    }
    
    public ReentrantReadWriteLock(boolean fair) {
        
        // 创建sync对象，下方会被读、写锁共用
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }
    
 	...
}
```



#### 内部类ReadLock

```java
public static class ReadLock implements Lock, java.io.Serializable {

        ... 
        private final Sync sync;

        protected ReadLock(ReentrantReadWriteLock lock) {
            
            /*
             * 读锁、写锁都共通使用同一个sync对象
             * 即ReentrantReadWriteLock中的sync
             */ 
            sync = lock.sync;
        }
    
		public void lock() {
            sync.acquireShared(1);
        }     
    
    	public void unlock() {
            sync.releaseShared(1);
        }
        
        ...
    }
```



#### 内部类WriteLock

```java
    public static class WriteLock implements Lock, java.io.Serializable {
        
        public void lock() {
            sync.acquire(1);
        }

        public void unlock() {
            sync.release(1);
        }
}
```



#### 内部类Sync

```java
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 6317671515068378041L;

        static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }

        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
    }
```

> 将state变量分为2部分
>
> - 高16位记录读锁
> - 低16为记录写锁



#### AQS的2对模板方法

> AQS中的许多方法被它的各种子类实现
>
> - tryAcquire
> - tryRelease
> - tryAcquireShared
> - tryReleaseShared

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    
    
    // ReadLock调用acquireShared、releaseShared方法
    
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }

    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
    
    // WriteLock调用acquire、release方法
    
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
    
}
```

> 读写锁也有公平、非公平锁，所以它们有4种组合



#### 非公平锁

> 对于读锁，只有在队列`第一个不是写线程`的情况下，才会去抢锁，这样可以`避免写线程饥饿`

```java
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -8159625535654395037L;
        final boolean writerShouldBlock() {
            return false; // writers can always barge
        }
        final boolean readerShouldBlock() {
            
            // 如果队列头部时写线程，则读线程阻塞
            return apparentlyFirstQueuedIsExclusive();
        }
    }
```



#### 公平锁

> 不管是写、读锁，只要队列有线程等待，就不会去获得锁

```java
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -2274990926593161451L;
        
        //有前驱节点则阻塞，以达到公平的效果
        
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
        final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
        }
    }

```



### 内部类Sync的方法

#### 写锁相关方法

##### 写线程的加锁

```java
        protected final boolean tryAcquire(int acquires) {

            Thread current = Thread.currentThread();
            int c = getState();
            int w = exclusiveCount(c);
            if (c != 0) {
                
                /*
                 * c != 0 而 w == 0，则表明有读线程持有锁。
                 * 只有 w != 0，写线程才可能持有锁，并判断
                 * 持有锁的线程是否当前线程。
                 */
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                
                
                // 重入次数最大值判断
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");

                setState(c + acquires);
                return true;
            }
            
            // 公平与非公平的策略控制
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            
            setExclusiveOwnerThread(current);
            return true;
        }
```



##### 写线程释放锁

```java
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
    }
```



#### 读锁相关方法

##### 读线程加锁

```java
		protected final int tryAcquireShared(int unused) {
            Thread current = Thread.currentThread();
            int c = getState();
			
             // 锁被写线程持有，并且持有锁的线程不是当前线程，则直接返回
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            
            int r = sharedCount(c);
                    
            // 公平与非公平的策略控制
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                
                // r = 0，表明当前线程是第一个获得锁的线程
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                    
                /* 
                 * r != 0，当前线程又是第一个线程
                 * 则firstReaderHoldCount++，这些是相关的统计数据
                 */
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
			
			// 方法将重试获得锁
            return fullTryAcquireShared(current);
        }
```



##### 读线程释放锁

```java
        protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            
            if (firstReader == current) {
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
            
            /* 
             * 读锁是共享锁，会有多个线程同时持有读锁，所以
             * 对读锁的释放不能直接减1，而需要CAS的方式
             */
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    
                    /* 
                     * 返回nextc == 0，如果为true，则等待
                     * 中的写线程可以获得锁
                     */
                    return nextc == 0;
            }
        }
```



## Condition

> Condition的功能与wait、notify方法功能类似，wait、notify需要搭配synchronized使用，而Condition需要搭配锁使用

![image-20220301225514380](..\..\图片\image-20220301225514380.png)



> Condition应用在其他同步类中，如阻塞队列、可重入锁、读写锁中的写锁中



### ReentrantLock的newCondition方法

> 调用了sync的newCondition方法，方法在AQS中实现

```java
public Condition newCondition() {
    return sync.newCondition();
}
```



### AQS的newCondition方法

```java
final ConditionObject newCondition() {
    return new ConditionObject();
}
```



### Condition的实现类

> 每一个Condition对象都阻塞了多个线程，因此ConditionObject内部有一个`双向链表`组成的`队列`

```java
public class ConditionObject implements Condition, java.io.Serializable {

    // 使用了AQS内部类Node
    private transient Node firstWaiter;
    private transient Node lastWaiter;

    ...
        
}
```



### Condition的await方法

```java
        public final void await() throws InterruptedException {
            
            // 收到中断信号，抛出异常
            if (Thread.interrupted())
                throw new InterruptedException();
            
            // 进入等待队列
            Node node = addConditionWaiter();
            
            // 释放锁
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            
            // 判断是否在AQS同步队列中
            while (!isOnSyncQueue(node)) {
                
                // 条件阻塞
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            
            // 唤醒后，调用acquireQueued方法获得锁
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            
            // 中断唤醒，会抛出异常
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```



### AQS的addConditionWaiter方法

```java
        private Node addConditionWaiter() {
            Node t = lastWaiter;
            
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            
            // Condition需要搭配锁使用，因为持有锁，不需要CAS操作
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
```



### ConditionObject的signal方法

```java
        public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            
            // 唤醒Condition队列中的第一个线程
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }

        private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
```



### AQS的transferForSignal方法

```java
    final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        // 将线程节点加入到互斥锁队列中，再调用unpark方法，唤醒线程
        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```
