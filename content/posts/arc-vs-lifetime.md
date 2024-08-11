---
title: "Lifetime vs Arc in Rust"
date: 2024-08-11T17:55:28+08:00
description: "Understanding the difference and knowing when to use what"
tags: ["rust"]
type: post
weight: 25
showTableOfContents: true
---

# Understanding Arc vs Lifetimes in Rust

In Rust, both `Arc` and lifetimes are tools used to manage how data is shared or accessed across different parts of a program, but they serve different purposes and are used in different contexts.

## 1. Lifetimes

Lifetimes in Rust are a way to ensure that references are always valid. They help the Rust compiler understand how long references to data should be valid to prevent dangling references, which could lead to undefined behavior.

### When to Use Lifetimes:
- **Borrowing:** When you have references (`&T` or `&mut T`) to some data, and you want to ensure that these references are valid for as long as they're needed.
- **Function Arguments and Returns:** When passing references into and out of functions, you often need to specify lifetimes to ensure the borrowed data is valid as expected.
- **Avoiding Cloning:** Lifetimes allow you to share data without needing to clone it, which can be more efficient.
- **Compile-Time Construct:** Lifetimes are a compile-time construct that enforces memory safety. They allow the Rust compiler to catch potential issues related to reference validity before the code even runs, eliminating entire classes of runtime errors related to memory management.

### Example:
```rust
struct Data<'a> {
    name: &'a str,
}

fn print_name(data: &Data) {
    println!("Name: {}", data.name);
}

fn main() {
    let s = String::from("Hello, world!");
    let data = Data { name: &s };

    print_name(&data); // This works because `s` lives long enough.
}
```

In this example, the lifetime `'a` ensures that the reference `&s` used in Data is valid for as long as Data exists. The lifetime ensures that `s` doesn't go out of scope while Data still references it.

## 2. Arc (Atomic Reference Counting)

`Arc` (short for Atomic Reference Counter) is a smart pointer that enables safe, concurrent, shared ownership of data. Unlike `Rc`, which is not thread-safe, `Arc` can be safely shared across multiple threads.

### When to Use `Arc`:
- **Multi-threading:** When you need to share data between multiple threads and want to ensure that the data is kept alive as long as there are references to it.
- **Shared Ownership:** When multiple parts of your program need ownership of the same data, and you don't want to clone it, but rather share it.

