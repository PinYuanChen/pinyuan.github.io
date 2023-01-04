---
layout: post
title: 'RxSwift: combineLatest, withLatestFrom, zip, merge'
categories: [iOS dev]
tag: [swift, rxswift]
---

Let's get straight to the point. What's the differences?

## combineLatest
It combines multiple observable sources and when each one of them emits an item, it will fetch other sources' latest items, merge them and pass the combination as a single observable.

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

Notice that at the beginning when the first emitted "1", the `combineLatest` operator didn't do anything. Because at this point the second source didn't produce any item yet. Until the second sent "A" then the `combineLatest` started working. From then on, whenever each one of them produced a fresh item, the operator combined it with the other source's latest value and passed through to the subscriber. 

You can only put up to 8 parameters into `combineLatest`. But...of course, you [hack](https://stackoverflow.com/questions/57071640/more-than-8-parameters-in-combinelatest-using-rxswift) it by combining other `combineLatest` observers to surpass the number limit.

## withLatestFrom
`withLatestFrom` might sounds similar to `combineLatest` but actually their mechanisms are quite different. You can only put one parameter in `withLatestFrom`. Let's take a look at an example:

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
`zip` is similar to `combineLatest` in many ways: they both combine up to 8 streams, they are triggered when each of streams emit items. The main difference is `zip` fetches items from sources in order. 

```swift
Observable
    .zip(first, second) { $0 + $1 }
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
1A
2B
3C
4D
*/
```

You can think of it like using an index to fetch elements from two different arrays. Here is the marble diagram for `zip`:

![marble-zip](/assets/posts-images/marble-zip.png)

## merge
`merge` simply combines multiple sources and when each of them emits an element you will get noticed. 

```swift
Observable
    .of(first, second)
    .merge()
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
1
A
2
B
C
D
3
4
*/
```

![marble-merge](/assets/posts-images/marble-merge.png)

That's it! If you have any questions or recommendations, please leave a comment down below. See you at the top! 😎