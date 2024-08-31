---
title: "Different ways to do async in Rust"
date: 2024-10-28T00:00:00+08:00
description: "A list of different ways and knowing when to use what"
tags: ["rust"]
type: post
weight: 25
showTableOfContents: true
---

What are the different ways of doing concurrent programming and what's available for the developers to do that 

Async/Await 

run a large number of concurrent tasks on a small number of OS threads, while preserving much of the look and feel of ordinary synchronous programming, through the async/await syntax.

Other techniques 

OS threads don't require any changes to the programming model, which makes it very easy to express concurrency. However, synchronizing between threads can be difficult, and the performance overhead is large. Thread pools can mitigate some of these costs, but not enough to support massive IO-bound workloads.

Give an example in C or C++ to do that 

Event-driven programming, in conjunction with callbacks, can be very performant, but tends to result in a verbose, "non-linear" control flow. Data flow and error propagation is often hard to follow.

Give an example in javascript 

Coroutines, like threads, don't require changes to the programming model, which makes them easy to use. Like async, they can also support a large number of tasks. However, they abstract away low-level details that are important for systems programming and custom runtime implementors.

Give an example in GO 