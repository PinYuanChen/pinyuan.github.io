---
layout: post
title: 'Retain Cycles in Closures'
categories: [iOS dev]
tag: [swift, memory leak, closure]
---

I assume you have basic concepts about Retain Cycle and know how to avoid it by adding `[weak self]` inside a closure. But I wanna ask you: do you have to add `[weak self]` to all closures? And do you know when you unwrap `self` by using `guard` might cause problems under some circumstances?

Take a look at this example, you might spot the problem right away.

```swift
func payOne() {
    DispatchQueue.main.asyncAfter(deadline: .now() + 5) {
        self.printOut()
    }
}
```
