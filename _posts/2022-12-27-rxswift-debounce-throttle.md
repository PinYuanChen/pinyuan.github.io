---
layout: post
title: 'Debounce and Throttle in RxSwift'
categories: [iOS dev]
tag: [swift, rxswift]
---

In some cases, we need to restrict the number of queries in a time span, so we can avoid overhead and make sure the app runs properly. That's when debounce and throttle weight in. These two are similar but have significantly different mechanisms. Some people may be confused about their usage. This article will dive into differences between the both, and give a brief intro about their usage. 

## Definitions
The general definitions of these two are

* Debounce: Whenever a new item comes, it will set up a timer. If the next item comes in and the timer is still running, it will reset the timer. After the timer runs out, it will perform the function. 

* Throttle: It will fire once in the period and only take the first item, ignoring the rest.

Both of them can deal with the condition I mentioned at the beginning of the article. Luckily, RxSwift has implemented debounce and throttle for us so we don't need to make them by ourselves. But I want to mention that throttle in RxSwift has a slight difference from the general definition. The default throttle in RxSwift takes the first and the last items. The throttle's API looks like this `throttle(_ dueTime:latest:scheduler:)` The official doc says

>
Returns an Observable that emits the first and the latest item emitted by the source Observable during sequential time windows of a specified duration.
>

The default value of the `latest` argument is set to `true`. So if you only want to get the first item, you simply set `latest` to `false`. 


## Use Cases
Knowing their features, you can decide when to use them under given circumstances.  

The most common use case of debounce is searching. If the search function requires calling an API, we would probably not query it every time the user types a character. Instead, we would like to call API after users finish typing.

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

By using debounce, we make sure that the app will wait 1 second after users finish typing and then fire up the request. 

The case of using throttle is in any situation you want to trigger it when the first item comes in. For example, a send button. You only want to take the first click and ignore other one. Then the subscription would look like this:

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
Remember, if you don't set the `latest` to `false`, you will receive both the first and the last items.

That’s it. If you have any questions or recommendations, please leave a comment below. See you at the top!