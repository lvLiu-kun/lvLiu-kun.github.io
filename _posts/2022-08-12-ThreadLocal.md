---
tags: Java ThreadLocal
key: threadLocal-01
---

# **工作以及面试中ThreadLocal的八个关键知识点**

## 清单

1. `ThreadLocal`是什么？为什么要使用`ThreadLocal`？
2. 日期转换工具类中使用`ThreadLocal`案例引出探讨
3. `ThreadLocal`的原理
4. 为什么不直接用线程id作为`ThreadLocalMap`的key
5. 为什么会导致内存泄漏呢？是因为弱引用吗？
6. Key为什么要设计成弱引用呢？强引用不行？
7. `InheritableThreadLocal`保证父子线程间的共享数据
8. `ThreadLocal`的应用场景和使用注意点

## 1. ThreadLocal是什么？为什么要使用ThreadLocal？

> **`ThreadLocal`是什么？**

`ThreadLocal`，即线程本地变量。创建一个`ThreadLocal`变量，每个访问这个变量的线程都会本地拷贝这个变量，多个线程操作这个变量的时候，实际操作的是自己本地内存里面的变量，从而起到**线程隔离**的作用，避免了并发场景下的线程安全问题。

```java
//创建一个ThreadLocal变量
static ThreadLocal<String> localVariable = new ThreadLocal<>();
```

> **为什么要使用`ThreadLocal`**？

并发场景下，存在多个线程同时修改一个共享变量的场景时，会出现**线性安全问题**。

为了解决线性安全问题，可以用加锁的方式，比如使用`synchronized` 或者`Lock`。但是加锁的方式，会导致系统变慢。

加锁示意图如下：

