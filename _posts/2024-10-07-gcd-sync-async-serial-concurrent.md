---
layout: post
title: 'GCD: Understand sync, async and serial, concurrent'
categories: [iOS dev]
tag: [swift, gcd, thread, queue, serial, concurrent]
---

Normally we use `DispatchQueue()` API provided by Apple to handle threading features in our daily works. But few will know the mechanism underneath the hood and jargons like `sync`, `async`, `serial` or `concurrent` means. All we need to know it's when we have to do something related to UI we should switch back to main queue, right? Otherwise, the app will crash to inform you make a unforgiven mistake.

In this article, I am going to introduce the mechanism of GCD, what is `Thread` and `Queue`, and their features. So you can have a clear picture of how they work together and what potential issues you might need to avoid.


## What is GCD?
Grand Central Dispatch (GCD) is a concurrency framework in. It manages a shared thread pool, allocating threads to tasks based on system load and available resources. GCD uses dispatch queues to organize work. We developers add tasks to these queues rather than managing threads directly.

## Thread
A thread is the smallest unit of execution within a process. It represents a single sequence of instructions that can be executed independently by the CPU.
Main Thread and Background Thread

Main Thread: The primary thread where UI operations and event handling occur. It's crucial for maintaining a responsive user interface.
Background Thread: Any thread other than the main thread. Used for time-consuming tasks to prevent blocking the main thread.

## Sync and Async
These are thread's features, describing the way they execute code.

Sync (Synchronous): Executes code sequentially, blocking the current thread until the task completes.
Async (Asynchronous): Initiates a task and continues execution without waiting for the task to complete, allowing other code to run in the meantime.

// serial and concurrent for queue
// combinations, sync + concurrent

