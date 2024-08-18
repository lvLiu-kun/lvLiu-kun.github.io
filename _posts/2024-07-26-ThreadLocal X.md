# [ThreadLocal](file:///P:/Java/Java/MyNotes/code/threadlocal/src/com/byLv/threadlocal/ThreadlocalSource.java)

ThreadLocal 底层有点类似于 Hashmap，如果用 Hashmap 对线程关联数据，则 key 为线程，value 为数据，也能达到线程隔离的效果。

但是为什么不用 HashMap？如果线程要关联多个数据，key 为线程，无法区分是哪个数据。

参考文档：
https://segmentfault.com/a/1190000041264583
参考视频：
王振国：
https://www.bilibili.com/video/BV1Y7411K7zz?p=301&vd_source=146731dbc824138172a64a7faf714aab
free-coder：
https://www.bilibili.com/video/BV1SJ41157oF?spm_id_from=333.337.search-card.all.click&vd_source=146731dbc824138172a64a7faf714aab
mic：
https://www.bilibili.com/video/BV1GP41157xr?spm_id_from=333.337.search-card.all.click&vd_source=146731dbc824138172a64a7faf714aab

## 实现原理

当我们用 Threadlocal 存值时，值会存在自己私有的线程中，首先会获取当前线程的成员变量 threadLocals（类型为ThreadLocal.ThreadLocalMap），里面有一个 Entry 数组，数据存在 Entry 对象里，key 为 threadlocal，value 为 存的值。

## 作用

实现线程之间的数据隔离（核心应用），每个线程拥有的都是独立的数据；
实现当前线程的全局变量共享，可以把数据存到 ThreadLocal 里面，进行上下游传递，避免了传参。
例子：
如何使用 ThreadLocal 来达到多线程的数据隔离呢？
在 Spring Web项目中，通常会把业务分成 Controller、Service、Dao 等等，也知道注解 @Autowired 默认使用单例模式。那有没有想过，当不同的请求线程进来后，因为Dao 层使用的是单例，即负责连接数据库的 Connection 也只有一个，如果这些线程都去连接数据库的话，会造成线程不安全的问题，Spring是怎样来解决的呢？
在 Dao 层里装配的 Connection 肯定是线程安全的，解决方案就是使用 ThreadLocal 方法。当每一个请求线程使用 Connection 的时候，都会从 ThreadLocal 中获取，如果值为 null，那就说明没有对数据库进行连接，连接后就会存入到 ThreadLocal 里，这样一来，每一个线程都有一个自己的 Connection，即每一个线程维护的是自己的数据，从而达到线程的隔离效果。
如何保存线程的全局变量？
原始 jdbc 在每个 dao 方法中都是先获取连接然后关闭连接，如果要在 service 中保证多个 dao 要么同时成功，要么同时失败，如生成订单后又要更新库存，就得保证它们存在同一个事务中，即确保使用的 connection 是同一个，就可以使用 ThreadLocal 来存，取的时候就从 ThreadLocal 里面取。
参考代码：
file:///N:/Java/JavaWeb/%E5%B0%9A%E7%A1%85%E8%B0%B7%E7%BB%8F%E5%85%B8%E7%89%88/%E8%B5%84%E6%96%99/15-Filter%E8%BF%87%E6%BB%A4%E5%99%A8/%E7%AC%94%E8%AE%B0/15_%E5%B0%9A%E7%A1%85%E8%B0%B7_Filter%E8%BF%87%E6%BB%A4%E5%99%A8_%E7%8E%8B%E6%8C%AF%E5%9B%BD%20-%20%E8%AF%BE%E5%A0%82%E7%AC%94%E8%AE%B0.pdf#page=9&zoom=auto,-202,299


## 定义 ThreadLocal 一般是用什么修饰？

static 修饰符。


## 用法

使用 set 方法，自动和线程绑定所设置的值；
使用 get 方法，获取和当前线程绑定的值（ThreadLocal 对象充当 key，然后通过 key 的 threadLocalHashCode 进行搜索）。

## 缺点

一个 Threadlocal 变量只能关联一个数据，要关联多个数据，就得多声明几个 Threadlocal 变量。

## 数据什么时候释放？

线程销毁后，自动释放。


## 为什么会存在内存泄漏？

一般 threadLocalMap 对象随着线程销毁而销毁，但是如果用的是线程池，那么就存在内存泄漏的问题。
为什么会存在内存泄漏问题？使用 Threadlocal 时，有两个引用指向 threadlocal 对象，一个是栈上的 threadlocal 对它的引用，一个是 threadLocalMap 中的 Key 对它的引用，当栈上的 threadlocal 不再引用了，因为 key 是弱引用，那 threadlocal 在下次 GC 的时候就会被回收，但是 value 是强引用，会随着线程一直存在，造成内存泄漏，且如果复用这个线程，使用的是上次任务的数据。所以在用完 threadlocal 后，手动调用 remove 方法清除下。

## key 为什么是弱引用而不是强引用？

如果是强引用，当我们栈中不再引用 threadlocal 了，但 threadLocalMap 中的 Key 还在引用，如果使用的是线程池，threadlocal 永远不会被回收，久而久之可能会导致 OOM。

## InheritableThreadLocal

继承 ThreadLocal，可以实现父子线程间的数据共享，当创建一个子线程时，如果父线程的 inheritableThreadLocals 不为 null，就会把它赋值给子线程的 inheritableThreadLocals。

有什么潜在问题？如果这个值被修改了，其他线程也会受到影响，所以可能会引发并发问题导致数据不一致。

