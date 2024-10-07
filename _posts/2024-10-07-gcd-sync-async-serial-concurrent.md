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

## Queue
A queue is an abstract data type that manages the execution of tasks. In iOS, queues are typically managed by Grand Central Dispatch (GCD).

The main queue is a serial queue that runs on the main thread. There is only one main queue in an iOS app. All UI updates must occur on the main queue to ensure thread safety and prevent UI-related issues.

You must be familiar with using this API to access the main queue.
```swift
DispatchQueue.main.async {
    // Code to run on the main queue
}
```

Unlike main queue, there can be many queues running on background threads. You can either use global queues provided by the system, or create your own custom ones.

## Serial and Concurrent
These two are queue's features.
Serial Queue: Executes one task at a time in the order they were added.
Concurrent Queue: Can execute multiple tasks simultaneously.

The system global queues run concurrently. As to custom queue, you can specify whether it is serial or concurrent by setting up the attribute.

```swift
// Serial Queue
let serialQueue = DispatchQueue(label: "com.example.serialQueue")

// Concurrent Queue
let concurrentQueue = DispatchQueue(label: "com.example.concurrentQueue", attributes: .concurrent)
```

## The Relationship Between Threads and Queues
Threads are actual paths of execution managed by the operating system. Queues are abstractions used to organize and manage the execution of tasks.
iOS uses a dynamic thread pool to execute tasks from queues. The system, not the developer, manages the creation and destruction of threads.

It works like this:

1. Queue Submission
When you submit a task to a queue (background or otherwise), you're not directly assigning it to a specific thread.


2. Thread Assignment
The system dynamically assigns tasks from queues to available threads in the thread pool. A single thread might execute tasks from different queues at different times.


3. Concurrent Execution
When using concurrent queues, multiple tasks can be executed simultaneously on different threads. But remember, a single thread executes only one task at a time, regardless of which queue it came from.

## Combinations of Sync/Async and Serial/Concurrent Queues
1. Sync on Serial Queue
Behavior: Tasks are executed one at a time in the order they were added.
The calling thread is blocked until the task completes.
Simple and predictable, but can lead to deadlocks if not used carefully.

```swift
let serialQueue = DispatchQueue(label: "com.example.serialQueue")
serialQueue.sync {
    print("Task executed synchronously on serial queue")
}
print("This will print after the task is completed")
```

2. Async on Serial Queue
Behavior: Tasks are executed one at a time in the order they were added.
The calling thread continues execution immediately without waiting for the task to complete.
Useful for offloading tasks that need to be executed in a specific order without blocking the current thread.

```swift
let serialQueue = DispatchQueue(label: "com.example.serialQueue")
serialQueue.async {
    print("Task executed asynchronously on serial queue")
}
print("This may print before or after the task is completed")
```

3. Sync on Concurrent Queue
Behavior: Multiple tasks can be executed simultaneously on different threads.
The calling thread is blocked until the specific task it submitted completes.
Other tasks on the concurrent queue may continue to execute in parallel.

```swift

```