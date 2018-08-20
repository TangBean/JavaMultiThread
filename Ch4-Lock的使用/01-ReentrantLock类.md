# ReentrantLock类

ReentrantLock 类：可重入锁，JDK 1.5 中新增加的。我们知道可以使用 synchronized 关键字实现线程间的互斥，使用 ReentrantLock 类也可以达到相同的效果，并且具有嗅探锁定，多路分支通知等扩展功能，使用起来也更为灵活。

## ReentrantLock类的使用简介

### 一个简单的使用示例

```java
private Lock lock = new ReentrantLock();
...
public void method() {
    lock.lock();
    for (int i = 0; i < 5; i++) { // 相当于把for循环这部分代码放入synchronized代码块中
        System.out.println(i);
    }
    lock.unlock();
}
```

### 标准使用示例

比较标准的使用方法是：

```java
private Lock lock = new ReentrantLock();
...
public void method() {
    try {
		lock.lock(); // try块中加锁
		...
    } catch (Exception e) {
		...
    } finally {
		lock.unlock(); // finally块中放锁
    }
}
```

例如：文件 MyService.java 代码如下：线程调用这两个方法都是同步的。

```java
public class MyService {
    private Lock lock = new ReentrantLock();

    public void methodA() {
        try {
            lock.lock();
            System.out.println("Method A begin, ThreadName=" + Thread.currentThread().getName()
                    + ", time=" + System.currentTimeMillis());
            Thread.sleep(5000);
            System.out.println("Method A end, ThreadName=" + Thread.currentThread().getName()
                    + ", time=" + System.currentTimeMillis());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void methodB() {
        try {
            lock.lock();
            System.out.println("Method B begin, ThreadName=" + Thread.currentThread().getName()
                    + ", time=" + System.currentTimeMillis());
            Thread.sleep(5000);
            System.out.println("Method B end, ThreadName=" + Thread.currentThread().getName()
                    + ", time=" + System.currentTimeMillis());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

## Condition类实现等待/通知

### Condition类的方法介绍

| Condition中的方法                 | 对应Object中的方法 | 作用                                          |
| --------------------------------- | ------------------ | --------------------------------------------- |
| `await()`                         | `wait()`           | 等待                                          |
| `await(long time, TimeUnit unit)` | `wait(long time)`  | 等待一定时间，TimeUnit 是用来指定 time 的单位 |
| `signal()`                        | `notify()`         | 通知一个等待的线程执行                        |
| `signalAll()`                     | `notifuAll()`      | 通知所有等待的线程执行                        |

**简单使用示例：**

```java
public class MyService {
    private Lock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();

    public void await() {
        try {
            lock.lock(); // 1.加锁
            condition.await(); // 2.等待
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock(); // 3.放锁
        }
    }

    public void signal() {
        try {
            lock.lock(); // 1.加锁
            condition.signal(); // 2.通知
        } finally {
            lock.unlock(); // 3.放锁
        }
    }
}
```

**注意：**使用 await() 和 signal() 前必须先 lock.lock()，这相当于只能在 synchronized 方法或同步代码块中调用 wait() 和 notify() 方法一样，也就是说在调用 await() 和 signal() 时，如果没有持有适当的锁，就会抛出 InterruptedException 异常。

### 多个Condition实现部分线程通知

使用 Condition 对象的一个最大的好处就是我们可以实现多路通知功能了，即我们可以创建多个 Condition 实例，线程对象可以注册在指定的 Condition 中，从而实现有选择地进行线程通知，这样可以更灵活的调度线程。

**示例：**两个线程处于 await 状态，使用 signalAll 通知一个再启动，不通知另一个，看程序是否会结束，即看看线程能不能分类通知是否启动。

文件 MyService.java 代码如下：

```java
public class MyService {
    private Lock lock = new ReentrantLock();
    private Condition conditionA = lock.newCondition();
    private Condition conditionB = lock.newCondition();

