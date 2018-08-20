# synchronized同步

"非线程安全"是我们要考虑的一个问题，"非线程安全"可能会导致数据的脏读，使得程序运行产生错误的结果。

"非线程安全"问题一般存在于"实例变量"中，因为实例变量是有可能被多个线程所共享的，而方法内部的私有变量对于每个线程都是有一个副本的，因此不存在"非线程安全"问题，也就是说只有共享资源的读写问题才需要我们通过"同步化"去解决线程安全问题。

**线程安全包括原子性和可见性两个方面**。

synchronized：同步，加个前缀 a-，就表示相反的意思了，asynchronized：异步。

首先我们先来看看没有非线程安全的程序运行起来会导致的问题。

## 非线程安全问题是如何出现的

先搭建环境如下：

文件 PublicVar.java 代码如下：

```java
public class PublicVar {
    public String username = "C";
    public String password = "CC";

    public void setValue(String username, String password) {
        try {
            if (username.equals("A")) {
                this.username = username;
                Thread.sleep(10000); // 在tha sleep时，thb改变了username的值，出现了数据不同步的问题
                this.password = password;
            } else {
                this.username = username;
                this.password = password;
            }
            System.out.println("Thread: " + Thread.currentThread().getName() + " setValue method" +
                    " username=" + this.username + " password=" + this.password);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public void getValue() {
        System.out.println("Thread: " + Thread.currentThread().getName() + " getValue method" +
                " username=" + username + " password=" + password);
    }
}
```

文件 ThreadA.java 代码如下：

```java
public class ThreadA extends Thread {
    private PublicVar publicVar;

    public ThreadA(PublicVar publicVar) {
        super();
        this.publicVar = publicVar;
    }

    @Override
    public void run() {
        super.run();
        publicVar.setValue("A", "AA");
    }
}
```

文件 ThreadB.java 代码如下：

```java
public class ThreadB extends Thread {
    private PublicVar publicVar;

    public ThreadB(PublicVar publicVar) {
        super();
        this.publicVar = publicVar;
    }

    @Override
    public void run() {
        super.run();
        publicVar.setValue("B", "BB");
    }
}
```

文件 Main.java 代码如下：

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        PublicVar publicVar = new PublicVar();
        ThreadA tha = new ThreadA(publicVar);
        tha.setName("Thread A");
        tha.start();
        Thread.sleep(5000);
        ThreadB thb = new ThreadB(publicVar);
        thb.setName("Thread B");
        thb.start();
    }
}
```

### 赋值时的线程安全问题

运行如上代码，运行结果如下：

```
Thread: Thread B setValue method username=B password=BB
Thread: Thread A setValue method username=B password=AA \\ 产生数据不同步问题
```

解决方法：把 setValue 方法改成同步的：

```java
synchronized public void setValue(String username, String password) {...}
```

### 取值时的线程安全问题(脏读)

修改文件 ThreadB.java 为：

```java
public class ThreadB extends Thread {
    private PublicVar publicVar;

    public ThreadB(PublicVar publicVar) {
        super();
        this.publicVar = publicVar;
    }

    @Override
    public void run() {
        super.run();
        publicVar.getValue();
    }
}
```

得到运行结果如下：

```
Thread: Thread B getValue method username=A password=CC
Thread: Thread A setValue method username=A password=AA
```

解决方法：把 getValue 方法改成同步的：

```java
synchronized public void getValue() {...}
```

也就是说，对于共享数据，其读写必须是同步的，多个线程可以同时读取，但是只要有一个线程在执行写入操作，其他线程就不能再进行任何读写操作了，必须等待执行写入操作的线程执行完才行。(在这里我们先不考虑效率问题，只保证结果正确)

我们主要采取的解决问题方式就是：借助 synchronized 关键字，synchronized 声明的方法或者代码块一定同步的，也就是是排队运行的。

## synchronized关键字

### synchronized的特性

- **synchronized 拥有锁重入的功能**，即当一个线程得到一个对象锁后，此时这个线程也是可以调用该对象其他被 synchronized 修饰的部分的

	- 如果不可锁重入的话，会造成死锁，造成死锁的原因如下：

		```java
		public class Service {
		    synchronized public void service1() {
		        System.out.println("service1");
		        service2(); // Note1
		    }
		    
		    synchronized public void service2() {
		        System.out.println("service2");
		           
		    }
		}
		```

		在 Note1 处，调用了 service2 方法，此时 service1 方法并没有结束，说明线程不会放开锁，可以执行 service2 也是需要锁的，如果线程无法再次获得锁，那么它将永远无法执行 service2，这样 service1 就永远不会结束，它也永远不会放开锁，程序的运行陷入死循环，导致死锁。

	- 当存在父子类继承关系时，子类是可以通过"可重入锁"调用父类的同步方法的

- 当一个线程执行出现异常时，其所持有的锁会自动释放

### synchronized使用方式

synchronized 有两种使用方式：

- `synchronized`修饰在方法前面，此时锁对象为当前对象本身，也就是`this`

	```java
	synchronized public void getValue() {...}
	```

- `synchronized (锁对象)`修饰在代码块前面

	- 可以指定锁对象为当前对象本身，即`synchronized (this)`
	- 也可以指定锁对象为其他自定义的对象，**注意：**如果要进行同步的多个代码块，那么这些需要进行同步的多个同步代码块的锁必须是**同一个对象**，即`synchronized (非this对象)`

> 这里可以引申出一个问题：如何判断 synchronized 方法使用的锁是 this？
>
> 方法：只要写一段代码，里面有一个同步方法，另一个方法中有一个 synchronized (this) 同步代码块，只要证明这两部分是同步的，就可以说明 synchronized 方法使用的锁是 this。

**为什么需要锁不是 this 对象的同步方法呢？**

因为这样可以提高程序的运行效率，如果在一个类中有很多个 synchronized 方法，这时虽然能实现同步，但是由于线程阻塞的原因，运行效率大大降低，如果我们根据同步需求将其分成需要进行同步几部分，每一部分用一个锁，这样就不会有这么多的方法的运行都需要抢夺一个锁，大大提高运行效率。

*不过要注意！！！使用`synchronized (非 this 对象 x)`进行同步时，x 必须是同一个对象，就是说基本上应该是成员变量而非局部变量，**锁不同的 synchronized 代码块是异步的**。

## 同步代码块放在非同步方法中执行产生的脏读问题

有线程安全问题的代码：

文件 MyOneList.java 代码如下：

```java
public class MyOneList {
    private List<String> list = new ArrayList<>();