![640](https://bylv2727.oss-cn-shenzhen.aliyuncs.com/PicGo/blog/2022-08-12-ThreadLocal/640.png)

使用`ThreadLoca`以空间换时间的方式，即`ThreadLocal`类访问共享变量时，会在每个线程的本地，都保存一份共享变量的拷贝副本。多线程对共享变量修改时，实际上操作的是这个变量副本，从而保证线性安全。

![640 (1)](https://bylv2727.oss-cn-shenzhen.aliyuncs.com/PicGo/blog/2022-08-12-ThreadLocal/640%20(1).png)

## 2. 日期转换工具类中使用ThreadLocal案例引出探讨

**反例演示**：

```
/**
 * 日期工具类
 */
public class DateUtil {

    private static final SimpleDateFormat simpleDateFormat =
            new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    public static Date parse(String dateString) {
        Date date = null;
        try {
            date = simpleDateFormat.parse(dateString);
        } catch (ParseException e) {
            e.printStackTrace();
        }
        return date;
    }
}
```

在多线程环境跑`DateUtil`这个工具类：

```
public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(10);

        for (int i = 0; i < 10; i++) {
            executorService.execute(()->{
                System.out.println(DateUtil.parse("2022-07-24 16:34:30"));
            });
        }
        executorService.shutdown();
    }
```

报错如下：

![640 (2)](https://bylv2727.oss-cn-shenzhen.aliyuncs.com/PicGo/blog/2022-08-12-ThreadLocal/640%20(2).png)

在`DateUtil`工具类，加上`ThreadLocal`：

```
/**
 * 日期工具类
 */
public class DateUtil {

    private static ThreadLocal<SimpleDateFormat> dateFormatThreadLocal =
            ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

    public static Date parse(String dateString) {
        Date date = null;
        try {
            date = dateFormatThreadLocal.get().parse(dateString);
        } catch (ParseException e) {
            e.printStackTrace();
        }
        return date;
    }

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(10);

        for (int i = 0; i < 10; i++) {
            executorService.execute(()->{
                System.out.println(DateUtil.parse("2022-07-24 16:34:30"));
            });
        }
        executorService.shutdown();
    }
}
```

运行结果：

```
Sun Jul 24 16:34:30 GMT+08:00 2022
Sun Jul 24 16:34:30 GMT+08:00 2022
Sun Jul 24 16:34:30 GMT+08:00 2022
Sun Jul 24 16:34:30 GMT+08:00 2022
Sun Jul 24 16:34:30 GMT+08:00 2022
Sun Jul 24 16:34:30 GMT+08:00 2022
Sun Jul 24 16:34:30 GMT+08:00 2022
Sun Jul 24 16:34:30 GMT+08:00 2022
Sun Jul 24 16:34:30 GMT+08:00 2022
Sun Jul 24 16:34:30 GMT+08:00 2022
```

**反例**报错这是因为`SimpleDateFormat`不是线性安全的，它以共享变量出现时，并发多线程场景下会报错。

接下来探讨

* 加了`ThreadLocal`为何没有有问题？
* 并发场景下`ThreadLocal`是如何保证的？

## 3. ThreadLocal的原理

### 3.1 ThreadLocal的内存结构图

`ThreadLocal`的内存结构图

![640 (3)](https://bylv2727.oss-cn-shenzhen.aliyuncs.com/PicGo/blog/2022-08-12-ThreadLocal/640%20(3).png)

从内存结构图可以看到：

- `Thread`类中，有个`ThreadLocal.ThreadLocalMap` 的成员变量。
- `ThreadLocalMap`内部维护了`Entry`数组，每个`Entry`代表一个完整的对象，`key`是`ThreadLocal`本身，`value`是`ThreadLocal`的泛型对象值。

### 3.2 关键源码分析

对照着几段关键源码来看，更容易理解一点哈~我们回到`Thread`类源码，可以看到成员变量`ThreadLocalMap`的初始值是为`null`

```
public class Thread implements Runnable {
   //ThreadLocal.ThreadLocalMap是Thread的属性
   ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

`ThreadLocalMap`的关键源码如下：

```
static class ThreadLocalMap {
    
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    //Entry数组
    private Entry[] table;
    
    // ThreadLocalMap的构造器，ThreadLocal作为key
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }
}
```

`ThreadLocal`类中的关键`set()`方法：

```
 public void set(T value) {
        Thread t = Thread.currentThread(); //获取当前线程t
        ThreadLocalMap map = getMap(t);  //根据当前线程获取到ThreadLocalMap
        if (map != null)  //如果获取的ThreadLocalMap对象不为空
            map.set(this, value); //K，V设置到ThreadLocalMap中
        else
            createMap(t, value); //创建一个新的ThreadLocalMap
    }
    
     ThreadLocalMap getMap(Thread t) {
       return t.threadLocals; //返回Thread对象的ThreadLocalMap属性
    }

    void createMap(Thread t, T firstValue) { //调用ThreadLocalMap的构造函数
        t.threadLocals = new ThreadLocalMap(this, firstValue); this表示当前类ThreadLocal
    }
    
```

`ThreadLocal`类中的关键`get()`方法

```
    public T get() {
        Thread t = Thread.currentThread();//获取当前线程t
        ThreadLocalMap map = getMap(t);//根据当前线程获取到ThreadLocalMap
        if (map != null) { //如果获取的ThreadLocalMap对象不为空
            //由this（即ThreadLoca对象）得到对应的Value，即ThreadLocal的泛型值
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value; 
                return result;
            }
        }
        return setInitialValue(); //初始化threadLocals成员变量的值
    }
    
     private T setInitialValue() {
        T value = initialValue(); //初始化value的值
        Thread t = Thread.currentThread(); 
        ThreadLocalMap map = getMap(t); //以当前线程为key，获取threadLocals成员变量，它是一个ThreadLocalMap
        if (map != null)
            map.set(this, value);  //K，V设置到ThreadLocalMap中
        else
            createMap(t, value); //实例化threadLocals成员变量
        return value;
    }
```

所以怎么回答`ThreadLocal`的实现原理？如下：

- `Thread`线程类有一个类型为`ThreadLocal.ThreadLocalMap`的实例变量`threadLocals`，即每个线程都有一个属于自己的`ThreadLocalMap`。
- `ThreadLocalMap`内部维护着`Entry`数组，每个`Entry`代表一个完整的对象，`key`是`ThreadLocal`本身，`value`是`ThreadLocal`的泛型值。
- 并发多线程场景下，每个线程`Thread`，在往`ThreadLocal`里设置值的时候，都是往自己的`ThreadLocalMap`里存，读也是以某个`ThreadLocal`作为引用，在自己的`map`里找对应的`key`，从而可以实现了**线程隔离**。

接下来探讨`ThreadLocalMap`为什么要用`ThreadLocal`作为key，以及直接用`线程Id`为什么不一样。

## **4. 为什么不直接用线程id作为ThreadLocalMap的key呢？**

举个代码例子，如下：

```
public class TianLuoThreadLocalTest {

    private static final ThreadLocal<String> threadLocal1 = new ThreadLocal<>();
    private static final ThreadLocal<String> threadLocal2 = new ThreadLocal<>();
 
}
```

这种场景：一个使用类，有两个共享变量，也就是说用了两个`ThreadLocal`成员变量的话。如果用线程`id`作为`ThreadLocalMap`的`key`，怎么区分哪个`ThreadLocal`成员变量呢？因此还是需要使用`ThreadLocal`作为`Key`来使用。每个`ThreadLocal`对象，都可以由`threadLocalHashCode`属性**唯一区分**的，每一个`ThreadLocal`对象都可以由这个对象的名字唯一区分（**下面的例子**），看下`ThreadLocal`代码：

```
public class ThreadLocal<T> {
  private final int threadLocalHashCode = nextHashCode();
  
  private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
  }
}
```

然后我们再来看下一个代码例子：

```
public class TianLuoThreadLocalTest {

    public static void main(String[] args) {
        Thread t = new Thread(new Runnable(){
            public void run(){
                ThreadLocal<TianLuoDTO> threadLocal1 = new ThreadLocal<>();
                threadLocal1.set(new TianLuoDTO("Hello,byLv"));
                System.out.println(threadLocal1.get());
                ThreadLocal<TianLuoDTO> threadLocal2 = new ThreadLocal<>();
                threadLocal2.set(new TianLuoDTO("Hello"));
                System.out.println(threadLocal2.get());
            }});
        t.start();
    }

}
//运行结果
TianLuoDTO{name='Hello,byLv'}
TianLuoDTO{name='Hello'}
```

再对比下这个图，可能就更清晰一点啦：

![640 (4)](https://bylv2727.oss-cn-shenzhen.aliyuncs.com/PicGo/blog/2022-08-12-ThreadLocal/640%20(4).png)

## **5. TreadLocal为什么会导致内存泄漏呢？**

### 5.1 弱引用导致的内存泄漏呢？

我们先来看看TreadLocal的引用示意图哈：

![640 (5)](https://bylv2727.oss-cn-shenzhen.aliyuncs.com/PicGo/blog/2022-08-12-ThreadLocal/640%20(5).png)

关于ThreadLocal内存泄漏，网上比较流行的说法是这样的：

> `ThreadLocalMap`使用`ThreadLocal`的**弱引用**作为`key`，当`ThreadLocal`变量被手动设置为`null`，即一个`ThreadLocal`没有外部强引用来引用它，当系统GC时，`ThreadLocal`一定会被回收。这样的话，`ThreadLocalMap`中就会出现`key`为`null`的`Entry`，就没有办法访问这些`key`为`null`的`Entry`的`value`，如果当前线程再迟迟不结束的话(比如线程池的核心线程)，这些`key`为`null`的`Entry`的`value`就会一直存在一条强引用链：Thread变量 -> Thread对象 -> ThreaLocalMap -> Entry -> value -> Object 永远无法回收，造成内存泄漏。

当ThreadLocal变量被手动设置为`null`后的引用链图：

![640 (6)](https://bylv2727.oss-cn-shenzhen.aliyuncs.com/PicGo/blog/2022-08-12-ThreadLocal/640%20(6).png)

实际上，`ThreadLocalMap`的设计中已经考虑到这种情况。所以也加上了一些防护措施：即在`ThreadLocal`的`get`,`set`,`remove`方法，都会清除线程`ThreadLocalMap`里所有`key`为`null`的`value`。

源代码中，是有体现的，如`ThreadLocalMap`的`set`方法：

```
  private void set(ThreadLocal<?> key, Object value) {

      Entry[] tab = table;
      int len = tab.length;
      int i = key.threadLocalHashCode & (len-1);

      for (Entry e = tab[i];
            e != null;
            e = tab[i = nextIndex(i, len)]) {
          ThreadLocal<?> k = e.get();

          if (k == key) {
              e.value = value;
              return;
          }

           //如果k等于null,则说明该索引位之前放的key(threadLocal对象)被回收了,这通常是因为外部将threadLocal变量置为null,
           //又因为entry对threadLocal持有的是弱引用,一轮GC过后,对象被回收。
            //这种情况下,既然用户代码都已经将threadLocal置为null,那么也就没打算再通过该对象作为key去取到之前放入threadLocalMap的value, 因此ThreadLocalMap中会直接替换调这种不新鲜的entry。
          if (k == null) {
              replaceStaleEntry(key, value, i);
              return;
          }
        }

        tab[i] = new Entry(key, value);
        int sz = ++size;
        //触发一次Log2(N)复杂度的扫描,目的是清除过期Entry  
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
          rehash();
    }
```

如`ThreadLocal`的`get`方法：

```
  public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        //去ThreadLocalMap获取Entry，方法里面有key==null的清除逻辑
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

private Entry getEntry(ThreadLocal<?> key) {
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];
        if (e != null && e.get() == key)
             return e;
        else
          //里面有key==null的清除逻辑
          return getEntryAfterMiss(key, i, e);
    }
        
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
        Entry[] tab = table;
        int len = tab.length;

        while (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == key)
                return e;
            // Entry的key为null,则表明没有外部引用,且被GC回收,是一个过期Entry
            if (k == null)
                expungeStaleEntry(i); //删除过期的Entry
            else
                i = nextIndex(i, len);
            e = tab[i];
        }
        return null;
    }
