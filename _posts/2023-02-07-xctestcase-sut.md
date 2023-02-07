---
layout: post
title: 'Do Not Create Your SUT in the setUpWithError'
categories: [iOS dev]
tag: [swift, unit-test, tdd]
---

As a newbie approaching the unit test, you may find it confusing to configure your system under test (SUT). You may lead to unexpected error if you are not familiar with the XCTest Framework's mechanism. On top of that, in different scenarios you might need different configurations of your SUT, and you might wonder what's the more efficient way to deal with it. In this article, I am gonna dive in test cases' lifecycle, and introduce a better way to create SUTs rather than initialize them in the `setUpWithError`.

## A XCTestCase's Lifecycle
A common mistake we tend to make is like the following example: 

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

XCTestFramework initiates a unique instance for each tests, that is, they are independent from each other. Therefore, the sut instance cannot be shared through multiple test cases. 

Whenever we create a unit test class file, the template offers us `setUpWithError` and `tearDownWithError` default implementations. As addressed by the comments, we will invoke the `setUpWithError` method before we execute each test and call the `tearDownWithError` after we finish the case. Many developers tend to create their system under test (SUT) inside the `setUpWithError` and set it to `nil` in the `tearDownWithError` like the following code:

```swift

var sut: SomeClass!

override func setUpWithError() throws {
    sut = SomeClass()
}

override func tearDownWithError() throws {
    sut = nil
}
```

## Create SUTs via Factory Methods