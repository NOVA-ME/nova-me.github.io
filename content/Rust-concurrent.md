+++
title = "Rust无畏并发"
date = 2021-07-25

[taxonomies]
tags = ["Rust"]
categories = ["Programming Language"]
+++

并发常见问题如下：

- 竞争状态（Race conditions），多个线程以不一致的顺序访问数据或资源
- 死锁（Deadlocks），两个线程互相等待对方停止使用其所拥有的资源，这会阻止它们继续运行
- 只会发生在特定情况且难以稳定重现和修复的 bug

接下来看看 Rust 中是如何处理并发问题的

<!-- more -->

## 标准库中的并发

标准库的```thread::spawn```会创建一个线程去执行传入的闭包函数，如下所示，join放在注释处的话，主线程会等待直到新线程执行完毕才开始执行

```rust
use std::{thread, time::Duration};

fn main() {
    let handler = thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {}, from the spawn thread!", i);
            thread::sleep(Duration::from_secs(1));
        }
    });
    // handler.join().unwrap();
    for i in 1..10 {
        println!("hi number {}, from main thread!", i);
        thread::sleep(Duration::from_secs(1));
    }
    handler.join().unwrap();
}
```

## move移动所有权到闭包

如下所是```move```会移动```vec```的所有权到闭包中去

```rust
use std::{thread, time::Duration, vec};

fn main() {
    let v = vec![1, 2, 3, 4, 5];

    let handler = thread::spawn(move || {
        for i in v.iter() {
            println!("hi number {}, from the spawn thread!", i);
            thread::sleep(Duration::from_secs(1));
        }
    });

    for i in 1..=5 {
        println!("hi number {}, from main thread!", i);
        thread::sleep(Duration::from_secs(1));
    }

    handler.join().unwrap();
}
```

## tokio的异步多线程

```rust
use tokio::{sync::mpsc, time::Instant};

#[tokio::main]
async fn main() {
    let now = Instant::now();

    let (tx, mut rx) = mpsc::channel(1000000);

    for i in 1..=1000000 {
        let tx1 = tx.clone();
        tokio::spawn(async move {
            tx1.send(format!("{}-{}", "hi", i)).await;
        });
    }

    tokio::spawn(async move {
        tx.send(String::from("final string!")).await;
    });

    while let Some(v) = rx.recv().await {
        println!("{}", v);
    }

    println!("use {} seconds", now.elapsed().as_secs());
}
```