```

### 5.2 key是弱引用，GC回收会影响ThreadLocal的正常工作嘛？

`ThreadLocal`的`key`既然是**弱引用**，GC会不会贸然把`key`回收掉，进而影响`ThreadLocal`的正常使用呢？

> **弱引用**：具有弱引用的对象拥有更短暂的生命周期。如果一个对象只有弱引用存在了，则下次GC**将会回收掉该对象**（不管当前内存空间足够与否）

其实不会的，因为有`ThreadLocal`变量引用着它，是不会被GC回收的，除非手动把`ThreadLocal变量设置为null`，跑个demo来验证一下：

```
  public class WeakReferenceTest {
    public static void main(String[] args) {
        Object object = new Object();
        WeakReference<Object> testWeakReference = new WeakReference<>(object);
        System.out.println("GC回收之前，弱引用："+testWeakReference.get());
        //触发系统垃圾回收
        System.gc();
        System.out.println("GC回收之后，弱引用："+testWeakReference.get());
        //手动设置为object对象为null
        object=null;
        System.gc();
        System.out.println("对象object设置为null，GC回收之后，弱引用："+testWeakReference.get());
    }
}
运行结果：
GC回收之前，弱引用：java.lang.Object@7b23ec81
GC回收之后，弱引用：java.lang.Object@7b23ec81
对象object设置为null，GC回收之后，弱引用：null
```

### 5.3 ThreadLocal内存泄漏的demo

给大家来看下一个内存泄漏的例子，其实就是用线程池，一直往里面放对象

```
public class ThreadLocalTestDemo {

