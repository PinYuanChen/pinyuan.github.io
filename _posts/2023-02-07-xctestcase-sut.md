---
layout: post
title: 'Do Not Create Your SUT in the setUpWithError'
categories: [iOS dev]
tag: [swift, unit-test, tdd]
---

As a newbie approaching the unit test, you may find it confusing to configure your system under test (SUT). You may lead to unexpected error if you are not familiar with the XCTest Framework's mechanism. On top of that, in different scenarios you might need different configurations of your SUT, and you might wonder what's the more efficient way to deal with it. In this article, I am gonna dive in test cases' lifecycle, and introduce a better way to create SUTs rather than initialize them in the `setUpWithError`.

## A XCTestCase's Lifecycle
A common mistake people made is like this example: 

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

We create a sut in the `SomeClassTests` scope and assume its property `val` will continue being modified as it passes through a series of tests. However, things don't go as expected, the failing message shows that the `val` doesn't equal to 2. How does that happen?

XCTestFramework initiates a unique instance for each tests, that is, they are independent from each other that they don't share the same sut. Since each cases has its fresh new sut, should we initiate it inside the test case scope to avoid confusion? Yet this seems cumbersome. 

Many developers tend to create their system under test (SUT) inside the `setUpWithError` and set it to `nil` in the `tearDownWithError` (the old names before Xcode 11.4 are `setUp` and `tearDown`). As addressed by the comments, the compiler will invoke the `setUpWithError` method before executing a test case and call the `tearDownWithError` after finishing it. The implementation looks like the following:

```swift
var sut: SomeClass!

override func setUpWithError() throws {
    sut = SomeClass()
}

override func tearDownWithError() throws {
    sut = nil
}
```

There are several downsides of this option:


## Create SUTs via Factory Methods