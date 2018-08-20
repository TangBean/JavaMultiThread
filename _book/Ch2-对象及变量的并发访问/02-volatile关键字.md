# volatile关键字

## volatile关键字介绍

volatile 的意思为：易变的，不稳定的。

**关键字 volatile 的主要作用是使变量在多个线程间可见，它强制这个变量从公共内存中取得变量的值，而不是从线程的私有数据栈中取得变量的值。**

想要了解 volatile 的作用，我们要先来介绍一下 Java 的内存机制，Java 的内存机制原理图如下：

![volatile内存原理图.png](http://ox7712i91.bkt.clouddn.com/volatile%E5%86%85%E5%AD%98%E5%8E%9F%E7%90%86%E5%9B%BE.png)

对于没有 volatile 修饰的变量，为了获得最佳速度，允许线程保存共享成员变量的私有拷贝，**而且只当线程进入或者离开同步代码块时才与共享成员变量的原始值对比**，也就是说，Java 使用一个主内存来保存变量当前值，而每个线程则有其独立的工作内存。线程访问变量的时候会将变量的值拷贝到自己的工作内存中，这样，当线程对自己工作内存中的变量进行操作之后，就造成了工作内存中的变量拷贝的值与主内存中的变量值不同。 

在这种情况下，如果我们运行如下结构的的代码程序将永远不会停止：

文件 Service.java 代码如下：

```java
public class Service {
    private boolean runFlag = true;

    public boolean isRunFlag() {
        return runFlag;
    }

    public void setRunFlag(boolean runFlag) {
        this.runFlag = runFlag;
    }

    public void runMethod() {
        while (runFlag) {
        }
        System.out.println("停止成功");
    }

    public void stopMethod() {
        runFlag = false;
    }
}
```

文件 ThreadA.java 代码如下：

```java
public class ThreadA implements Runnable {
    private Service service;

    public ThreadA(Service service) {
        super();
        this.service = service;
    }

    @Override
    public void run() {
        service.runMethod();
    }
}
```

文件 Run.java 代码如下：

```java
public class Run {
    public static void main(String[] args) throws InterruptedException {
        Service service = new Service();
        ThreadA tha = new ThreadA(service);
        new Thread(tha).start();
        Thread.sleep(1000);
        service.stopMethod(); // 在这里运行stopMethod()并没有修改正在运行的tha中的service.runFlag的值！
        System.out.println("已经发起停止命令");
    }
}
```

这是因为我们通过 Run.java 中的`service.stopMethod();`并没有修改正在运行的 tha 的私有堆栈中的 service.runFlag，而这个问题也正是由于公共堆栈和私有堆栈中的值不同步造成的。

当我们把 service.runFlag 用 volatile 修饰之后：(修改文件 Service.java 代码)

```java
private boolean runFlag = true; --> volatile private boolean runFlag = true;
```

即可解决这个问题。

也就是说，**volatile 关键字可以解决变量在多个线程之间的可见性问题，它可以让多线程读取共享变量时可以获得该变量的最新值**。

## volatile和synchronized的对比

显然，synchronized 要比 volatile 强大很多，它们之间有如下几点区别：

- volatile 是线程同步的轻量级实现，性能要好于 synchronized，但只能用来修饰变量，而 synchronized 可以用来修饰方法和代码块。
- volatile 只能保证数据的可见性，不能保证数据的原子性，而 synchronized 可以保证数据的原子性，并可以间接保证数据的可见性 (因为它会将私有内存和公共内存中的数据做同步)。
- 多线程访问 volatile 不会发生阻塞，而 synchronized 会出现阻塞。

> 原子性：原子操作是不能分割的整体，没有其他线程能够中断或检查正在原子操作中的变量。

## synchronized的可见性功能

修改文件 Service.java 代码如下：

```java
public class Service {
    private boolean runFlag = true; // 不用volatile修饰

    public boolean isRunFlag() {
        return runFlag;
    }

    public void setRunFlag(boolean runFlag) {
        this.runFlag = runFlag;
    }

    public void runMethod() {
     /* -------------------------------- */
        String anyString = new String();
        while (runFlag) {
            synchronized (anyString) {
            }
        }
     /* -------------------------------- */
        System.out.println("停止成功");
    }

    public void stopMethod() {
        runFlag = false;
    }
}
```

可以达到与 volatile 相同的效果。

## i++的非原子性

i++，就是 i = i + 1，这个操作虽然只有一行，不过是一个典型的非原子操作，它的操作步骤分解如下：

1. 从内存中取出 i 的值 (load)
2. 计算 i 加 1 的值 (use)
3. 将 i 的值写道内存中 (assign赋值)

这三步是典型的非原子操作，在多线程的环境中运行很容易出现问题。

## 使用原子类进行i++操作

原子类：`Atomic各种`，例如：`AtomicInteger`，`AtomicIntegerArray`，`AtomicBoolean`等。

**原子类可以在没有锁的情况下做到线程安全，不过原子类中的每个方法是一个原子操作，不过方法和方法之间并不是原子的。**

**示例：**

文件 AddCountThread.java 代码如下：

```java
public class AddCountThread implements Runnable {
    private AtomicInteger count = new AtomicInteger(0);

    @Override
    public void run() {
        for (int i = 0; i < 10000; i++) {
            System.out.println(count.incrementAndGet());
        }
    }
}
```

文件 Run.java 代码如下：

```java
public class Run {
    public static void main(String[] args) {
        AddCountThread countThread = new AddCountThread();
        for (int i = 0; i < 5; i++) {
            new Thread(countThread).start();
        }
    }
}
```

输出：

```
...
49997
49998
49999
50000 // 能加到50000，说明保证了count+1的原子性
```

