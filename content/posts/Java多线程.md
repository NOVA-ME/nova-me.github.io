+++
title="Java多线程"
date=2018-08-20

[taxonomies]
categories = ["Tech"]
tags = ["Java", "多线程"]

+++

**“空间换时间”** 是多线程编程的核心哲学之一：我们通过消耗内存（创建线程栈、维护线程池状态、上下文切换的开销）来换取任务处理的并发度和响应速度（时间）。在工作中时常会用到多线程，下面介绍下常用的Java多线程工具。

<!-- more -->

### 一、 线程管理：池化思想的极致体现

**核心理念：** 避免频繁创建和销毁线程带来的系统开销（时间），通过维护一定数量的常驻线程（空间）来随时待命。

#### ThreadPoolExecutor（线程池核心类）

```java
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                10,
                100,
                1,
                TimeUnit.MINUTES,
                new LinkedBlockingQueue<>(20000),
                Executors.defaultThreadFactory(), (runnable, executor) -> {
                // 当最大线程数+队列大小满负荷还有线程加入后的拒绝策略 默认ignored
        });
```

```java
public class UniverseThreadFactory implements ThreadFactory {

    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;

    UniverseThreadFactory() {
        group = Thread.currentThread().getThreadGroup();
        // 可以自定义线程名称
        namePrefix = "xxx" +
                poolNumber.getAndIncrement() +
                "-thread-";
    }

    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,
                namePrefix + threadNumber.getAndIncrement(),
                0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
```

### 二、 并发流程控制：多线程的“红绿灯”

**核心理念：** 协调不同线程之间的执行顺序，确保逻辑正确性。

#### CountDownLatch

初始化 new CountDownLatch(N)。子线程执行完调用 countDown()，主线程调用 await() 阻塞等待数值归零。

```Java
	// 初始化
	CountDownLatch countDownLatch = new CountDownLatch(20);
        ArrayList<Integer> numList = Lists.newArrayListWithCapacity(20);
        for (int i = 0; i < 20; i++) {
            numList.add(i + 1);
        }
        for (Integer i : numList) {
            threadPoolExecutor.execute(() -> {
                try {
                    System.out.println(i);
                } catch (Exception e) {
                    // 添加日志
                } finally {
                    // 子线程执行完调用 在finally块里面保证完成后调用
                    countDownLatch.countDown();
                }
            });
        }
        try {
            System.out.println("到达等待地点");
            // 阻塞等待数值归零
            countDownLatch.await();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        threadPoolExecutor.shutdown();
```

#### CyclicBarrier

初始化new CyclicBarrier(N,()->{})。线程调用 await() 表示自己到达。可循环使用，适合分阶段的并行计算

```java
	int parties = 3;
        List<Integer> numList = Lists.newArrayList();
        for (int i = 0; i < parties; i++) {
            numList.add(i + 1);
        }
        List<Integer> calcList = Lists.newArrayList();
        CyclicBarrier cyclicBarrier = new CyclicBarrier(parties, () -> {
            // 到达集合地点后最先执行的方法
            System.out.println("集合成功，执行方法成功" + calcList.toString());
        });
        for (Integer i : numList) {
            threadPoolExecutor.execute(() -> {
                try {
                    Thread.sleep((long) (Math.random() * 5000));
                    System.out.println("线程：" + Thread.currentThread().getName() + "到达集合点1，已经有" + (cyclicBarrier.getNumberWaiting() + 1) + "个到达");
                    calcList.add(cyclicBarrier.getNumberWaiting());
                    cyclicBarrier.await();
                    Thread.sleep((long) (Math.random() * 5000));
                    System.out.println("线程：" + Thread.currentThread().getName() + "到达集合点2，已经有" + (cyclicBarrier.getNumberWaiting() + 1) + "个到达");
                    calcList.add(cyclicBarrier.getNumberWaiting());
                    cyclicBarrier.await();
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }
            });
        }
```

