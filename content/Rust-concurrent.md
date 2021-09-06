+++
title = "Rust无畏并发"
date = 2021-07-25

[taxonomies]
tags = ["Rust"]
categories = ["Programming Language"]
+++

Rust无畏并发
<!-- more -->
## 标准库

```rust
use std::thread;
use std::time::Duration;

fn main(){
    thread::spawn(||{
        for i in 1..10{
            println!("hi number {} from the spaw thread", i);
            thread::sleep(Duration::from_millis(5));
        }
    });

    for i in 1..10 {
        println!("hi number {} from the main thread", i);
        thread::sleep(Duration::from_millis(10));
    }
}
```
