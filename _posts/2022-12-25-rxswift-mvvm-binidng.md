---
layout: post
title: 'MVVM + RxSwift: An Elegant Way to Bind ViewModels'
categories: [iOS dev]
tag: [swift, mvvm, rxswift]
---

You might have read many articles talking about how to bind view models via RxSwift. But most of them are not suitable for practical use cases. Here I want to demonstrate you how I use MVVM with RxSwift in my job, and it is

* Easy to track data flow
* Scalable for complex scenarios
* Fully testable

## Two Common Approaches
Chances are you have read this type of binding
```swift
protocol ViewModelType {
  associatedtype Input
  associatedtype Output
  
  func transform(input: Input) -> Output
}
```
The drawback is obvious: you have to provide all the inputs and outputs and deal with them at the same time. It's not an ideal approach because in reality the inputs and outputs might trigger in different time. So you might seen another alternative
```swift
protocol ViewModelType {
    associatedtype Input
    associatedtype Output

    var input: Input { get }
    var output: Output { get }
}

class SomeViewModel: ViewModelType {
    let input: Input
    let output: Output

    struct Input {
        // define your input
    }

    struct Output {
        // define your output
    }

    init() {
        // deal with inputs and outputs binding
    }
}
``` 
This method creates two private structs to handle input and output. But the problem is it deal with them in initialization and once you have multiple inputs and outputs subjects, the `init()` will become pretty messy. 

## Protocol Comes into Play

## Example