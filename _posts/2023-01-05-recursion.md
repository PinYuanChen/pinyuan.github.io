---
layout: post
title: 'Late Deallocation Caused By Nested Closures'
categories: [iOS dev]
tag: [swift, recursion, closure]
---

I assume you have basic concepts about Retain Cycle and know how to avoid it by adding `[weak self]` inside a closure. But even if you've done everything right, you might still run into some problems that you cannot release objects at proper time. 

The code below shows a recursive call scenario. Upon entering the `DetailViewController`, we execute the `pay` function and call the class function `payMoney`, which is a simplified function simulating API calling. Inside the `payMoney` function, it simply returns an empty closure. But that's not the point. The point is in the `payMoney`'s callback, we monitor the `retryCount` and after one second we call the `pay` function recursively if the `retryCount` is greater than zero. Have you spotted any problems?  

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
Everything looks just fine. We use `[weak self]` inside the `payMoney`'s callback to prevent memory leak. We use `guard` to unwrap `self` as many people will do to get rid of optional mark("?"). Then we call the `pay` function inside the `asyncAfter`'s closure. Nothing suspicious, right? 

Well, if you leave the `DetailViewController` during the recursive calls, you will find that even the recursion still goes on. You can download the code here and test by yourself. That seems like a memory leak. But how does it happen?

Although it seems like a memory leak, it's actually not. Apple's [document](https://developer.apple.com/documentation/xcode/making-changes-to-reduce-memory-use) defines a memory leak as :

>
A memory leak occurs when allocated memory becomes unreachable and the app can’t deallocate it. 
>

At this case, the memory is not unreachable. The proof is after the recursion hits the bottom case, the `DetailViewController` will be released. You will see the `deinit` do print out the message. So it is not a leak case, but a "late dallocation" issue. That is, the object doesn't release at your expected moment.
