# Java多线程概述

## 概念

**进程：**受操作系统管理的基本运行单元，可以理解成运行在内存中的 exe 文件。

**线程：**在进程中独立运行的子任务，被调用的时机是随机的。

## 实现多线程的基本方法

### 继承Thread类

受制于 Java 的单继承机制，并不是十分灵活。

**实现：**

```java
public class MyThread extends Thread {
    @Override
    public void run() {
        super.run();
        System.out.println("MyThread");
    }
}
```

**启动线程：**

```java
MyThread myThread = new MyThread();
myThread.start();
```

### 实现Runnable接口(常用)

**实现：**

```java
public class MyThread implements Runnable {
    @Override
    public void run() {
        super.run();
        System.out.println("MyThread");
    }
}
```

**启动线程：**

```java
Runnable runnable = new MyThread();
Thread thread = new Thread(runnable); // 启动还是要依靠Thread类的
thread.start();
```

这是因为 Thread 有一个`Thread(Runnable targer)`构造函数的缘故，而且 Thread 类也是实现了 Runnable 接口的，所以我们甚至可以将一个 Thread 对象的 run() 方法交给其他线程调用。

## synchronized关键字

synchronized 关键字可以用来修饰任意对象和方法，作用就是给它们加上锁，这样当多个线程要使用这个对象或调用这个方法时，要以排队的方式进行，也就是在同一时刻只能有一个线程拿到这把锁，其他线程在尝试拿到这把锁。

## Thread中的4个方法

### currentThread()

返回**代码段正在被哪个线程调用**的信息。

Thread 有 7 种构造函数，比较常用的有以下几种：

```java
public Thread();
public Thread(String name);
public Thread(Runnable target);
public Thread(Runnable target, String name);
```

可以看出，我们不仅可以创建一个全新的 Thread 对象，也可给 Thread 传入一个已经创建好的线程对象，这是因为 Thread 类中有一个：

```java
private Runnable target;
```

我们可以创建一个 Thread 对象 thread，然后用 thread 包裹着另一个线程对象 target，这样我们就不用写 thread 的 run 方法了，直接调用 target 的 run 方法就好，感觉上像是实现了线程本身与 run 方法执行的内容的分离。

这里有一个关于`Thread.currentThread().getName()`和`this.getName()`结果的不同的问题，详细的例子和下面的 isAlive() 中一起给出。

### isAlive()

判断当前线程是否处在活动状态。

**示例：**

CountOperate.java

```java
public class CountOperate extends Thread {
    public CountOperate() {
        System.out.println("CountOperate---begin");
        System.out.println("Thread.currentThread().getName()=" + Thread.currentThread().getName());
        System.out.println("Thread.currentThread().isAlive()=" + Thread.currentThread().isAlive());
        System.out.println("this.getName()=" + this.getName());
        System.out.println("this.isAlive()=" + this.isAlive());
        System.out.println("CountOperate---end");
        System.out.println("-------------------------------------------------");
    }

    @Override
    public void run() {
        System.out.println("run---begin");
        System.out.println("Thread.currentThread().getName()=" + Thread.currentThread().getName());
        System.out.println("Thread.currentThread().isAlive()=" + Thread.currentThread().isAlive());
        System.out.println("this.getName()=" + this.getName());
        System.out.println("this.isAlive()=" + this.isAlive());
        System.out.println("run---end");
        System.out.println("-------------------------------------------------");
    }
}
```

CountOperateMain.java

```java
public class CountOperateMain {
    public static void main(String[] args) throws InterruptedException {
        CountOperate countOperate = new CountOperate();
        Thread t1 = new Thread(countOperate);
        System.out.println("main is begin, t1.isAlive()=" + t1.isAlive());
        t1.setName("At1");
        t1.start();
        System.out.println("main is ending, t1.isAlive()=" + t1.isAlive());
        System.out.println("-------------------------------------------------");
        Thread.sleep(5000);
        System.out.println("main is ended, t1.isAlive()=" + t1.isAlive());
        System.out.println("-------------------------------------------------");
    }
}
```

Output：

```
CountOperate---begin
Thread.currentThread().getName()=main
Thread.currentThread().isAlive()=true
this.getName()=Thread-0
this.isAlive()=false
CountOperate---end
-------------------------------------------------
main is begin, t1.isAlive()=false
main is ending, t1.isAlive()=true
-------------------------------------------------
run---begin
Thread.currentThread().getName()=At1  // Note 1
Thread.currentThread().isAlive()=true  // Note 2
this.getName()=Thread-0  // Note 3
this.isAlive()=false  // Note 4
run---end
-------------------------------------------------
main is ended, t1.isAlive()=false
-------------------------------------------------
```