    synchronized public void add(String data) {
        list.add(data);
    }

    synchronized int getSize() {
        return list.size();
    }
}
```

文件 MyService.java 代码如下：

```java
public class MyService {
    public void addServiceMethod(MyOneList list, String data) {
        try {
            synchronized (list) {
                if (list.getSize() < 1) {
                    Thread.sleep(2000);
                    list.add(data);
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

文件 MyThread1.java 代码如下：

```java
public class MyThread1 extends Thread {
    private MyOneList list;

    public MyThread1(MyOneList list) {
        super();
        this.list = list;
    }

    @Override
    public void run() {
        MyService service = new MyService();
        service.addServiceMethod(list, "thread1");
    }
}
```

文件 MyThread2.java 代码如下：

```java
public class MyThread2 extends Thread {
    private MyOneList list;

    public MyThread2(MyOneList list) {
        super();
        this.list = list;
    }

    @Override
    public void run() {
        MyService service = new MyService();
        service.addServiceMethod(list, "thread2");
    }
}
```

文件 Main.java 代码如下：

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        MyOneList list = new MyOneList();
        MyThread1 thread1 = new MyThread1(list);
        thread1.setName("thread1");
        thread1.start();
        MyThread2 thread2 = new MyThread2(list);
        thread2.setName("thread2");
        thread2.start();
        Thread.sleep(6000);
        System.out.println("listSize=" + list.getSize());
    }
}
```

运行结果如下：

```
listSize=2
```

出错原因：MyService.java Note1 处，当 thread1 执行到时延时，list 的长度仍为 0，此时 thread2 已经可以调用 getSize() 方法了，因此 thread2 也可以进入 if 代码块中，就会导致 list 进行了两次 add 操作。

解决方法：修改 MyService.java 文件为：

```java
public class MyService {
    public void addServiceMethod(MyOneList list, String data) {
        try {
            synchronized (list) { // 把if用synchronized代码块包起来，因为这里只有一个list对象，用其做锁很合适
                if (list.getSize() < 1) {
                    Thread.sleep(2000);
                    list.add(data);
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

## `synchronized (非this对象)`的三个结论

假设 x 是一个非 this 对象，那么有以下三个结论：

1. 多个线程执行`synchronized (x) {}`同步代码块时呈同步效果
2. 当其他线程执行 x 对象中 synchronized 同步方法时呈同步效果
3. 当其他线程执行 x 对象中`synchronized (this) {}`同步代码块时呈同步效果

*注意：如果其他线程调用 x 不加 synchronized 的方法时，还是异步调用。*

## 静态同步synchronized方法与synchronized(class)代码块

**先说结论：synchronized 关键字加到 static 方法上是给 Class 类上锁，而加到非 static 则是给对象上锁。**

也就是说：synchronized 关键字加到 static 方法和 synchronized (class) 代码块的作用是一样的。

可以通过如下代码证明：

文件 Service.java 代码如下：

```java
public class Service {
    synchronized public void method1() { // 普通同步方法
        System.out.println(Thread.currentThread().getName() + " begin.");
        System.out.println(Thread.currentThread().getName() + " end.");
    }

    synchronized public static void method2() { // 静态同步方法
        try {
            System.out.println(Thread.currentThread().getName() + " begin.");
            Thread.sleep(3000);
            System.out.println(Thread.currentThread().getName() + " end.");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void method3() {
        synchronized (Service.class) { // 用当前类的class类作为锁的同步代码块
            System.out.println(Thread.currentThread().getName() + " begin.");
            System.out.println(Thread.currentThread().getName() + " end.");
        }
    }
}
```

文件 Run.java 代码如下：

```java
public class Run {
    public static void main(String[] args) {
        Service service = new Service();
        Thread thread2 = new Thread() {
            @Override
            public void run() {
                service.method2();
            }
        };
        thread2.setName("thread2");
        thread2.start();

        Thread thread1 = new Thread() {
            @Override
            public void run() {
                service.method1();
            }
        };
        thread1.setName("thread1");
        thread1.start();

        Thread thread3 = new Thread() {
            @Override
            public void run() {
                service.method3();
            }
        };
        thread3.setName("thread3");
        thread3.start();
    }
}
```

运行结果：

```
thread2 begin.
thread1 begin.
thread1 end.
thread2 end.
thread3 begin.
thread3 end.
```

观察运行结果可知，thread2 和 thread3 是同步的，和 thread1 是异步的。

## 用String做锁的坑

我们知道 JVM 中具有 String 常量池缓存的功能，这里我们先简单的介绍以下字符串常量池：

> JVM 为了提高性能减小内存开销，在实例化字符串常量的时候进行了如下优化：
>
> - 为字符串开辟一个字符串常量池
> - 创建字符串常量时，首先检查字符串常量池是否存在该字符串
> 	- 存在该字符串，返回引用实例
> 	- 不存在，实例化该字符串并放入池中
>
> 字符串常量池实现的基础：
>
> - 字符串是不可变的，可以不用担心数据冲突进行共享
> - 运行时，实例创建的**全局字符串常量池中有一个表**，总是**为池中每个唯一的字符串对象维护一个引用**,这就意味着它们一直引用着字符串常量池中的对象，所以，在常量池中的这些字符串不会被垃圾收集器回收 

由于这个特性，如果我们用 String 作为锁，只要我们对两个 synchronized 同步代码块使用用相同的字符串做锁，这两个 synchronized 同步代码块就是同步的，这会比较容易带来问题，所以我们一般都不使用 String 作为锁，而用其他对象，比如`new Object()`，**同时要注意：只要对象不变，既使该对象的属性改变了，结果还是同步的**，比如我们使用一个`User user`作为锁，即使我们改变了`user.username`的值，结果仍是同步的。

## 产生死锁

### 如何产生死锁

死锁的产生方法：

- 有两个线程(th1, th2)，两个锁(lock1, lock2)
- th1 和 th2 按照如下顺序希求锁，两个线程在得到一个锁后都有相同时间的时延
	- th1：`lock1->lock2`
	- th2：`lock2->lock1`

**产生死锁的本质：**两个线程在需要对方掌握的资源同时又不肯放开自己手中的资源，由此陷入的胶着状态

**产生死锁的程序：**

文件 DeadLock.java 代码如下：

```java
public class DeadLock implements Runnable {
    private String username;
    private Object lock1 = new Object();
    private Object lock2 = new Object();

    public void setUsername(String username) {
        this.username = username;
    }

    @Override
    public void run() {
        if (username.equals("a")) {
            synchronized (lock1) {
                try {
                    System.out.println("username=" + username);
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (lock2) {
                    System.out.println(username + ": lock1->lock2");
                }
            }
        }
        if (username.equals("b")) {
            synchronized (lock2) {
                try {
                    System.out.println("username=" + username);
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (lock1) {
                    System.out.println(username + ": lock2->lock1");
                }
            }
        }
    }
}
```

文件 Run.java 代码如下：

```java
public class Run {
    public static void main(String[] args) throws InterruptedException {
        DeadLock deadLock = new DeadLock();
        Thread th1 = new Thread(deadLock);
        deadLock.setUsername("a");
        th1.start();
        Thread.sleep(100);
        Thread th2 = new Thread(deadLock);
        deadLock.setUsername("b");
        th2.start();
    }
}
```

结果：

```
username=a
username=b
// 而后程序运行停止，不再向下执行，陷入死锁，可以通过下一节的方法进行检测
```

### 如何检测死锁

可以通过 JDK 的自带工具`jps`和`jstack`来检测死锁。步骤如下：

- 使用`jps`命令得到 Run 线程的 id
    ```bash
    C:\Users\Bean\IdeaProjects\LearnSpring>jps
    13428 Launcher
    14852 Run
    16868
    2664 Jps
    ```

- `jstack -l [id值]`，查看结果
    ```
    C:\Users\Bean\IdeaProjects\LearnSpring>jstack -l 14852
    ...
    Java stack information for the threads listed above:
    ===================================================
    "Thread-1":
            at ch2.deadlock.DeadLock.run(DeadLock.java:36)
            - waiting to lock <0x00000000d6327928> (a java.lang.Object)
            - locked <0x00000000d6327938> (a java.lang.Object)
            at java.lang.Thread.run(Thread.java:748)
    "Thread-0":
            at ch2.deadlock.DeadLock.run(DeadLock.java:23)
            - waiting to lock <0x00000000d6327938> (a java.lang.Object)
            - locked <0x00000000d6327928> (a java.lang.Object)
            at java.lang.Thread.run(Thread.java:748)

    Found 1 deadlock.
    ```

