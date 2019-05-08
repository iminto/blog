---
title: Java里常见的几个语法小坑
date: 2018-05-05 14:18:23
tags: [Java]
---
很久没更新博客了，想到几个小坑，虽然没啥技术含量，但或许有人不知道呢。
#### 1.删除sublist的元素导致原对象元素被删除
看下面这段代码
```java
List<Integer> students=new ArrayList<Integer>();
        for (int i = 0; i <5 ; i++) {
            students.add(i);
        }
        List<Integer> subList=new ArrayList<Integer>();
        subList=students.subList(0,5);
        subList.remove(0);
        subList.remove(1);
        for (int i = 0; i <5 ; i++) {
            System.out.println(i+"="+students.get(i));
        }
```
students是个list，然后我们新建立了一个subList对象，这个对象截取了students的一部分，我们删除了subList对象里的一些元素，看下运行结果。
```java
0=1
1=3
2=4
Exception in thread "main" java.lang.IndexOutOfBoundsException: Index: 3, Size: 3
	at java.util.ArrayList.rangeCheck(ArrayList.java:657)
	at java.util.ArrayList.get(ArrayList.java:433)
	at bai.ListDo.main(ListDo.java:17)
```
难道说，删除subList对象里的元素也会导致students里的元素被删除？我明明是新建了一个对象啊。然而，事实确实是这样的。
我们要理解一个事情，使用new新建一个对象，只是开辟了一块空间，用来存放这个对象的地址指针，但是这个新建的对象地址，指向的却是原有对象，也就是说，使用subList这个方法的时候，并没有从students里把内容拷贝了一份，仅仅是纪录了一个指针的移动，这样从某种角度来说，是提高了性能节省内存的做法。
看一下subList这个方法的JavaDoc我们就更清楚了。
```java
Returns a view of the portion of this list between the specified
     * <tt>fromIndex</tt>, inclusive, and <tt>toIndex</tt>, exclusive.  (If
     * <tt>fromIndex</tt> and <tt>toIndex</tt> are equal, the returned list is
     * empty.)  The returned list is backed by this list, so non-structural
     * changes in the returned list are reflected in this list, and vice-versa.
     * The returned list supports all of the optional list operations supported
     * by this list.<p>
```
什么时候会用到subList方法呢，通常是接收到了一个大的list，需要切割成一个个小的子list再加工处理，以减少内存占用和提高性能，如果不注意的话，就很容易触发这种隐形的bug。所以，使用subList时不要轻易做增删操作，要么不使用subList方法，而是手动add.
#### 2.SimpleDateFormat的线程安全问题
很多博客和文章都会告诉我们，一定要注意SimpleDateFormat的线程安全问题，那究竟是怎么回事呢？
看下面的代码
```java
public class DateFormatTest extends Thread {
    @Override
    public void run() {
        while(true) {
            try {
                this.join(2000);
            } catch (InterruptedException e1) {
                e1.printStackTrace();
            }
            try {
                System.out.println(this.getName()+":"+DateUtil.parse("2018-05-05 12:12:12"));
            } catch (ParseException e) {
                e.printStackTrace();
            }
        }
    }
    public static void main(String[] args) {
        for(int i = 0; i < 3; i++){
            new DateFormatTest().start();
        }

    }
}

class DateUtil {

    private static final  SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    public static  String formatDate(Date date)throws ParseException{
        return sdf.format(date);
    }

    public static Date parse(String strDate) throws ParseException{
        return sdf.parse(strDate);
    }
}
```
运行这段代码后，会发现Thread-1会报出Exception in thread "Thread-0" Exception in thread "Thread-1" java.lang.NumberFormatException: multiple points 的异常，并且导致Thread-2有一些错误的日期输出。为什么呢，原因在于SimpleDataFormat不是线程安全的，因为SimpleDataFormat里面用了Calendar 这个成员变量来实现SimpleDataFormat,并且在Parse 和Format的时候对Calendar 进行了修改，calendar.clear()，calendar.setTime(date);
为了线程安全和效率的双重兼顾，建议使用ThreadLocal，代码如下：
```java
public class DateUtil1 {

    private static final ThreadLocal<DateFormat> messageFormat = new ThreadLocal<DateFormat>();
    private static final String MESSAGE_FORMAT = "MM-dd HH:mm:ss.ms";

    private static final DateFormat getDateFormat() {
        DateFormat format = messageFormat.get();
        if (format == null) {
            format = new SimpleDateFormat(MESSAGE_FORMAT, Locale.getDefault());
            messageFormat.set(format);
        }

        return format;
    }
}
```
如果自己没有把握的话，还是建议每次new一个SimpleDataFormat对象。
Java里面还有许多线程不安全的类，使用这些类的时候，务必注意使用同步原语，或者使用new新建一个对象省事，或者使用对应的线程安全的类。比如hashMap对应的ConcurrentHashMap.
#### 3.split的坑
看下面的代码，
```java
String[] re="2|33|4".split("|");
        for (int i = 0; i <re.length ; i++) {
            System.out.println(re[i]);
        }
```
你以为输出的结果会是2,33,4，实际上却是 2,|，3,3，|，4。为什么呢，稍微看一下split的方法注释就知道了，原来split的分隔符参数实际上是一个正则表达式，而不是普通的字符串。
所以，正确的写法应该是String.split("\\|")
当然，这种坑纯粹是由于对Java基本方法的使用不熟悉造成的，是完全可以避免的。
