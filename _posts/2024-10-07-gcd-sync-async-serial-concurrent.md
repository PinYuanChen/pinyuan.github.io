---
layout: post
title: 'GCD: Understand sync, async and serial, concurrent'
categories: [iOS dev]
tag: [swift, gcd, thread, queue, serial, concurrent]
---

Normally we use `DispatchQueue()` API provided by Apple to handle threading features in our daily works. But few will know the mechanism underneath the hood and jargons like `sync`, `async`, `serial` or `concurrent` means. All we need to know it's when we have to do something related to UI we should switch back to main queue, right? Otherwise, the app will crash to inform you make a unforgiven mistake.

In this article, I am going to introduce the mechanism of GCD, what is `Thread` and `Queue`, and their features. So you can have a clear picture of how they work together and what potential issues you might need to avoid.

// GCD mechanism
// Main Thread and Background thread
// sync and async for thread, sync block current thread
// serial and concurrent for queue
// combinations, sync + concurrent

