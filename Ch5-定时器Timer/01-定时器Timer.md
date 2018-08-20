# 定时器Timer

Timer 类的作用就是设置计划任务，不过它是不能单独使用的，它只负责按照要求启动任务，我们需要把要执行的任务提供给它，负责提供任务的就是 TimerTask 类，TimerTask 是一个实现了 Runnable 接口的抽象类，我们要新建 TimerTask 的子类并在 run() 方法中编写我们要计划执行的任务，然后新建 Timer 对象，通过 Timer 对象启动它。

## 需要用到的类和一些方法技术

我们先实现一个 TimerTask 类的子类：

```java
public class MyTask extends TimerTask {
    @Override
    public void run() {
        System.out.println("执行任务");
    }
}
```

**根据当前时间获取Date类的方法：**

```java
Calendar calendarRef = Calendar.getInstance();
calendarRef.set(Calendar.SECOND, calendarRef.get(Calendar.SECOND) + 10);
Date runDate = calendarRef.getTime();
```

## 一个标准的Timer使用方法实例

```java
MyTask task = new MyTask(num);
Timer timer = new Timer(true); // 设置创建守护线程
timer.schedule(task, runDate);
```

Timer 有如下两类执行任务的方式。

## 用Date firstTime设置任务开始执行的时间

在 firstTime 时执行任务。

- **普通执行：**`schedule(TimerTask task, Date firstTime)`
- **周期执行：**`schedule(TimerTask task, Date firstTime, long period)`

## 用long delay设置任务开始执行的时间

即从当前时间开始，等待 delay 时间，然后执行任务。

- **普通执行：**`schedule(TimerTask task, long delay)`
- **周期执行：**`schedule(TimerTask task, long delay, long period)`

## 注意事项

- Timer 要创建成守护线程：`Timer timer = new Timer(true)`。
- 如果 Timer 执行任务时发现给定的时间已经过期了，会立即执行相和歌任务。
- 可以用一个 Timer 设置多个 TimerTask 的执行，这些 TimerTask 会被按照调用 schedule 的顺序执行，如果中间有一个 task 特别耗时导致在它后面执行的任务过期了，那么到它后面的任务执行时，这个 timer 会发现这个任务过期了，并立刻执行它，不过这会导致这个任务执行的时间和设定的不一样。

## 取消任务执行

- TimerTask 类的 cancel() 方法：将自身从任务队列中清除。
- Timer 类的 cancel() 方法：将任务队列中的全部任务清空。

不过 Timer 的 cancel() 有时会因为 Timer 类中的 cancel() 方法没有争抢到 queue 锁而取消不成功。*(见ch5.cancel.TimerCancelTest.java)*

## scheduleAtFixedRate()和schedule()的区别

**区别：**

- 有追赶执行性：scheduleAtFixedRate()
- 无追赶执行性：schedule()

**什么是追赶执行性：**

追赶执行性体现在任务给定的时间过期了的时候，有追赶执行性的任务会把从给定的时间到当前时间中漏执行的任务的全都补执行完成，再开始按周期执行，而无追赶执行性的任务只补执行一次的，就开始按周期执行。

*(见ch5.*fixrate.ScheduleTest.java)*