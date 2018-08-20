# SimpleDataFormat类的非线程安全处理

SimpleDateFormat 类主要负责日期转换与格式化，但是它是非线程安全的，如果只用一个 SimpleDateFormat 供给多个线程使用的话，极易出现日期转换错误的情况。

如下面这段代码，我们启动了 10 个线程用来转换日期，不过，这 10 个线程用的是一个 SimpleDateFormat 对象：

```java
package ch7.simpledateformat.formaterror;

public class MyThread extends Thread {
    private SimpleDateFormat simpleDateFormat;
    private String dateString;

    public MyThread(SimpleDateFormat simpleDateFormat, String dateString) {
        super();
        this.simpleDateFormat = simpleDateFormat;
        this.dateString = dateString;
    }

    @Override
    public void run() {
        try {
            // 先把字符串日期转换成Date
            Date dateRef = simpleDateFormat.parse(dateString);
            // 再把Date日期转换成字符串
            String newDateString = simpleDateFormat.format(dateRef).toString();
            // 检查转换回来的字符串和原字符串是否相等，如果有转换错的就打印出来
            if (!newDateString.equals(dateString)) {
                System.out.println("ThreadName=" + Thread.currentThread().getName() +
                        " 转换错误，把 " + dateString + " 转换成了 " + newDateString);
            }
        } catch (ParseException e) {
            e.printStackTrace();
        }
    }
}
```

```java
package ch7.simpledateformat.formaterror;

public class Run {
    public static void main(String[] args) {
        // 10个线程用的都是这个SimpleDateFormat对象
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd");
        String[] dateStrings = {"2000-01-01", "2000-01-02", "2000-01-03", "2000-01-04", "2000-01-05",
                "2000-01-06", "2000-01-07", "2000-01-08", "2000-01-09", "2000-01-10"};

        MyThread[] myThreads = new MyThread[10];
        for (int i = 0; i < 10; i++) {
            myThreads[i] = new MyThread(simpleDateFormat, dateStrings[i]);
            myThreads[i].setName("thread-" + (i + 1));
        }
        for (int i = 0; i < 10; i++) {
            myThreads[i].start();
        }
    }
}
```

这个错误出现的主要是因为 SimpleDateFormat 是线程不安全的，想要保证线程安全，我们必须给每一个线程准备一个 SimpleDateFormat 类，这就用到了 ThreadLocal，我们通过一个 DateTools 类来解决这个问题：

```java
package ch7.simpledateformat.resolveerror;

public class DateTools {
    private static ThreadLocal<SimpleDateFormat> tl = new ThreadLocal<>();

    public static SimpleDateFormat getSimpleDateFormat(String datePattern) {
        SimpleDateFormat simpleDateFormat = tl.get();
        if (simpleDateFormat == null) {
            simpleDateFormat = new SimpleDateFormat(datePattern);
            tl.set(simpleDateFormat);
        }
        return simpleDateFormat;
    }
}
```

然后在 MyThread#run() 中通过 DateTools 获取 SimpleDateFormat：

```java
package ch7.simpledateformat.resolveerror;

public class MyThread extends Thread {
    private String dateString;

    public MyThread(String dateString) {
        super();
        this.dateString = dateString;
    }

    @Override
    public void run() {
        try {
            // 通过DateTools获取SimpleDateFormat
            Date dateRef = DateTools.getSimpleDateFormat("yyyy-MM-dd").parse(dateString);
            String newDateString = DateTools.getSimpleDateFormat("yyyy-MM-dd").
                format(dateRef).toString();
            
            if (!newDateString.equals(dateString)) {
                System.out.println("ThreadName=" + Thread.currentThread().getName() +
                        " 转换错误，把 " + dateString + " 转换成了 " + newDateString);
            }
        } catch (ParseException e) {
            e.printStackTrace();
        }
    }
}
```

