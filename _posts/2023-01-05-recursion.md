---
layout: post
title: 'A Recursion Issue'
categories: [iOS dev]
tag: [swift, memory leak, closure]
---

I assume you have basic concepts about Retain Cycle and know how to avoid it by adding `[weak self]` inside a closure. Chances are you will use `guard` to unwrap `self` to make your code looks sharp. But do you know under some circumstances it will cause problems?

Take a look at this example, you might spot the problem right away.

```swift

```