### Example:

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(String::from("Hello, Arc!"));

    let mut handles = vec![];

    for _ in 0..5 {
        let data_clone = Arc::clone(&data);
        let handle = thread::spawn(move || {
            println!("Thread says: {}", data_clone);
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

In this example, `Arc` allows the `String` data to be safely shared across multiple threads. Each thread gets its own `Arc` reference to the data, and the data is only deallocated once the last `Arc` is dropped.

## When to Use Lifetimes vs Arc

### Use `Lifetimes` when:
- You have a single-threaded context.
- You want to share references without ownership transfer.
- You aim to avoid unnecessary allocations or cloning.
- You want to enforce strict borrowing rules that ensure data validity without runtime overhead.

### Use `Arc` when:
- You need to share data across multiple threads.
- You need shared ownership, where multiple parts of the code require ownership of the same data.
- You can't use lifetimes because you need the data to live beyond a particular scope or across threads.

## Comparison and Scenarios

### Single-threaded Sharing:
- **Lifetimes:** Use when you can manage the data's scope and lifetime manually, and you don't need shared ownership.
- **Arc:** Not needed in single-threaded contexts unless you specifically need shared ownership with runtime reference counting.

### Multi-threaded Sharing:
- **Lifetimes:** Not applicable, as lifetimes can't ensure safety across threads.
- **Arc:** Use when you need to share data between threads safely.

### Performance Considerations:
- **Lifetimes:** No runtime overhead since it's purely compile-time checked.
- **Arc:** Introduces some overhead due to atomic reference counting, but is necessary for thread-safe shared ownership.

##  Real-World Example: Managing Global State with Lifetimes and `Arc` in Rust

In a real-world Rust application, you might have a global state that needs to be accessed and shared across multiple services. These services may have their own state and might need to share some of this state with each other while also accessing the global state. This scenario can be managed using both lifetimes and `Arc`.

## Scenario Overview

1. **Global State:** A configuration or state shared across the entire application.
2. **Service 1:** A service that has its own state and needs to access the global state.
3. **Service 2:** A service that has its own state and needs to access the global state.

### Struct Definitions

- **GlobalState:** The global configuration/state.
- **Service1State:** The state specific to Service 1.
- **Service2State:** The state specific to Service 2.

### Managing Global State with Lifetimes

Using lifetimes, you can ensure that the references to the global state remain valid across different services.

#### Example:

```rust
struct GlobalState {
    config: String,
}

struct Service1State<'a> {
    global: &'a GlobalState,
    service_data: String,
}

struct Service2State<'a> {
    global: &'a GlobalState,
    service_data: String,
}

impl<'a> Service1State<'a> {
    fn new(global: &'a GlobalState, data: String) -> Self {
        Self {
            global,
            service_data: data,
        }
    }

    fn access_global(&self) {
        println!("Service 1 accessing global config: {}", self.global.config);
    }
}

impl<'a> Service2State<'a> {
    fn new(global: &'a GlobalState, data: String) -> Self {
        Self {
            global,
            service_data: data,
        }
    }

    fn access_global(&self) {
        println!("Service 2 accessing global config: {}", self.global.config);
    }
}

fn main() {
    let global_state = GlobalState {
        config: String::from("Global Configuration"),
    };

    let service1 = Service1State::new(&global_state, String::from("Service 1 Data"));
    let service2 = Service2State::new(&global_state, String::from("Service 2 Data"));

    service1.access_global();
    service2.access_global();
}
```

In this example, `Service1State` and `Service2State` both hold references to the `GlobalState`. The lifetimes `'a` ensure that these references remain valid as long as `GlobalState` exists.

### Sharing State Between Services with `Arc`

When you need to share state between services and across threads, using `Arc` is a better approach. `Arc` allows multiple ownership and safe sharing of data between threads.

#### Example:

```rust
use std::sync::Arc;
use std::thread;

struct GlobalState {
    config: String,
}

struct Service1State {
    global: Arc<GlobalState>,
    service_data: String,
}

struct Service2State {
    global: Arc<GlobalState>,
    service_data: String,
}

impl Service1State {
    fn new(global: Arc<GlobalState>, data: String) -> Self {
        Self {
            global,
            service_data: data,
        }
    }

    fn access_global(&self) {
        println!("Service 1 accessing global config: {}", self.global.config);
    }
}

impl Service2State {
    fn new(global: Arc<GlobalState>, data: String) -> Self {
        Self {
            global,
            service_data: data,
        }
    }

    fn access_global(&self) {
        println!("Service 2 accessing global config: {}", self.global.config);
    }
}

fn main() {
    let global_state = Arc::new(GlobalState {
        config: String::from("Global Configuration"),
    });

    let service1 = Service1State::new(Arc::clone(&global_state), String::from("Service 1 Data"));
    let service2 = Service2State::new(Arc::clone(&global_state), String::from("Service 2 Data"));

    let service1_thread = thread::spawn(move || {
        service1.access_global();
    });

    let service2_thread = thread::spawn(move || {
        service2.access_global();
    });

    service1_thread.join().unwrap();
    service2_thread.join().unwrap();
}
```

### Why not just use Arc all the time?

Given the above 2 examples, you might be wondering why not use `Arc` or `Rc` both the times. Sure, this is possible but keep in mind of the overhead due to atomic operations that manage reference counting, which is only needed for thread-safe, shared ownership across multiple threads. In single-threaded contexts or when shared ownership isn't required, simple references with lifetimes (&T) are more efficient and straightforward. Lifetimes provide compile-time safety without any runtime cost, ensuring that references are valid without the need for complex ownership semantics. Therefore, while Arc is essential for multi-threaded scenarios, in many cases, using lifetimes or other simpler constructs is more performant and appropriate.

## C++ Tangent

I like C++ (17 and above only) and often make comparisons between C++ features and Rust to understand systems better. While Rust's `Arc` and lifetimes have parallels in C++, there's no direct equivalent to Rust's lifetimes, and this is partly due to how C++ handles memory management.

### `Arc` in Rust vs. `std::shared_ptr` in C++

The closest equivalent in C++ is `std::shared_ptr`, which also allows multiple ownership of dynamically allocated memory. Both `Arc` and `std::shared_ptr` use reference counting to manage the lifetime of the object they own, deallocating it only when the last reference is dropped.

### Example in C++:
```cpp
#include <iostream>
#include <memory>
#include <thread>
#include <vector>

void print_shared(std::shared_ptr<std::string> data) {
    std::cout << "Thread says: " << *data << std::endl;
}

int main() {
    auto data = std::make_shared<std::string>("Hello, std::shared_ptr!");

    std::vector<std::thread> threads;
    for (int i = 0; i < 5; ++i) {
        threads.emplace_back(print_shared, data);
    }

    for (auto& t : threads) {
        t.join();
    }

    return 0;
}
```

### Lifetimes in Rust vs C++

Lifetimes in Rust ensures references are always valid, preventing dangling pointers and other memory safety issues at **compile time**. However, there is no direct equivalent to Rust's lifetimes in C++. C++ relies on manual memory management or smart pointers like `std::unique_ptr` and `std::shared_ptr` to manage lifetimes, but these do not provide the same compile-time guarantees as Rust.

I've spent a considerable amount of time debugging invalid references in C/C++, which I no longer need to worry about in Rust

## Summary

Lifetimes are about ensuring references are valid and avoiding ownership issues within a single thread or scope. `Arc` is about safely sharing data with shared ownership across threads, ensuring that the data lives as long as it's needed, even in a multi-threaded context.

Oh and also learn Rust.

