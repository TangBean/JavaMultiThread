# ThreadLocal类

## ThreadLocal的作用

我们可以在类中定义 static 变量，这样这个类中的每个对象都会共享这个变量，现在我们想要对这个共享进行划分，划分的原则是：同一个线程共享一个变量，不同的线程共享另一个变量。想要实现这样的功能要用到：ThreadLocal 类。

ThreadLocal 有如下 3 个比较重要的方法：

- get()：获取当前线程的值。
- set()：设置当前线程的值。
- remove()：移除当前线程的值。

## ThreadLocal的原理

ThreadLocal 内部有一个 static 的内部类 ThreadLocalMap：

```java
static class ThreadLocalMap {...}
```

在 Thread 类内部有一个 ThreadLocal.ThreadLocalMap 类型的成员变量。

```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```

这个 Map 的 Entry 是这样的：

```java
Entry(ThreadLocal<?> k, Object v) {
    super(k);
    value = v;
}
```

也就是说是用 ThreadLocal 对象作为 key 存在每个线程中的，当一个线程它要获得 threadLocal 对象中存储的值时，它会调用这个 threadLocal 对象的 get 方法，get 方法会去这个线程中取得 threadLocals 这个 ThreadLocal.ThreadLocalMap，然后根据要获得的这个 threadLocal 对象去 map 中寻找值并返回。

## 解决第一次使用get()方法返回null问题

想要实现带初始值的 ThreadLocal，我们需要写一个 extends ThreadLocal 的新类，通过观察 ThreadLocal 中的 get() 方法和 setInitialValue() 方法可以发现：当第一次使用 ThreadLocal 类的 get() 方法时，最终得到的值是 initialValue() 的返回值。

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t); // 第一次使用时map为null
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue(); // 会返回Initial的结果
}

private T setInitialValue() {
    T value = initialValue(); // 来自initialValue()方法
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}

protected T initialValue() {
    return null; // 返回为null，所以初始值为null
}
```
所以想要一个带初始值的 ThreadLocal 方法，我们需要写一个 ThreadLocal 的子类，然后覆盖它的 initialValue() 方法，使其返回新的初始值即可。

```java
public class ThreadLocalExt extends ThreadLocal {
    @Override
    protected T initialValue() {
        return "我是新的默认初始值，有了我以后，第一次调用get()就不会返回null了";
    }
}
```

## InheritableThreadLocal类

使用 InheritableThreadLocal 可以在子线程中取得父线程在 ThreadLocal 中存的值，不过如果在子线程取得值的同时，父线程修改了 InheritableThreadLocal 中的值，那么子线程只能取得旧值。

同时子线程还可以通过覆盖 InheritableThreadLocal 的 childValue(Object parentValue) 方法对父线程设置的值进行修改：

```java
public class InheritableThreadLocalExt extends InheritableThreadLocal {
    @Override
    protected T initialValue() {
        return new Date().getTime();
    }
    
    @Override
    protected Object childValue(Object parentValue) {
        return parentValue + " 我是在子线程后加的！";
    }
}
```