重点在于观察 Output 的结果，尤其是 Note 1 和 Note 3，Note 2 和 Note 4 的不同！！！

这种不同源自于 Thread.currentThread() 和 this 得到的根本就是不同的两个线程，Thread.currentThread() 得到的是 thread 线程对象，this 得到的是 countOperate 线程对象，thread 是活的，而 countOperate 是死的，countOperate 只是给 thread 线程提供了 run 方法而已，countOperate 根本就没有 start() 过。

### sleep(毫秒数)

让“正在执行的线程”即 Thread.currentThread() 返回的线程，休眠指定的毫秒数。

### getId()

取得线程的唯一标识。

## 停止线程

终止一个线程是一个复杂的问题，Java 中有以下 3 种终止正在运行的线程的方法：

- run() 方法执行完之后自动退出
- 用 stop() 方法强制退出 (强烈建议不使用，要作废了，还不安全)
- 用 interrupt() 方法中断线程 (比较温和，不会强行终止一个正在运行的线程)

### 判断线程是否为停止状态

想要安全的停止一个线程，用 interrupt() 是最好的方法，不过调用了 interrupt() 之后线程并不会停止，因为 interrupt() 只是给线程打上了一个终止的 tag，我们要在线程中不时检查它是不是已经被打上 tag，如果打上了 tag 在用一些其他办法终止线程，所以我们要能判断一个线程是否是停止状态，即检查线程是否被 interrupt() 打上 tag，我们有以下两种方法可以进行检测：

- `this.interrupted()`：测试当前线程是否已经是中断状态，**执行后将状态标志清除为 false，是 static 的**。
	- 就是说这个方法如果调用两次，第一次为 true 后会将状态标志设为 false，第二次调用就为 false 了。
- `this.isInterrupted()`：测试当前线程是否已经是中断状态，**执行后不状态清除状态标志，不是 static 的**。

### 安全停止线程的方法

#### 异常法

MyThread.java

```java
public class MyThread extends Thread {
    @Override
    public void run() {
        super.run();
        try {
            for (int i = 0; i < 500000; i++) {
                if (this.isInterrupted()) {
                    System.out.println("线程已经停止，准备退出");
                    throw new InterruptedException(); // 也可以在这用return终止，不过还是异常好
                }
                System.out.println("i=" + (i + 1));
            }
        } catch (InterruptedException e) {
            System.out.println("进入MyThread.java的catch块");
        }
    }
}
```

MyThreadMain.java

```java
public class MyThreadMain {
    public static void main(String[] args) {
        try {
            MyThread thread = new MyThread();
            thread.start();
            thread.sleep(2000);
            thread.interrupt();
        } catch (InterruptedException e) {
            System.out.println("main catch");
        }
        System.out.println("main end");
    }
}
```

Output：

```
...
i=104430
i=104431
i=104432
main end
线程已经停止，准备退出
进入MyThread.java的catch块
```

#### sleep中停止

如果在 sleep 状态下停止线程，会 catch InterruptedException 然后进入 catch 块，并清楚停止状态值使之为 false。

MyThread.java

```java
public class MyThread extends Thread {
    @Override
    public void run() {
        super.run();
        try {
            for (int i = 0; i < 500000; i++) {
                if (this.isInterrupted()) {
                    System.out.println("线程已经停止，准备退出");
                    throw new InterruptedException(); // 也可以在这用return终止，不过还是异常好
                }
                System.out.println("i=" + (i + 1));
            }
        } catch (InterruptedException e) {
            System.out.println("进入MyThread.java的catch块");
        }
    }
}
```

MyThreadMain.java

```java
public class MyThreadMain {
    public static void main(String[] args) {
        try {
            MyThread thread = new MyThread();
            thread.start();
            thread.sleep(2000);
            thread.interrupt();
        } catch (InterruptedException e) {
            System.out.println("main catch");
        }
        System.out.println("main end");
    }
}
```

Output：

```
...
i=104430
i=104431
i=104432
main end
线程已经停止，准备退出
进入MyThread.java的catch块
```

### stop暴力停止线程的弊端

调用 stop() 会抛出 java.lang.ThreadDeath 异常，不过该异常不需要显式捕捉。

使用 stop() 会导致怎样的异常呢？我们来看下面这个例子：

