---
layout: post
title: 'Do Not Create Your SUT in the setUp Method'
categories: [iOS dev]
tag: [swift, unit-test, tdd]
---

Many people get confused of the lifecycle of the `XCTestCase` the first time they approach it. A common mistake we tend to make is like the following example: 

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

XCTestFramework initiates a unique instance for each tests. Therefore, the sut class cannot be shared through multiple test cases. 

Whenever we create a unit test class file, the template offers us `setUp` and `tearDown` default implementations. As addressed by the comments, we will invoke the `setUp` method before we execute each test and call the `tearDown` after we finish the case. Many developers tend to create their system under test (SUT) inside the `setUp` and set it to `nil` in the `tearDown` like the following code:

```swift

var sut: SomeClass!

override func setUpWithError() throws {
    sut = SomeClass()
}

override func tearDownWithError() throws {
    sut = nil
}
```

