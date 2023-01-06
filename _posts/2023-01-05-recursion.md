---
layout: post
title: 'A Recursion Issue'
categories: [iOS dev]
tag: [swift, recursion, closure]
---

I assume you have basic concepts about Retain Cycle and know how to avoid it by adding `[weak self]` inside a closure. Chances are you will use `guard` to unwrap `self` to make your code looks sharp. But do you know under some circumstances it will cause problems?

Take a look at this example, you might spot the problem right away.

```swift
class PayAPI {
    class func payMoney(completion: @escaping () -> ()) {
        completion()
    }
}

class DetailViewController: UIViewController {
    
    override func viewDidLoad() {
        super.viewDidLoad()
        pay()
    }
    
    var retryCount = 10
    
    func pay() {
        PayAPI.payMoney { [weak self] in
            guard let self = self else { return }
            if self.retryCount > 0 {
                DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
                    print("count down \(self.retryCount)")
                    self.retryCount -= 1
                    self.pay()
                }
            } else {
                print("I paid the bill")
            }
        }
    }
    
    deinit {
        print("I left.")
    }
}

```

Apple's [document](https://developer.apple.com/documentation/xcode/making-changes-to-reduce-memory-use)

>
A memory leak occurs when allocated memory becomes unreachable and the app can’t deallocate it. Allowing an allocated-memory pointer to go out of scope without freeing the memory can cause a memory leak.
>