SynchronizedObject.java

```java
public class SynchronizedObject {
    private String username = "a";
    private String password = "aa";

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    synchronized public void printString(String username, String password) {
        try {
            this.username = username;
            Thread.sleep(100000);
            this.password = password;
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

SynchronizedObjectMain.java

```java
public class SynchronizedObjectMain {
    public static void main(String[] args) throws InterruptedException {
        SynchronizedObject object = new SynchronizedObject();
        Thread thread = new Thread() {
            @Override
            public void run() {
                object.printString("b", "bb");
            }
        };
        thread.start();
        Thread.sleep(500);
        thread.stop();
        System.out.println(object.getUsername() + " " + object.getPassword());
    }
}
```

Output：

```
b aa
```

在设置完 username 后，我们用 stop() 强制终止了线程，导致了用户名和密码的不一致，这种事务性问题是需要在 catch 块中进行一些回滚操作的，而用 stop() 终止线程并不能给我们这样的机会，所以 stop() 有可能会导致数据不同步。

## 暂停线程

- suspend()：暂停线程
- resume()：恢复线程

这两个方法，尤其 suspend() 用起来还是十分危险的，所以 suspend() 也在作废方法的范围中。使用不当会导致如下两个问题：

- 独占公共同步对象
- 数据不同步

这两个问题的详细说明如下：

### 独占公共同步对象

如果一个线程在同步块中 suspend 了(就是在 synchronized 块中 suspend 了)，它就会一直卡在那，还不放开锁，导致其他线程无法访问公共同步对象。

普通的例子很好举，只要在一个方法前加上 synchronized 修饰，然后在这个方法里面调用`Thread.currentThread.suspend();`即可，不过有一些比较隐蔽的同步块，也可能会导致独占公共同步对象，使得程序卡住。例子如下：

PrintSuspendThread.java

```java
public class PrintSuspendThread extends Thread {
    private long i = 0;

    @Override
    public void run() {
        while (true) {
            i++
            System.out.println(i);  // Note 1
        }
    }
}
```

PrintSuspendThreadMain.java

```java
public class PrintSuspendThreadMain {
    public static void main(String[] args) throws InterruptedException {
        PrintSuspendThread printSuspendThread = new PrintSuspendThread();
        printSuspendThread.start();
        Thread.sleep(1000);
        printSuspendThread.suspend();
        System.out.println("main end");
    }
}
```

如果不加 Note 1 那行，程序会输出："main end"，然后卡住，如果加了 Note 1 那行，程序根本无法输出"main end"就会被卡住。这是为什么呢？**因为：**程序被停在了 println 方法里面，println 的源码如下：

```java
public void println(int x) {
    synchronized (this) {
        print(x);
        newLine();
    }
}
```
也就是说 println 是同步的，它内部有一个同步锁，当 printSuspendThread 线程被 suspend 在了 println 方法的同步块中，它会一直拿着锁不释放，main 线程中的 println 方法拿不到锁，因此无法执行。

### 数据不同步

数据不同步的原因和 stop() 方法的影响很像，直接看 stop() 方法导致数据不同步的原因即可，把那的 stop() 换成 suspend() 影响是一样的。

## 线程的优先级

### 优先级的特性

**设置线程优先级：**`setPriorty(int newPriorty)`，可以设置为 1~10，如果不在这个范围会抛出 IllegalArgumentException 异常。

**得到线程优先级：**`getPriorty()`

**三个 Thread 中预设的优先级等级：**

```java
public final static int MIN_PRIORTY = 1;
public final static int NORM_PRIORTY = 5;
public final static int MAX_PRIORTY = 10;
```

**线程优先级有继承的特性：**子线程和父线程具有相同的优先级。

**优先级具有随机性：**优先级高只是抢 CPU 时间片时抢到的概率高而已，并不一定就执行的比优先级低的快。

### yield方法

介绍了优先级，有一个感觉类似于达到降低自己的优先级的方法：yield()。

它的作用是放弃当前的 CPU 资源，将它让给其他任务去占用 CPU 执行时间，不过刚放弃就立刻又抢到了 CPU 时间片也是又可能的。

## 守护线程

**定义：**一种特殊的线程，当进程中不存在非守护线程了，守护线程自动销毁。

**示例：**典型的守护线程是垃圾回收线程，当进程中没有非守护线程了，垃圾回收线程就没有存在的必要了，会自动销毁。

**设置方法：**`thread.setDaemon(true);`
