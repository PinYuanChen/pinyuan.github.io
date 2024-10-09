---
layout: post
title: 'GCD: Understanding Sync/Async and Serial/Concurrent Queues'
categories: [iOS dev]
tag: [swift, gcd, thread, sync, async, queue, serial, concurrent]
---

As iOS developers, we frequently use Apple's DispatchQueue API to manage threading in our applications. However, many of us may not fully grasp the underlying mechanisms or the meaning of terms like "sync," "async," "serial," or "concurrent." Often, we simply know that UI-related tasks should be performed on the main queue to avoid crashes. But there's much more to understand about GCD.

In this article, we'll delve into the inner workings of GCD, explore the concepts of threads and queues, and examine their key characteristics. By the end, you'll have a clearer understanding of how these components work together.

## What is GCD?
Grand Central Dispatch (GCD) is a powerful concurrency framework developed by Apple for iOS and macOS. It simplifies the process of writing multi-threaded code by abstracting away the complexities of thread management.

GCD efficiently manages a shared thread pool, dynamically allocating threads to tasks based on system load and available resources. This approach optimizes performance and resource utilization.

At the heart of GCD are dispatch queues. These queues manage the tasks submitted to them. As developers, we add tasks to these queues, and GCD handles the underlying thread management and task execution.

## Thread
A thread is the smallest unit of execution within a process. It represents a single sequence of instructions that can be executed independently by the CPU. Each thread has its own stack but shares the same heap memory with other threads in the same process. iOS devices have a limited number of cores, so the system manages thread execution efficiently.

### Types of Threads
**Main Thread:** The primary thread where UI operations and event handling occur. It's crucial for maintaining a responsive user interface.

**Background Thread:** Any thread other than the main thread. Used for time-consuming tasks to prevent blocking the main thread.

### Sync/Async
These terms describe how code is executed in relation to the current flow of the program.

**Sync (Synchronous):** Synchronous execution operates in a sequential manner, where each task blocks the current thread until it completes. In this mode, the program diligently waits for each operation to finish before proceeding to the next line of code, ensuring a straightforward and predictable flow of execution. While this approach is simple to reason about, it comes with significant drawbacks: long-running operations performed on the main thread can lead to an unresponsive user interface, degrading the user experience.

**Async (Asynchronous):** Asynchronous execution initiates a task and immediately continues with the next line of code, without waiting for the task to complete. This approach allows other code to run concurrently, significantly improving the responsiveness of the application, especially for time-consuming operations. However, it introduces complexity in managing task completion and error handling, requiring careful implementation of completion handlers.

## Queue
A queue is an abstract data type that manages the execution of tasks. In iOS, queues are typically managed by Grand Central Dispatch (GCD).

The main queue is a serial queue that runs on the main thread. There is only one main queue in an iOS app. All UI updates must occur on the main queue to ensure thread safety and prevent UI-related issues.

Accessing the main queue is a common operation in iOS development. Here's the typical syntax:

```swift
DispatchQueue.main.async {
    // Code to run on the main queue
    updateUI()
}
```

Unlike the main queue, there can be many queues running on background threads. These are used for non-UI tasks to keep the main queue free for UI updates. There are two types of background queues:

1. Global Queues: Provided by the system, shared across your app.
2. Custom Queues: Created by developers for specific purposes.

