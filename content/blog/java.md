+++
title = "Java并发总结（一）"
date = 2019-09-25
[taxonomies]
  tags = ["Java"]
+++

本文只是对Java并发知识的一个总结, Java并发是一个很难的问题，不是一篇博文能够说清，以后还需在实践中多多积累学习才是。

<!-- more -->

## 多线程的概念
 多线程程序在较低的层次上扩展了多任务的概念：一个程序同时执行多个任务。通常，
每一个任务称为一个线程（ thread), 它是线程控制的简称。可以同时运行一个以上线程的程
序称为多线程程序（multithreaded)。
那么，多进程与多线程有哪些区别呢？ 本质的区别在于每个进程拥有自己的一整套变
量， 而线程则共享数据。 这听起来似乎有些风险， 的确也是这样， 在本章稍后将可以看到这
个问题。然而，共享变量使线程之间的通信比进程之间的通信更有效、 更容易。 此外， 在有
些操作系统中，与进程相比较， 线程更“ 轻量级”， 创建、 撤销一个线程比启动新进程的开
销要小得多。

## 线程状态
Java线程包含如下状态：
- new(新创建)
- Runnable(可运行)
- Blocked(被阻塞)
- Waiting(等待)
- Timed waiting(计时等待)
- Terminated(被终止)
- [Java SE9 Enum Thread.State](https://docs.oracle.com/javase/9/docs/api/java/lang/Thread.State.html)

## 线程属性
Java线程包含如下属性：
- Priority(优先级)
- Daemon thread(守护线程)
- 未捕获异常处理
### 优先级
```java
public final static int MIN_PRIORITY = 1;

public final static int NORM_PRIORITY = 5;

public final static int MAX_PRIORITY = 10;

public final void setPriority(int newPriority) {
    ThreadGroup g;
    checkAccess();
    if (newPriority > MAX_PRIORITY || newPriority < MIN_PRIORITY) {
        throw new IllegalArgumentException();
    }
    if ((g = getThreadGroup()) != null) {
        if (newPriority > g.getMaxPriority()) {
            newPriority = g.getMaxPriority();
        }
        setPriority0(priority = newPriority);
    }
}
```
Java线程优先级默认值为5，取值区间为1-10的一个整数

### 守护线程
```java
public final void setDaemon(boolean on) {
    checkAccess();
    if (isAlive()) {
        throw new IllegalThreadStateException();
    }
    daemon = on;
}
```
```java
public static void main(String[] args) {
    Thread thread = new Thread(new MyRunnable());
    thread.setDaemon(true);
}
```
守护线程是程序运行时在后台提供服务的线程，不属于程序中不可或缺的部分。

当所有非守护线程结束时，程序也就终止，同时会杀死所有守护线程。

main() 属于非守护线程。

在线程启动之前使用 setDaemon() 方法可以将一个线程设置为守护线程。
## 线程创建方式
有三种使用线程的方式：
- 实现Runnable接口
- 实现Callable接口
- 继承Thread类
## 线程池
### Executor的类型
Executor 管理多个异步任务的执行，而无需程序员显式地管理线程的生命周期。这里的异步是指多个任务的执行互不干扰，不需要进行同步操作。
主要有三种Executor：
- CachedThreadPool：一个任务创建一个线程
- FixedThreadPool：所有任务只能使用固定大小的线程
- SingleThreadExecutor：相当于大小为 1 的 FixedThreadPool

### Executor的中断操作
调用 Executor 的 shutdown() 方法会等待线程都执行完毕之后再关闭，但是如果调用的是 shutdownNow() 方法，则相当于调用每个线程的 interrupt() 方法
```java
private static final int CORE_POOL_SIZE = 5;
private static final int MAX_POOL_SIZE = 10;
private static final int QUEUE_CAPACITY = 100;
private static final Long KEEP_ALIVE_TIME = 1L;

public static void main(String[] args) {

    //使用阿里巴巴推荐的创建线程池的方式
    //通过ThreadPoolExecutor构造函数自定义参数创建
    ThreadPoolExecutor executor = new ThreadPoolExecutor(
            CORE_POOL_SIZE,
            MAX_POOL_SIZE,
            KEEP_ALIVE_TIME,
            TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(QUEUE_CAPACITY),
            new ThreadPoolExecutor.CallerRunsPolicy());
}
```
如果只想中断 Executor 中的一个线程，可以通过使用 submit() 方法来提交一个线程，它会返回一个 Future<?> 对象，通过调用该对象的 cancel(true) 方法就可以中断线程
```java
Future<?> future = executorService.submit(() -> {
    // ..
});
future.cancel(true);
```
## 同步
> 线程之间是有共享内存的，为了防止运行时出现数据竞争，Java提供了一系列的工具来解决这个问题
### Synchronized
内置锁，当它用来修饰一个方法或者代码块的时候，能够保证在同一时刻最多只有一个线程执行该段代码，另一个线程必须等待当前线程执行完以后才能执行该段代码（未执行之前，该线程处于阻塞状态）。同时它也是一个可重入锁。

1. 同步一个类
```java
public static Singleton getInstance() {
    if (singleton == null) {
        synchronized (Singleton.class) {
            if (singleton == null) {
                singleton = new Singleton();
            }
        }
    }
    return singleton;
}
```
上述代码作用于整个类，也就是说两个线程调用同一个类的不同对象上的这种同步语句，也会进行同步。(如果锁住this,那么就只有同一个对象的同步语句才会进行同步)

2. 同步一个方法
```java
public synchronized void func () {
    // ...
}
```
作用于同一个对象。

### ReentrantLock
ReentrantLock 是 java.util.concurrent（J.U.C）包中的锁
```java
public class LockExample {

    private Lock lock = new ReentrantLock();

    public void func() {
        lock.lock();
        try {
            for (int i = 0; i < 10; i++) {
                System.out.print(i + " ");
            }
        } finally {
            // 确保释放锁，从而避免发生死锁
            lock.unlock();
        }
    }
}
```
### 比较
1. 锁的实现
synchronized 是 JVM 实现的，而 ReentrantLock 是 JDK 实现的。
2. 性能
新版本 Java 对 synchronized 进行了很多优化，例如自旋锁等，synchronized 与 ReentrantLock 大致相同。
3. 等待可中断
当持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待，改为处理其他事情。ReentrantLock 可中断，而 synchronized 不行。
4. 公平锁
公平锁是指多个线程在等待同一个锁时，必须按照申请锁的时间顺序来依次获得锁。
公平锁将由于在挂起线程和恢复线程时存在的开销而极大的降低效率。
而非公平锁由于是在请求时锁已经为可用状态就直接获取，不需要进行什么额外的操作，因此效率更高。
synchronized 中的锁是非公平的，ReentrantLock 默认情况下也是非公平的，但是也可以是公平的。
5. 锁绑定多个条件
一个 ReentrantLock 可以同时绑定多个 Condition 对象。

## 线程之间的协作
当多个线程可以一起工作去解决某个问题时，如果某些部分必须在其它部分之前完成，那么就需要对线程进行协调。

### join()
在线程中调用另一个线程的 join() 方法，会将当前线程挂起，而不是忙等待，直到目标线程结束。

### wait()  / notify() /  notifyAll()
- 调用 wait() 使得线程等待某个条件满足，线程在等待时会被挂起，当其他线程的运行使得这个条件满足时，其它线程会调用 notify() 或者 notifyAll() 来唤醒挂起的线程。
- 它们都属于 Object 的一部分，而不属于 Thread。
- 只能用在同步方法或者同步控制块中使用，否则会在运行时抛出 IllegalMonitorStateException。
- 使用 wait() 挂起期间，线程会释放锁。这是因为，如果没有释放锁，那么其它线程就无法进入对象的同步方法或者同步控制块中，那么就无法执行 notify() 或者 notifyAll() 来唤醒挂起的线程，造成死锁。

### wait() 和 sleep() 的区别
- wait() 是 Object 的方法，而 sleep() 是 Thread 的静态方法。
- wait() 会释放锁，sleep() 不会。

### await() / signal() / signalAll()
- java.util.concurrent 类库中提供了 Condition 类来实现线程之间的协调，可以在 Condition 上调用 await() 方法使线程等待，其它线程调用 signal() 或 signalAll() 方法唤醒等待的线程。
- 相比于 wait() 这种等待方式，await() 可以指定等待的条件，因此更加灵活。
- 使用 Lock 来获取一个 Condition 对象。

## 小结
这次总结了线程的基本概念，以及锁的运用，下次介绍下Java并发编程的工具箱😄

## 参考
- 《Java并发编程实战》
- 《Java核心技术（卷一）》第十四章
- [CS-NOTES](https://github.com/CyC2018/CS-Notes/blob/master/notes/Java%20%E5%B9%B6%E5%8F%91.md)
