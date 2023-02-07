---
layout: post
title: 'Do Not Create Your SUT in setUp Method'
categories: [iOS dev]
tag: [swift, unit-test, tdd]
---

Whenever we create a unit test class file, the template offers us `setUp` and `tearDown` default implementations. Many developers tend to create their system under test(SUT) inside `setUp` and set it to nil in `tearDown`.

```swift

var sut: SomeClass?

override func setUpWithError() throws {
    // Put setup code here. This method is called before the invocation of each test method in the class.
    sut = SomeClass()
}

override func tearDownWithError() throws {
    // Put teardown code here. This method is called after the invocation of each test method in the class.
    sut = nil
}
```