    public void await_A() {
        try {
            lock.lock();
            System.out.println("begin await_A, time=" + System.currentTimeMillis()
                    + ", ThreadName=" + Thread.currentThread().getName());
            conditionA.await();
            System.out.println("end await_A, time=" + System.currentTimeMillis()
                    + ", ThreadName=" + Thread.currentThread().getName());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void signalAll_A() {
        try {
            lock.lock();
            System.out.println("signalAll_A, time=" + System.currentTimeMillis()
                    + ", ThreadName=" + Thread.currentThread().getName());
            conditionA.signalAll();
        } finally {
            lock.unlock();
        }
    }

    public void await_B() {
        try {
            lock.lock();
            System.out.println("begin await_B, time=" + System.currentTimeMillis()
                    + ", ThreadName=" + Thread.currentThread().getName());
            conditionB.await();
            System.out.println("end await_B, time=" + System.currentTimeMillis()
                    + ", ThreadName=" + Thread.currentThread().getName());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void signalAll_B() {
        try {
            lock.lock();
            System.out.println("signalAll_B, time=" + System.currentTimeMillis()
                    + ", ThreadName=" + Thread.currentThread().getName());
            conditionB.signalAll();
        } finally {
            lock.unlock();
        }
    }
}
```

文件 ThreadA.java 代码如下：

```java
public class ThreadA extends Thread {
    private MyService service;

    public ThreadA(MyService service) {
        super();
        this.service = service;
        this.setName("ThreadA");
    }

    @Override
    public void run() {
        service.await_A();
    }
}
```

文件 ThreadB.java 代码如下：

```java
public class ThreadB extends Thread {
    private MyService service;

    public ThreadB(MyService service) {
        super();
        this.service = service;
        this.setName("ThreadB");
    }

    @Override
    public void run() {
        service.await_B();
    }
}
```

文件 Run.java 代码如下：

```java
public class Run {
    public static void main(String[] args) throws InterruptedException {
        MyService service = new MyService();
        new ThreadA(service).start();
        new ThreadB(service).start();
        Thread.sleep(3000);
        service.signalAll_A();
    }
}
```

运行结果：

```
begin await_A, time=1533963611003, ThreadName=ThreadA
begin await_B, time=1533963611014, ThreadName=ThreadB
signalAll_A, time=1533963614014, ThreadName=main      // main线程发送A线程再次启动的通知
end await_A, time=1533963614014, ThreadName=ThreadA   // A线程接到main线程发送通知再次启动
// 没有线程通知B线程结束，所以程序不会停止，而是一直处于运行状态，因为B线程还在await中
```

### Condition实现生产者/消费者模式

**一对一交替打印：**

文件 MyService.java 代码如下：

```java
public class MyService {
    private Lock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();
    private boolean hasValue = false;

    public void set() {
        try {
            lock.lock();
            while (hasValue) { // Note1: if --> while
                condition.await();
            }
            System.out.println("set");
            hasValue = true;
            condition.signalAll(); // Note2: signal --> signAll
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void get() {
        try {
            lock.lock();
            while (!hasValue) { // Note3: if --> while
                condition.await();
            }
            System.out.println("get");
            hasValue = false;
            condition.signalAll(); // Note4: signal --> signAll
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

文件 ThreadA.java 代码如下：

```java
public class ThreadA extends Thread {
    private MyService service;

    public ThreadA(MyService service) {
        super();
        this.service = service;
    }

    @Override
    public void run() {
        while (true) {
            service.set();
        }
    }
}
```

文件 ThreadB.java 代码如下：

```java
public class ThreadB extends Thread {
    private MyService service;

    public ThreadB(MyService service) {
        super();
        this.service = service;
    }

    @Override
    public void run() {
        while (true) {
            service.get();
        }
    }
}
```

文件 Run.java 代码如下：

```java
public class Run {
    public static void main(String[] args) {
        MyService service = new MyService();
        new ThreadA(service).start();
        new ThreadB(service).start();
    }
}
```

运行结构：

```
...
set
get
set
get
set
... // 实现交替打印 set & get
```

## 公平锁与非公平锁

**公平锁：**线程获取锁的顺序按照线程加锁的顺序分配，即先进先出(FIFO)。

**非公平锁：**线程随机获得锁，是一种锁抢占机制，不过可能会造成某些线程一直拿不到锁。

**如何创建公平锁与非公平锁：**

ReentrantLock 有带 boolean 型参数 isFair 的构造器：

```java
public ReentrantLock(boolean fair) {}
```

可以选择创建公平锁 (true)，还是非公平锁 (false)。

## Lock的方法们

这些对象都是`Lock lock = new ReentrantLock()`中 lock 对象的方法。

### 计数类

- `getHoldCount()`：查询当前线程线程保持此锁定的个数，就是调用 lock() 的次数。
- `getQueueLength()`：返回正在争夺此锁的估计数。
- `getWaitQueueLength(Condition condition)`：返回等待与此锁有关的的给定 condition 的线程的估计数。

### has类

- `hasQueuedThread(Thread thread)`：查询 thread 线程是否在等待获取此锁。
- `hasQueuedThreads()`：查询是否有线程在等待获取此锁。
- `hasWaiters(Condition condition)`：查询是否有线程在等待与此锁有关的 condition 条件。

### 有关lock的属性类

- `isFair()`：判断此锁是不是公平锁。
- `isHeldByCurrentThread()`：查询当前线程是否保持此锁。
- `isLocked()`：查询此锁是否由任意线程保持。

### lock()方法的各种增强版方法（重要）

- `lockInterruptibly()`：如果当前线程未被中断，获取锁定，如果当前线程已有中断标记，则抛出异常。
- `boolean tryLock()`：调用时如果锁并没有被其他线程保持，才获取该锁。
- `boolean tryLock(long timeout, TimeUnit unit)`：在给定的时间中，锁也没有被其他线程保持，才获取该锁。就是说我想拿这个锁，这个锁现在也没人拿，不过我也不拿，我要等一会，如果等够了时间锁还没人拿，我在把锁拿走。

**lockInterruptibly()：**

使用 lockInterruptibly()，如果拿到这个锁的线程已经被打上了中断标记，就会直接抛出 InterruptedException 异常，否则就算是打上了标记，不自己在程序中用 interrupted() 来判断并抛异常，线程也是不会停的，只是有个标记而已，而 lockInterruptibly() 方法可以帮我们省去判断中断标记抛异常这些步骤。

*通过代码演示如下：*

文件 MyService.java 代码如下：

```java
public class MyService {
    public ReentrantLock lock = new ReentrantLock();

    public void waitMethod() {
        try {
            lock.lock(); // Note
            System.out.println("lock begin " + Thread.currentThread().getName());
            for (int i = 0; i < Integer.MAX_VALUE / 10; i++) {
                String str = new String();
                Math.random();
            }
            System.out.println("lock end " + Thread.currentThread().getName());
        } finally {
            if (lock.isHeldByCurrentThread()) { // 这里要判断一下当前线程是否持有锁
                lock.unlock();
            }
        }
    }
}
```

文件 Run.java 代码如下：

```java
public class Run {
    public static void main(String[] args) throws InterruptedException {
        MyService service = new MyService();
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                service.waitMethod();
            }
        };

        Thread threadA = new Thread(runnable);
        threadA.setName("A");
        threadA.start();
        Thread.sleep(500);
        Thread threadB = new Thread(runnable);
        threadB.setName("B");
        threadB.start();
        threadB.interrupt();
        System.out.println("main end");
    }
}
```

运行结果：

```
lock begin A
main end
lock end A
lock begin B  // 可以发现，B仍然好好运行着呢
lock end B
```

可以发现，B 仍然好好运行着呢，接下来我们把 MyService.java 中 Note 处的`lock.lock();`替换成`lock.lockInterruptibly();`，可以得到如下运行结果：

```
lock begin A
main end
java.lang.InterruptedException
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.
		acquireInterruptibly(AbstractQueuedSynchronizer.java:1220)
	at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335)
	at ch4.lockinterrupt.MyService.waitMethod(MyService.java:10)
	at ch4.lockinterrupt.Run$1.run(Run.java:9)
	at java.lang.Thread.run(Thread.java:748)
lock end A
```

可以发现 B 被中断了。

**boolean tryLock()：**

文件 MyService.java 代码如下：

```java
public class MyService {
    private ReentrantLock lock = new ReentrantLock();

    public void waitMethod() {
        if (lock.tryLock()) {
            System.out.println(Thread.currentThread().getName() + "获得锁");
        } else {
            System.out.println(Thread.currentThread().getName() + "没获得锁");
        }
    }
}
```

文件 Run.java 代码如下：

```java
public class Run {
    public static void main(String[] args) {
        MyService service = new MyService();
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                service.waitMethod();
            }
        };

        Thread threadA = new Thread(runnable);
        threadA.setName("A");
        threadA.start();
        Thread threadB = new Thread(runnable);
        threadB.setName("B");
        threadB.start();
    }
}
```

运行结果：

```
A获得锁
B没获得锁 // B tryLock()时，lock被A持有，B就无法获得锁了
```

### await()方法的增强版

- `awaitUninterruptibly()`：不能被中断的 await() 方法，正常在 await() 中的线程被中断了就立即会抛异常，然后线程就真的断了，用 awaitUninterruptibly() 等待的线程是无视中断操作的，就是说不能通过中断然这个等待中的线程抛异常停止。
- `awaitUntil(Date deadline)`：等待到 deadline 的时间，如果没人唤醒它就自己唤醒，如果有线程在 deadline 前唤醒它，就提前唤醒。

**awaitUninterruptibly()：**

文件 MyService.java 代码如下：

```java
public class MyService {
    private ReentrantLock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();

    public void testMethod() {
        try {
            lock.lock();
            System.out.println("wait begin");
            condition.await();  // Note
            System.out.println("wait end");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

文件 Run.java 代码如下：

```java
public class Run {
    public static void main(String[] args) throws InterruptedException {
        MyService service = new MyService();
        Thread thread = new Thread() {
            @Override
            public void run() {
                service.testMethod();
            }
        };
        thread.start();
        Thread.sleep(3000);
        thread.interrupt();
    }
}
```

运行结果：

```
wait begin
java.lang.InterruptedException // await中interrupt线程，抛异常！
	...
```

把 MyService.java 中 Note 处的`condition.await();`替换成`condition.awaitUninterruptibly();`，得到如下结果：

```
wait begin
// 只输出一行就永远卡住了，即使在await中，线程也不会被interrupt中断掉
```

## 使用Condition实现线程的顺序执行

假设有 Condition： conditionA，conditionB，conditionC 和 Thread：threadA，threadB，threadC，使三个线程按照 A - B - C 的顺序执行：(注意代码中的 Note 处，是方法的关键)

文件 Run.java 代码如下：

```java
public class Run {
    volatile private static int nextPrint = 1;      // Note0
    private static Lock lock = new ReentrantLock();
    private static Condition conditionA = lock.newCondition();
    private static Condition conditionB = lock.newCondition();
    private static Condition conditionC = lock.newCondition();

    public static void main(String[] args) {
        Thread threadA = new Thread() {
            @Override
            public void run() {
                try {
                    lock.lock();
                    while (nextPrint != 1) {
                        conditionA.await();         // NoteA1
                    }
                    System.out.println("threadA");
                    nextPrint = 2;                  // NoteA2
                    conditionB.signalAll();         // NoteA3
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        };

        Thread threadB = new Thread() {
            @Override
            public void run() {
                try {
                    lock.lock();
                    while (nextPrint != 2) {
                        conditionB.await();         // NoteB1
                    }
                    System.out.println("threadB");
                    nextPrint = 3;                  // NoteB2
                    conditionC.signalAll();         // NoteB3
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        };

        Thread threadC = new Thread() {
            @Override
            public void run() {
                try {
                    lock.lock();
                    while (nextPrint != 3) {
                        conditionC.await();         // NoteC1
                    }
                    System.out.println("threadC");
                    nextPrint = 1;                  // NoteC2
                    conditionA.signalAll();         // NoteC3
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        };

        Thread[] threadAs = new Thread[5];
        Thread[] threadBs = new Thread[5];
        Thread[] threadCs = new Thread[5];
        for (int i = 0; i < 5; i++) {
            threadAs[i] = new Thread(threadA);
            threadBs[i] = new Thread(threadB);
            threadCs[i] = new Thread(threadC);
            threadAs[i].start();
            threadBs[i].start();
            threadCs[i].start();
        }
    }
}
```

# ReentrantReadWriteLock类

## ReentrantReadWriteLock的操作特点

- 共享操作：读读
- 互斥操作：写写，读写，写读

## ReentrantReadWriteLock操作方法

```java
private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

lock.readLock().lock();    // 获得读锁
lock.readLock().unlock();  // 释放读锁

lock.writeLock().lock();   // 获得写锁
lock.writeLock().unlock(); // 释放写锁
```

