# join方法

## join()方法的功能

如下代码的作用是：Run 中的 main 线程要等待 myThread 线程的执行，在它执行完之后再执行一下再结束。

文件 MyThread.java 代码如下：

```java
public class MyThread extends Thread {
    @Override
    public void run() {
        try {
            int secondValue = (int) (Math.random() * 10000);
            System.out.println(secondValue);
            Thread.sleep(secondValue);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

文件 Run.java 代码如下：

```java
public class Run {
    public static void main(String[] args) {
        try {
            MyThread myThread = new MyThread();
            myThread.start();
            myThread.join();
            System.out.println("myThread执行完了，我再执行一下");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

运行结果：

```
3783
myThread执行完了，我再执行一下
```
## join过程中线程被中断

使用 join 方法中线程被中断的效果 = 使用 wait 方法中线程被中断的效果。因为 join 方法内部就是用 wait 方法实现的。

**我们通过如下程序来验证一下：**

**模块：**

- ThreadA：一个执行任务的线程。
- ThreadB：负责创建并启动 ThreadA，并等待 ThreadA 执行完毕后继续执行。
- ThreadC：负责中断 ThreadB 线程

**预计结果：**ThreadB 线程的 catch 块会捕捉到异常，ThreadB 被异常停止后，ThreadA和、 会继续执行。

文件 ThreadA.java 代码如下：

```java
public class ThreadA extends Thread {
    @Override
    public void run() {
        for (int i = 0; i < Integer.MAX_VALUE; i++) {
            String str = new String();
            Math.random();
        }
    }
}
```

文件 ThreadB.java 代码如下：

```java
public class ThreadB extends Thread {
    @Override
    public void run() {
        try {
            ThreadA threadA = new ThreadA();
            threadA.start();
            threadA.join();
            System.out.println("ThreadB运行结束了");
        } catch (InterruptedException e) {
            System.out.println("ThreadB被中断，抛了个异常被catch了");
            e.printStackTrace();
        }
    }
}
```

文件 ThreadC.java 代码如下：

```java
public class ThreadC extends Thread {
    private ThreadB threadB;

    public ThreadC(ThreadB threadB) {
        super();
        this.threadB = threadB;
    }

    @Override
    public void run() {
        threadB.interrupt();
    }
}
```

文件 Run.java 代码如下：

```java
public class Run {
    public static void main(String[] args) throws InterruptedException {
        ThreadB threadB = new ThreadB();
        threadB.start();
        Thread.sleep(500);
        ThreadC threadC = new ThreadC(threadB);
        threadC.start();
    }
}
```

运行结果：

```
ThreadB被中断，抛了个异常被catch了
java.lang.InterruptedException
	at java.lang.Object.wait(Native Method)
	at java.lang.Thread.join(Thread.java:1252)
	at java.lang.Thread.join(Thread.java:1326)
	at ch3.join.interrupt.ThreadB.run(ThreadB.java:9)
```
## join(long)和sleep(long)的区别

join 还有一个带参数的方法：join(long)，这个方法是等待传入的参数的毫秒数，如果计时过程中等待的方法执行完了，就接着往下执行，如果计时结束等待的方法还没有执行完，就不再继续等待，而是往下执行。

**那么 join(long) 和 sleep(long) 又有什么区别呢？**

- 如果等待的方法提前结束，join(long) 不会再计时了，而是往下执行，而 sleep(long) 一定要等待够足够的毫秒数。
- join(long) 会释放锁，sleep(long) 不会释放锁，原因是 join(long) 方法内部是用 wait(long) 方法实现的。

**join(long) 的内部实现原理：**

join() 的内部实现原理就是 join(0)，所以我们只看 join(long) 的实现原理就好。

```java
public final synchronized void join(long millis) throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;

    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (millis == 0) {
        while (isAlive()) {
            wait(0); // 本质是wait！！！
        }
    } else {
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
            wait(delay); // 本质是wait！！！
            now = System.currentTimeMillis() - base;
        }
    }
}
```

## join()后面的代码提前运行的意外

