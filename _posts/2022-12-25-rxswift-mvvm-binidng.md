---
layout: post
title: 'MVVM + RxSwift: An Elegant Way to Bind ViewModels'
categories: [iOS dev]
tag: [swift, mvvm, rxswift]
---

You might have read many articles talking about how to bind view models via RxSwift. But most of them are not suitable for practical use cases. Here I want to share with you how I use MVVM along with RxSwift in my job, and it is

* Easy to track data flow
* Scalable for complex scenarios
* Fully testable

Without further ado, let's jump right into it.

## Two Common Approaches
Chances are you have already read this type of binding
```swift
protocol ViewModelType {
  associatedtype Input
  associatedtype Output
  
  func transform(input: Input) -> Output
}
```

The drawback is obvious: you have to provide all the inputs and outputs in `func transform`, and you have to deal with them at the same time. It's lack of flexibility because, in reality, the inputs and outputs might trigger at different time. So you might also seen this alternative

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
Well, this method creates two private structs to handle input and output. But the problem is: it deal with them in initialization, and once you have multiple input and output subjects, the `init()` will become pretty messy. 

## Protocol Comes into Play
Let's think of it again: is there a better way to create input and output interfaces? You may ring a bell that protocol might be a better solution. Since Apple claims Swift is a protocol oriented language and protocol can act as a good interface. So let's see how to build it.

```swift
protocol ViewModelInput { 
    func sendAPI()
}
protocol ViewModelOutput { 
    var apiResult: Observable<Model> { get }
}

protocol ViewModelPrototype {
    var output: ViewModelOutput { get }
    var input: ViewModelInput { get }
}
```
First, we declare an input protocol and define a function that get called from outside. Then we declare an output protocol that pass observable data to views for UI binding. Then we define a protocol to put input and output protocols inside.
Then the view model class conforms to `ViewModelPrototype`

```swift
class ViewModel: ViewModelPrototype {    
    var output: ViewModelOutput { self }
    var input: ViewModelInput { self }
}
```
This is the basic structure of protocol design. You may wonder how it work in real cases. Let me walk you through a typical scenario. 

## Usage Example