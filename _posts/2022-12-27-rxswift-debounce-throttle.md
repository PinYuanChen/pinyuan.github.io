---
layout: post
title: 'Debounce and Throttle in RxSwift'
categories: [iOS dev]
tag: [swift, rxswift]
---

In some cases, we need to restrict the number of query in a short period of time, so we can avoid overhead and make sure the app runs properly. That's when debounce and throttle weight in. These two are similar but have significant different mechanisms. Some people may be confused of their usages. This article will dive into differences between the both, and give brief intro about their usages. 

## Definitions
The general definitions of these two are

* Debounce: It will set up a timer and whenever it detects a new event coming and the timer isn't stop, it will reset the timer. After the timer run out, then it will perform the function. 

* Throttle: It will fire once in a given time. It will take the first item and ignore the rest.

Both of them can deal with the condition I mentioned at the beginning of the article. Luckily, RxSwift has implemented debounce and throttle for us so we don't need to make them by ourselves. But I want to mention that throttle in RxSwift has a slightly difference from the general definition. The default throttle takes the first and the last event. The throttle api looks like this `throttle(_ dueTime:latest:scheduler:)` The official doc says

>
Returns an Observable that emits the first and the latest item emitted by the source Observable during sequential time windows of a specified duration.
>

The default value of `latest` argument is set to `true`. So if you only want to get the first item, you simply set `latest` to `false`. 


## Use Cases
Knowing their features, you can decide when to use them at given circumstances.  

The most common use case of debounce is searching. If we need to send an API for searching, we would probably not query it every time the user types a character for it produced lots of unnecessary costs. Instead, we would like to call API after users finish typing.

```swift
    searchBar
    .rx
    .text
    .debounce(.seconds(1), scheduler: MainScheduler.instance)
    .subscribe(onNext: { [weak self] query in
        // do API request and reload data
    })
    .disposed(by: disposeBag)
```

By using debounce, we make sure that the app will wait 1 second after users finish typing then fire up the request. 

The case of using throttle is any situations that you only want to trigger it when the first item comes in. For example, a send button. You only want to take the first click and ignore other one during a time span. Then the subscription would look like this:

```swift
    button
    .rx
    .tap
    .throttle(.milliseconds(100), latest: false, scheduler: MainScheduler.instance)
    .subscribe(onNext: { [weak self] query in
        // get the first item and do something
    })
    .disposed(by: disposeBag)
```