    private static ThreadLocal<TianLuoClass> tianLuoThreadLocal = new ThreadLocal<>();


    public static void main(String[] args) throws InterruptedException {

        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(5, 5, 1, TimeUnit.MINUTES, new LinkedBlockingQueue<>());

        for (int i = 0; i < 10; ++i) {
            threadPoolExecutor.execute(new Runnable() {
                @Override
                public void run() {
                    System.out.println("创建对象：");
                    TianLuoClass tianLuoClass = new TianLuoClass();
                    tianLuoThreadLocal.set(tianLuoClass);
                    tianLuoClass = null; //将对象设置为 null，表示此对象不在使用了
                   // tianLuoThreadLocal.remove();
                }
            });
            Thread.sleep(1000);
        }
    }

    static class TianLuoClass {
        // 100M
        private byte[] bytes = new byte[100 * 1024 * 1024];
    }
}


创建对象：
创建对象：
创建对象：
创建对象：
Exception in thread "pool-1-thread-4" java.lang.OutOfMemoryError: Java heap space
 at com.example.dto.ThreadLocalTestDemo$TianLuoClass.<init>(ThreadLocalTestDemo.java:33)
 at com.example.dto.ThreadLocalTestDemo$1.run(ThreadLocalTestDemo.java:21)
 at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
 at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
 at java.lang.Thread.run(Thread.java:748)
