---
layout: post
title: 'Delayed Deallocation Caused By Nested Closures'
categories: [iOS dev]
tag: [swift, recursion, closure]
---

I assume you have basic concepts about Retain Cycles and know how to avoid them by adding `[weak self]` inside closures. However, you may still encounter situations where you cannot release objects as expected even if you have followed best practices.

The code below demonstrates a recursive call scenario. You can download the project [here](https://github.com/PinYuanChen/delayed-dealloc). When entering the `DetailViewController`, we execute the `pay` function and call the class function `payMoney`, which is a simplified function simulating API call. Inside the `payMoney` function, it simply returns an empty closure. In the `payMoney`'s callback, we monitor the `retryCount` and after one second we call the `pay` function recursively if the `retryCount` is greater than zero. Have you spotted any problems?  

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
Everything looks just fine. We use `[weak self]` inside the `payMoney`'s callback to prevent memory leaks. We use `guard` to unwrap `self` as many people will do to get rid of the annoying optional mark("?"). Then, we call the `pay` function inside the `asyncAfter`'s closure. Nothing suspicious, right? 

However, if you leave the `DetailViewController` during the recursive calls, you will see that the recursion continues even though you have left the `DetailViewController`. Hmmm...seems like a memory leak. But how does it happen?

Well, it's actually not a leak. According to Apple's [document](https://developer.apple.com/documentation/xcode/making-changes-to-reduce-memory-use), a memory leak occurs

>
when allocated memory becomes unreachable and the app can’t deallocate it. 
>

The memory is not unreachable in this case. When the recursion hits the bottom case, the `DetailViewController` will be released, as seen by the `deinit` message being printed. This is not a memory leak, but rather a "delayed deallocation" issue, meaning the object is not being released at the expected time.

If we look at the code again, we can see that `asyncAfter` is a nested closure which keeps a reference to the strong self unwrapped by `guard`. This is why the `DetailViewController` remains alive in the background even after you have navigated away from it. To fix this, you can add `[weak self]` inside the `asyncAfter` closure, which will cause the `DetailViewController` to be released immediately when you leave the page. Alternatively, you can remove the `guard let self = self` line and make all `self`s optional. 

While browsing the Internet, I found Besher Al Maleh's [article](https://medium.com/@almalehdev/you-dont-always-need-weak-self-a778bec505ef) to be extremely helpful. I highly recommend reading it. The article also includes an epic flowchart for determining when to use `weak self` inside a closure.

![closure-weak-self-decision](https://miro.medium.com/max/4800/1*yHX-8dJrQpH7R2hfM_21MQ.webp)

Maleh's [demo sample](https://github.com/almaleh/weak-self) also provides an example of delayed deallocation caused by a sempaphore:

```swift
func delayedAllocSemaphore() {
    DispatchQueue.global(qos: .userInitiated).async {
        let semaphore = DispatchSemaphore(value: 0)
        print(self.view.description)
        _ = semaphore.wait(timeout: .now() + 99.0)
    }
}
```

Besides, if you set a long time interval in the `URLSessionConfiguration`, it will also lead to delayed deallocation.

```swift

/* If you execute the URLSession task immediately, but set a long timeout interval, it will delay the deallocation of your controller until you either cancel the task, get a response back, or timeout. Using [weak self] will prevent that delay. Note: Using port 81 on the url helps simulate a request timeout */

func delayedAllocAsyncCall() {
    let url = URL(string: "https://www.google.com:81")!
        
    let sessionConfig = URLSessionConfiguration.default
    sessionConfig.timeoutIntervalForRequest = 999.0
    sessionConfig.timeoutIntervalForResource = 999.0
    let session = URLSession(configuration: sessionConfig)
        
    let task = session.downloadTask(with: url) { localURL, _, error in
        guard let localURL = localURL else { return }
        let contents = (try? String(contentsOf: localURL)) ?? "No contents"
        print(contents)
        print(self.view.description)
    }
    task.resume()
}
```
