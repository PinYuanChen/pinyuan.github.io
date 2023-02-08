---
layout: post
title: 'Do Not Create Your SUT in the setUpWithError'
categories: [iOS dev]
tag: [swift, unit-test, tdd]
---

As a beginner in unit testing, you may face confusion when setting up your system under test (SUT). If you're not familiar with the workings of the XCTest Framework, you may encounter unexpected errors. Additionally, different test scenarios may require different SUT configurations, making it unclear how to set up your SUT in the most efficient way. In this article, I'll delve into the lifecycle of test cases and provide a better method for creating a SUT instead of initializing it in the `setUpWithError` method.

## A XCTestCase's Lifecycle
A common mistake is like this: 

```swift
import XCTest

class SomeClass {
    var val = 0
}

class SomeClassTests: XCTestCase {

    var sut = SomeClass()

    func test_sut_addOne() {
        sut.val += 1
        XCTAssertEqual(sut.val, 1)
    }

    func test_sut_addAnotherOne() {
        sut.val += 1
        XCTAssertEqual(sut.val, 2) 
        // test_sut_addAnotherOne(): XCTAssertEqual failed: ("1") is not equal to ("2")
    }

}
```

We create the SUT in the `SomeClassTests` scope and assume its property `val` will persist across multiple test cases. However, the failing test message shows it's not, as the `val` in `test_sut_addAnotherOne` doesn't equal 2. This happens because the XCTestFramework creates a unique instance for each test case, meaning that the SUT is independent between test cases and does not persist from one case to another.

One solution to this problem might be to initialize the SUT inside each test case, but this approach can quickly become cumbersome. To avoid this, many developers create the SUT inside the `setUpWithError` method and set it to `nil` in the `tearDownWithError` method (the old names before Xcode 11.4 were `setUp` and `tearDown`). The `setUpWithError` method is invoked before each test case, and the `tearDownWithError` method is called after each test case has finished. Here is an example of this implementation:

```swift
var sut: SomeClass!

override func setUpWithError() throws {
    sut = SomeClass()
}

override func tearDownWithError() throws {
    sut = nil
}
```

The drawback of this centralized initialization approach is that it lacks flexibility in configuring the SUT. As previously mentioned, we need to test various scenarios that may require different setups. This method makes it challenging to modify the SUT. While property injection can be used to achieve this, it would add a lot of boilerplate code to each test case. On top of that, you would need to scroll up or down to check the SUT setup, making it harder to understand the context and read the code.

## Create SUTs via Factory Methods

An ideal solution to this problem is to use a factory method. By using constructor injection, you can tailor your SUT to match the specific requirements of each test case. For example:

```swift
private func makeSUT(account: String? = nil, password: String? = nil) -> Validator {
    return Validator(account: account, password: password)
}
```

You can call this helper function to create the SUT you need:

```swift
func test_empty_account {
    let sut = makeSUT(password: "any pwd")
    // ...
}

func test_empty_password {
    let sut = makeSUT(account: "any account")
    // ...
}
```