### Global Queues
Global queues are concurrent queues provided by the system. They are categorized by [Quality of Service (QoS)](https://developer.apple.com/documentation/dispatch/dispatchqos/qosclass) levels, which determine their priority:

```swift
let globalQueue = DispatchQueue.global(qos: .userInitiated)
globalQueue.async {
    // Perform background task
}
```

### Custom Queues
You can create your own queues for specific tasks:

```swift
let customQueue = DispatchQueue(label: "com.example.myqueue", attributes: .concurrent)
customQueue.async {
    // Perform task on custom queue
}
```

### Serial/Concurrent
Queues can be either serial or concurrent, which determines how they execute tasks.

**Serial Queue:**
* Executes one task at a time in the order they were added.
* Useful for maintaining a specific order of execution or for synchronizing access to a shared resource.

**Concurrent Queue:**
* Can execute multiple tasks simultaneously.
* Useful for parallelizing independent tasks to improve performance.

The main queue is always serial. Global queues are always concurrent. For custom queues, you can specify whether they are serial or concurrent when creating them:

```swift
// Serial Queue
let serialQueue = DispatchQueue(label: "com.example.serialQueue")

// Concurrent Queue
let concurrentQueue = DispatchQueue(label: "com.example.concurrentQueue", attributes: .concurrent)
```

## The Relationship Between Threads and Queues
Threads are actual paths of execution managed by the operating system. Queues are abstractions used to organize and manage the execution of tasks. iOS uses a dynamic thread pool to execute tasks from queues. The system, not the developer, manages the creation and destruction of threads.

It works like this:

1. **Queue Submission:** When you submit a task to a queue (background or otherwise), you're not directly assigning it to a specific thread.

2. **Thread Assignment:** The system dynamically assigns tasks from queues to available threads in the thread pool. A single thread might execute tasks from different queues at different times.

3. **Concurrent Execution:** When using concurrent queues, multiple tasks can be executed simultaneously on different threads. But remember, a single thread executes only one task at a time, regardless of which queue it came from.

## Combinations of Sync/Async and Serial/Concurrent Queues
**Sync on Serial Queue**
* Tasks are executed one at a time in the order they were added.
* The calling thread is blocked until the task completes.
* Simple and predictable, but can lead to deadlocks if not used carefully.

```swift
let serialQueue = DispatchQueue(label: "com.example.serialQueue")

func performTask(id: Int) {
    print("Task \(id) starts")
    Thread.sleep(forTimeInterval: 1)
    print("Task \(id) ends")
}

for i in 1...3 {
    serialQueue.sync { performTask(id: i) }
}
print("Serial Sync finished")

/*
Task 1 starts
Task 1 ends
Task 2 starts
Task 2 ends
Task 3 starts
Task 3 ends
Serial Sync finished
*/
```

**Async on Serial Queue**
* Tasks are executed one at a time in the order they were added.
* The calling thread continues execution immediately without waiting for the task to complete.
* Useful for offloading tasks that need to be executed in a specific order without blocking the current thread.

```swift
for i in 1...3 {
    serialQueue.async { performTask(id: i) }
}
print("Serial Async finished（Note: Queue may have tasks still executing）")

/*
Task 1 starts
Serial Async finished（Note: Queue may have tasks still executing）
Task 1 ends
Task 2 starts
Task 2 ends
Task 3 starts
Task 3 ends
*/
```

**Sync on Concurrent Queue**
* The calling thread is blocked until the specific task it submitted completes.

```swift
let concurrentQueue = DispatchQueue(label: "com.example.concurrentQueue", attributes: .concurrent)

for i in 1...3 {
    concurrentQueue.sync { performTask(id: i) }
}
print("Concurrent Sync finished")

/*
Task 1 starts
Task 1 ends
Task 2 starts
Task 2 ends
Task 3 starts
Task 3 ends
Concurrent Sync finished
*/
```
In this example, tasks are dispatched from the same thread to a concurrent queue, the order of execution is preserved. 

While this behavior might seem similar to a serial queue, it's important to note that a concurrent queue can still execute other tasks concurrently if they're dispatched from different threads.

```swift
for i in 1...6 {
    if i % 2 == 0 {
        DispatchQueue.global().async {
            concurrentQueue.sync { performTask(id: i) }
        }
    } else {
        concurrentQueue.sync { performTask(id: i) }
    }
}
print("Concurrent Sync finished")

/*
Task 1 starts
Task 1 ends
Task 3 starts
Task 2 starts
Task 3 ends
Task 5 starts
Task 4 starts
Task 2 ends
Task 5 ends
Concurrent Sync finished
Task 4 ends
Task 6 starts
Task 6 ends
*/
```

**Async on Concurrent Queue**
* Multiple tasks can be executed simultaneously on different threads.
* The calling thread continues execution immediately without waiting for the task to complete.
* Offers the highest level of concurrency and is ideal for independent, time-consuming tasks.

```swift
for i in 1...3 {
    concurrentQueue.async { performTask(id: i) }
}
print("Concurrent Async finished（Note: Queue may have tasks still executing）")

/*
Task 1 starts
Task 2 starts
Task 3 starts
Concurrent Async finished（Note: Queue may have tasks still executing）
Task 3 ends
Task 2 ends
Task 1 ends
*/
// Note: Tasks may complete out of order
```