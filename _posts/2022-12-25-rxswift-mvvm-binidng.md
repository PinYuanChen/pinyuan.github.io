---
layout: post
title: 'MVVM + RxSwift: An Elegant Way to Bind ViewModels'
categories: [iOS dev]
tag: [swift, mvvm, rxswift]
---

You might have read many articles about how to bind view models via RxSwift. But most of them are not suitable for practical use cases. Here I want to share with you how I use MVVM along with RxSwift in my job, and it is

* Easy to track data flow
* Scalable for complex scenarios
* Fully testable

Without further ado, let's jump right into it.

## Two Common Approaches
Chances are you have read this type of binding
```swift
protocol ViewModelType {
  associatedtype Input
  associatedtype Output
  
  func transform(input: Input) -> Output
}
```

The drawback is obvious: you have to provide all the inputs and outputs in `func transform`, and you have to deal with them at the same time. It lacks flexibility because, in reality, the inputs and outputs might trigger at different time. So you might also have seen this alternative

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
Well, this method creates two private structs to handle input and output. But the problem is: it deals with them in initialization, and once you have multiple input and output subjects, the `init()` will become pretty messy. 

## Protocol Comes Into Play
Let's think of it: what's the purpose of creating input and output instances? Because they act as `interfaces`. It might ring a bell to you that there's another one who can act as an interface: `protocol`. Since Apple claims Swift is a protocol-oriented language, we developers must explore its full potential. Let's see how to build it in a protocol way.

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
First, we create an input protocol and define a `sendAPI` function. Our view controller will call this function when it needs to fetch data. Then we declare an output protocol that pass API result as an observable to views for UI binding. Last, we define a protocol (`ViewModelPrototype`) to put input and output protocols inside.
The view model class conforms to `ViewModelPrototype` like this:

```swift
class ViewModel: ViewModelPrototype {    
    var output: ViewModelOutput { self }
    var input: ViewModelInput { self }

    private let _apiResult = BehaviorRelay<Model?>(value: nil)
}

extension ViewModel: ViewModelInput {
    func sendAPI() {
        // call api and fetch data
        let result = NetworkManager.callSomeAPI()
        _apiResult.accept(result)
    }
}

extension ViewModel: ViewModelOutput {
    var apiResult: Observable<Model> {
        _apiResult.compactMap { $0 }.asObservable()
    }
}
```
Our view model conforms to input and output protocols in separate extensions. We use a private variable `_apiResult` to store and pass the API result. Then we unwrap the result via `compactMap` and turn it into an observable for a subscription. That's it. This is the basic protocol design. In the next section, I'll walk you through how to bind the view model to the view controller.

## Binding
Let's say we have a view controller and it needs to call an API to get data when it shows up. Here is how we define the view controller:

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

We declare a `viewModel` variable and its type is an existential container that conforms to `ViewModelPrototype`. When we initialize the view controller, we inject the view model into it. In this way, we decouple the view controller and the view model so we can do testing. Then we create the `bind` function in a private extension. The function takes the view model `prototype`, be careful, it's not `ViewModel` type but `ViewModelPrototype`, because it forces us to interact with the view model through interfaces. You must type `.input` or `.output` to access the view model's properties. And for our human beings, it's much more readable for us to trace code because we can easily know whether it's input or output calls. 

That's it. If you have any questions or recommendations, please leave a comment below. See you at the top!