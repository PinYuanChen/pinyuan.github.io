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
        // initialize two structs and
        // deal with inputs and outputs binding
    }
}
``` 
Well, this method creates two private structs to handle input and output. But the problem is: it deal with them in initialization, and once you have multiple input and output subjects, the `init()` will become pretty messy. 

## Protocol Comes into Play
Let's think of it: what's the purpose of creating input and output instances? Because they act as `interfaces`. It might ring a bell to you that there's another one can act as a interface: `protocol`. Since Apple claims Swift is a protocol oriented language, we developers must explore its full potential. Let's see how to build it in a protocol way.

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
This is the basic structure of protocol design. You may wonder how it work in real cases and what's the benefit. Let me walk you through a typical scenario. 

## Usage Example
Let's say we have a view controller and it needs to call an api to get data when it shows up. Here is how we define the view controller
```swift
class ViewController: UIViewController {
    var viewModel: ViewModelPrototype?

    override func viewDidLoad() {
        super.viewDidLoad()
        guard let viewModel = viewModel else { return }
        bind(viewModel)
    }

    private let disposeBag = DisposeBag()
}

private extension ViewController {
    func bind(_ viewModel: ViewModelPrototype) {
        viewModel
            .output
            .apiResult
            .withUnretained(self)
            .subscribe(onNext: { owner, result in
                // deal with the api result
            })
            .disposed(by: disposeBag)

        viewModel
            .input
            .sendAPI()
     }
}
```
We declare a `viewModel` variable and its type is an existential container that conforms to `ViewModelPrototype`. When we initialize the view controller, we inject the view model into it as well. In this way, we decouple the vc and vm so we can do testing. Then we create the `bind` function in private extension. The function takes the view model `prototype`, be careful, it's not `ViewModel` but `ViewModelPrototype`, because it forces us to interact with view model through interfaces. You must type `.input` or `.output` to access the view model's properties. And for our human beings, it's much readable for us to trace code because we can easily know whether it's input or output calling. 

That's it. I hope you enjoy it. If you have any question or recommendation, please leave a comment below.