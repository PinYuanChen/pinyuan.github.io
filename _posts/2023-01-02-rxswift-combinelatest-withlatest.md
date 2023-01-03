---
layout: post
title: 'RxSwift: combineLatest, withLatestFrom, zip, merge'
categories: [iOS dev]
tag: [swift, rxswift]
---

Let's get straight to the point. What's the differences?

## combineLatest
It combines multiple observables and whenever each of them emits an item, it will fetch other sources' latest item and put them together.  

![marble-combineLatest](/assets/posts-images/marble-combinelatest.png)

```swift
let first = PublishSubject<String>()
let second = PublishSubject<String>()
let disposeBag = DisposeBag()

Observable
    .combineLatest(first, second) { ($0, $1) }
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)

first.onNext("1")
second.onNext("A")
first.onNext("2")
second.onNext("B")
second.onNext("C")
second.onNext("D")
first.onNext("3")
first.onNext("4")

/*
 print out
("1", "A")
("2", "A")
("2", "B")
("2", "C")
("2", "D")
("3", "D")
("4", "D")
*/
```

Notice that at the beginning when the first emitted "1", the `combineLatest` operator didn't pass the value downstream. Until the second sent "A" then the `combineLatest` formed them into a tuple. From then on, whenever each of them sent an item, it would combined with the other source's latest item. You can only put up to 8 parameters into `combineLatest`. And...of course, you [hack](https://stackoverflow.com/questions/57071640/more-than-8-parameters-in-combinelatest-using-rxswift) it by combining other `combineLatest` observers to observe more than 8 sources.

## withLatestFrom
`withLatestFrom` might sounds similar to `combineLatest` but actually their mechanisms are different. You can only put one parameter in `withLatestFrom`. Let's take a look at an example:

```swift
first
    .withLatestFrom(second) { $0 + $1 }
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
        
first.onNext("1")
second.onNext("A")
first.onNext("2")
second.onNext("B")
second.onNext("C")
second.onNext("D")
first.onNext("3")
first.onNext("4")

/* print out
2A
3D
4D
*/
```
The rule is: when the first emits an item, it will fetch the second source and pass them to the subscriber. And it is the same as `combineLatest`, if your second source has no value, `withLatestFrom` won't pass the result. On top of that, notice that when the second emits "B" and "C", the print function doesn't fire. The reason is that we observe the change of the first. It is when the first passes a new item then we will check the latest value from the second. If the first doesn't emit a new value, the observer doesn't care about the second source's changes.  

![marble-withLatestFrom](/assets/posts-images/marble-withlatestfrom.png)

## zip

![marble-zip](/assets/posts-images/marble-zip.png)

## merge

![marble-merge](/assets/posts-images/marble-merge.png)