```

运行结果出现了OOM，`tianLuoThreadLocal.remove();`加上后，则不会`OOM`。

```
创建对象：
创建对象：
创建对象：
创建对象：
创建对象：
创建对象：
创建对象：
创建对象：
......
```

我们这里**没有手动设置**`tianLuoThreadLocal`变量为`null`，但是还是**会内存泄漏**。因为我们使用了线程池，线程池有很长的生命周期，因此线程池会一直持有`tianLuoClass`对象的`value`值，即使设置`tianLuoClass = null;`引用还是存在的。这就好像，你把一个个对象`object`放到一个`list`列表里，然后再单独把`object`设置为`null`的道理是一样的，列表的对象还是存在的。

```
    public static void main(String[] args) {
        List<Object> list = new ArrayList<>();
        Object object = new Object();
        list.add(object);
        object = null;
        System.out.println(list.size());
    }
    //运行结果
    1
```

所以内存泄漏就这样发生啦，最后内存是有限的，就抛出了`OOM`了。如果我们加上`threadLocal.remove();`，则不会内存泄漏。为什么呢？因为`threadLocal.remove();`会清除`Entry`，源码如下：

```
    private void remove(ThreadLocal<?> key) {
      Entry[] tab = table;
      int len = tab.length;
      int i = key.threadLocalHashCode & (len-1);
      for (Entry e = tab[i];
          e != null;
          e = tab[i = nextIndex(i, len)]) {
          if (e.get() == key) {
              //清除entry
              e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```

既然内存泄漏不一定是因为弱引用，那设计为弱引用的原因是什么呢？接下来探讨。

## **6. Entry的Key为什么要设计成弱引用呢？**

通过源码，我们是可以看到`Entry`的`Key`是设计为弱引用的(`ThreadLocalMap`使用`ThreadLocal`的弱引用作为`Key`的)。为什么要设计为弱引用呢？

![640 (7)](https://bylv2727.oss-cn-shenzhen.aliyuncs.com/PicGo/blog/2022-08-12-ThreadLocal/640%20(7).png)

我们先来回忆一下四种引用：

- **强引用**:我们平时`new`了一个对象就是强引用，例如` Object obj = new Object();`即使在内存不足的情况下，JVM宁愿抛出OutOfMemory错误也不会回收这种对象。
- **软引用**：如果一个对象只具有软引用，则内存空间足够，垃圾回收器就不会回收它；如果内存空间不足了，就会回收这些对象的内存。
- **弱引用**:具有弱引用的对象拥有更短暂的生命周期。如果一个对象只有弱引用存在了，则下次GC**将会回收掉该对象**（不管当前内存空间足够与否）。
- **虚引用**:如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收器回收。虚引用主要用来跟踪对象被垃圾回收器回收的活动。

我们先来看看官方文档，为什么要设计为弱引用：

```
To help deal with very large and long-lived usages, the hash table entries use WeakReferences for keys.
为了应对非常大和长时间的用途，哈希表使用弱引用的 key。
```

我再把ThreadLocal的引用示意图搬过来：

![640 (8)](https://bylv2727.oss-cn-shenzhen.aliyuncs.com/PicGo/blog/2022-08-12-ThreadLocal/640%20(8).png)

下面我们分情况讨论：

- 如果`Key`使用强引用：当`ThreadLocal`的对象被回收了，但是`ThreadLocalMap`还持有`ThreadLocal`的强引用的话，如果没有手动删除，`ThreadLocal`就不会被回收，会出现Entry的内存泄漏问题。
- 如果`Key`使用弱引用：当`ThreadLocal`的对象被回收了，因为`ThreadLocalMap`持有`ThreadLocal`的弱引用，即使没有手动删除，`ThreadLocal`也会被回收。`value`则在下一次`ThreadLocalMap`调用`set,get，remove`的时候会被清除。

因此可以发现，使用弱引用作为`Entry`的`Key`，可以多一层保障：弱引用`ThreadLocal`不会轻易内存泄漏，对应的`value`在下一次`ThreadLocalMap`调用`set,get,remove`的时候会被清除。

实际上，我们的内存泄漏的根本原因是，不再被使用的`Entry`，没有从线程的`ThreadLocalMap`中删除。一般删除不再使用的`Entry`有这两种方式：

- 一种就是，使用完`ThreadLocal`，手动调用`remove()`，把`Entry从ThreadLocalMap`中删除
- 另外一种方式就是：`ThreadLocalMap`的自动清除机制去清除过期`Entry`.（`ThreadLocalMap`的`get(),set()`时都会触发对过期`Entry`的清除）

## 7. InheritableThreadLocal 保证父子线程间的共享数据

我们知道`ThreadLocal`是线程隔离的，如果我们希望父子线程共享数据，如何做到呢？可以使用`InheritableThreadLocal`，demo 如下：

```java
public class InheritableThreadLocalTest {

   public static void main(String[] args) {
       ThreadLocal<String> threadLocal = new ThreadLocal<>();
       InheritableThreadLocal<String> inheritableThreadLocal = new InheritableThreadLocal<>();

       threadLocal.set("关注公众号：byLv");
       inheritableThreadLocal.set("关注公众号：程序员 byLv");

       Thread thread = new Thread(()->{
           System.out.println("ThreadLocal value " + threadLocal.get());
           System.out.println("InheritableThreadLocal value " + inheritableThreadLocal.get());
       });
       thread.start();
       
   }
}

//运行结果
ThreadLocal value null
InheritableThreadLocal value 关注公众号：程序员田螺
```

可以发现，在子线程中，是可以获取到父线程的 `InheritableThreadLocal `类型变量的值，但是不能获取到 `ThreadLocal `类型变量的值。

获取不到 `ThreadLocal` 类型的值，很好理解，因为它是线程隔离的嘛。`InheritableThreadLocal` 是如何做到的呢？原理是什么呢？

在`Thread`类中，除了成员变量`threadLocals`之外，还有另一个成员变量：`inheritableThreadLocals`。它们两类型是一样的：

```java
public class Thread implements Runnable {
   ThreadLocalMap threadLocals = null;
   ThreadLocalMap inheritableThreadLocals = null;
 }
```

`Thread` 类的 `init` 方法中，有一段初始化设置：

```java
 private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
      
        ......
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;

        /* Set thread ID */
        tid = nextThreadID();
    }
 static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
        return new ThreadLocalMap(parentMap);
    }
```

可以发现，当父线程的 `inheritableThreadLocals` 不为 null 时，就会将赋值给前线程的`inheritableThreadLocals`。

## **8. ThreadLocal的应用场景和使用注意点**

`ThreadLocal`的**很重要一个注意点**，就是使用完，要手动调用 remove()。

而`ThreadLocal`的应用场景主要有以下这几种：

- 使用日期工具类，当用到`SimpleDateFormat`，使用ThreadLocal保证线性安全

- 全局存储用户信息（用户信息存入`ThreadLocal`，那么当前线程在任何地方需要时，都可以使用）

- 保证同一个线程，获取的数据库连接`Connection`是同一个，使用`ThreadLocal`来解决线程安全的问题

- ```java
  public class LogContext {
      private static final ThreadLocal<StringBuilder> logThreadLocal = ThreadLocal.withInitial(StringBuilder::new);
  
      public static void addLog(String log) {
          logThreadLocal.get().append(log).append("\n");
      }
  
      public static String getLog() {
          return logThreadLocal.get().toString();
      }
  
      public static void clearLog() {
          logThreadLocal.remove();
      }
  }
  
  public class ExampleService {
      public void process() {
          try {
              LogContext.addLog("Start processing");
              // ...（业务逻辑代码）
              LogContext.addLog("Processing finished successfully");
          } catch (Exception e) {
              LogContext.addLog("Processing failed: " + e.getMessage());
          } finally {
              String log = LogContext.getLog();
              System.out.println("Thread log: " + log);
              LogContext.clearLog(); // 清除当前线程的日志
          }
      }
  }
  ```

  

### 参考资料

[1]彻底理解ThreadLocal：
https://www.cnblogs.com/xzwblog/p/7227509.html

[2]ThreadLocal是如何导致内存泄漏的：
https://zhuanlan.zhihu.com/p/346291694

[3]深入分析 ThreadLocal 内存泄漏问题：
https://www.jianshu.com/p/1342a879f523