---
layout: post
title: 'RxSwift: combineLatest, withLatestFrom, zip, merge'
categories: [iOS dev]
tag: [swift, rxswift]
---

Let's get straight to the point. What's the differences?

## combineLatest
It combines multiple observable sources and when each of them emits an item, it will fetch the other sources' latest items and merge them into a single observable.

![marble-combineLatest](/assets/posts-images/marble-combinelatest.png)

```swift
let first = PublishSubject<Int>()
let second = PublishSubject<String>()
let disposeBag = DisposeBag()

Observable
    .combineLatest(first, second) { ($0, $1) }
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)

first.onNext(1)
second.onNext("A")
first.onNext(2)
second.onNext("B")
second.onNext("C")
second.onNext("D")
first.onNext(3)
first.onNext(4)

/*
 print out
(1, "A")
(2, "A")
(2, "B")
(2, "C")
(2, "D")
(3, "D")
(4, "D")
*/
```

Notice that at the beginning, when the first emitted "1", the `combineLatest` operator didn't do anything. Because at this point the second source didn't produce any item. Until the second source produced the first string "A" then the `combineLatest` started working. From then on, whenever each of them emitted a fresh item, the operator combined it with the other one's latest item and passed through to the subscriber. 

You can only put up to 8 sources into `combineLatest`. But...of course, you [hack](https://stackoverflow.com/questions/57071640/more-than-8-parameters-in-combinelatest-using-rxswift) it by combining other `combineLatest` observers to surpass the number limit.

## withLatestFrom
`withLatestFrom` might sounds similar to `combineLatest` but actually their mechanisms are quite different. Let's take a look at an example:

```swift
let first = PublishSubject<String>()
let second = PublishSubject<String>()
let disposeBag = DisposeBag()

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
The rule is: when the first source emits an item, it will fetch the second source's latest item and pass them to the subscriber. You can only put one parameter in `withLatestFrom`. And jst like `combineLatest`, `withLatestFrom` only works when both sources have at least one value. On top of that, notice when the second emits "B" and "C", the print function doesn't fire. The reason is that we observe the change of the first source. The first source is the main character and only when it passes a new item then will we check the the second source's value. If the first source doesn't emit a new one, the observer doesn't care about the second source's changes.  

![marble-withLatestFrom](/assets/posts-images/marble-withlatestfrom.png)

## zip
`zip` is similar to `combineLatest` in many ways: they both combine up to 8 sources, they are triggered when each of sources emit new items. The main difference is `zip` fetches items in order. Here is an usage example:

```swift
let first = PublishSubject<String>()
let second = PublishSubject<String>()
let disposeBag = DisposeBag()

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

You can think of it as using an index to fetch elements from two different arrays. The sources' elements pair according to the same index.

![marble-zip](/assets/posts-images/marble-zip.png)

## merge
`merge` observes multiple sources at the same time and when each of them emits an element, you get that single item right away. Things to note about the `merge` is that the observables' arguments must be the same type.

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