---
layout: post
title: 'Delayed Deallocation Caused By Nested Closures'
categories: [iOS dev]
tag: [swift, recursion, closure]
---

I assume you have basic concepts about Retain Cycle and know how to avoid it by adding `[weak self]` inside a closure. But even if you've done everything right, you might still run into some problems that you cannot release objects as expected. 

The code below demonstrates a recursive call scenario. Upon entering the `DetailViewController`, we execute the `pay` function and call the class function `payMoney`, which is a simplified function simulating API calling. Inside the `payMoney` function, it simply returns an empty closure. But that's not the point. The point is in the `payMoney`'s callback, we monitor the `retryCount` and after one second we call the `pay` function recursively if the `retryCount` is greater than zero. Have you spotted any problems?  

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
Everything looks just fine. We use `[weak self]` inside the `payMoney`'s callback to prevent memory leak. We use `guard` to unwrap `self` as many people will do to get rid of the annoying optional mark("?"). Then we call the `pay` function inside the `asyncAfter`'s closure. Nothing suspicious, right? 

Well, if you leave the `DetailViewController` during the recursive calls, you will see that even the recursion still goes on. You can download the code here and test by yourself. That seems like a memory leak. But how does that happen?

Honestly, though it seems pretty like a memory leak, it's not. Apple's [document](https://developer.apple.com/documentation/xcode/making-changes-to-reduce-memory-use) defines a memory leak as :

>
A memory leak occurs when allocated memory becomes unreachable and the app can’t deallocate it. 
>

The memory is not unreachable at this case. The proof is after the recursion hits the bottom case, the `DetailViewController` will be released. You will see the `deinit` do print out the message. So it is not a leak case, but a "delayed dallocation" issue. That is, the object doesn't release at our expected time.

Let's walk through the code again. Notice that `asyncAfter` is a nested closure. It keeps the reference to the strong self unwrapped by `guard`. So it keeps an additional reference to the `self`. That's why it keeps alive in the background even though you go back to the former page. You can add `[weak self]` inside the `asyncAfter` closure. Then you will see the `DetailViewController` get killed right away when we leave the page. If you feel it too verbose, you can remove the `guard let self = self` line and make all `self`s optional. 

While I was browsing on the Internet, I found Besher Al Maleh's [article](https://medium.com/@almalehdev/you-dont-always-need-weak-self-a778bec505ef) extremely helpful. I highly recommend you read it through. He also offered an epic flowchart to guide you through whether you show add `weak self` inside a closure.

![closure-weak-self-decision](https://miro.medium.com/max/4800/1*yHX-8dJrQpH7R2hfM_21MQ.webp)

That's it! If you have any questions or recommendations, please leave a comment down below. See you at the top! 🤜🤛