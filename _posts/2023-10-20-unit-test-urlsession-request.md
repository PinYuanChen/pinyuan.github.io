---
layout: post
title: 'Using URLProtocol for Unit Test'
categories: [iOS dev]
tag: [swift, url protocol, url session, unit test, networking]
---

At the early stages of development, it's common for the backend server to be unfinished. So, how can app developers create tests for network requests in these situations? Furthermore, even with a functioning server, unstable network conditions can lead to inconsistent test results and longer test durations, potentially slowing development. How can we tackle these challenges while still ensuring robust testing for our apps?

As to the first question, the answer is "YES". Even without a complete server, you can still craft tests to assess various scenarios using networking frameworks like URLSession, Alamofire, and others. According to Apple's testing guidelines presented in [WWDC]('https://developer.apple.com/videos/play/wwdc2018/417'), our test suite should resemble a pyramid. At its base are numerous unit tests, followed by a smaller set of medium-sized integration tests, and capped off with a select few end-to-end tests.
 
>
These(Unit tests) are characterized by being simple to read, producing clear failure messages when we detect a problem, and by running very quickly, often in the order of hundreds or thousands of tests per minute.
>

In this post, I'll delve into writing unit tests for requests without actually initiating network tasks. I will probably write another post about end-to-end tests in the future, for now, our focus will solely be on unit testing.

Below is a conventional API protocol that outlines the result and the corresponding load function.

```swift
public protocol APIClient {
    typealias Result = Swift.Result<(Data, HTTPURLResponse), Error>
    func load(request: URLRequest, completion: @escaping (Result) -> Void)
}
```

We've created a class called `URLSessionAPIClient` that conforms to the `APIClient` protocol.

```swift
public class URLSessionAPIClient: APIClient {
    private let session: URLSession
    
    public init(session: URLSession = .shared) {
        self.session = session
    }
    
    private struct UnexpectedValuesRepresentation: Error {}
    
    public func load(request: URLRequest, completion: @escaping (APIClient.Result) -> Void) {
        session.dataTask(with: request) { data, response, error in
            completion(Result {
                if let error = error {
                    throw error
                } else if let data = data, let response = response as? HTTPURLResponse {
                    return (data, response)
                } else {
                    throw UnexpectedValuesRepresentation()
                }
            })
        }.resume()
    }
}
```

This class contains a `URLSession` property, which it utilizes for making requests. You have the option to inject a `URLSession` instance or to use the default `shared` one. Next, we'll establish a test file named `URLSessionAPIClientTests`. For creating instances in our tests, we'll use the previously mentioned [factory method](https://pinyuanchen.github.io/posts/xctestcase-sut/).

```swift
final class URLSessionAPIClientTests: XCTestCase {
    // helpers
    private func makeSUT() -> APIClient {
        let sut = URLSessionAPIClient()
        return sut
    }
}
```

Time to write some tests. We aim to ensure that the behaviors of `URLSessionAPIClient` match our expectations. Initially, we'll assess if it can accurately initiate a GET request.

```swift
func test_getFromURL_performsGETRequestWithURL() { }    
```

A challenge arises: how can we initiate a request using a mock URL? 

The solution lies in intercepting the request to return a response aligning with our specifications. And here comes a handy tool called `URLProtocol`, which fulfills what we need. Despite its name, `URLProtocol` isn't a protocol but rather a class within the URL loading system. When a URL session request is initiated, the system automatically creates a `URLProtocol` instance to manage the process. Our objective is to simulate `URLProtocol` and intercept the request by implementing the requisite methods.

When subclassing `URLProtocol`, there are four essential methods to implement.

>
class func canInit(with request: URLRequest)
class func canonicalRequest(for request: URLRequest) -> URLRequest
func startLoading()
func stopLoading()
>

In the `startLoading` section lies the crux of our implementation. Here, we'll ascertain if any stubbed data exists and, if so, return them as responses, essentially intercepting the request. The four methods are implemented as follows:

```swift
private class URLProtocolStub: URLProtocol {
    override class func canInit(with request: URLRequest) -> Bool {
        return true
    }
        
    override class func canonicalRequest(for request: URLRequest) -> URLRequest {
        return request
    }

    override func startLoading() {
        // We check our stubbed artifacts here.
    }

    override func stopLoading() {}
}
```

For `canInit`, we straightforwardly return `true`, ensuring the request is valid, and the same logic applies to `canonicalRequest`. As there's no need for any specific action in `stopLoading`, it remains empty. Our primary focus then shifts to the `startLoading` method. We'll require a property to capture and later verify the request. This can be achieved using a closure property.

```swift
private class URLProtocolStub: URLProtocol {
    private static var requestObserver: ((URLRequest) -> Void)?

    static func observeRequests(observer: @escaping (URLRequest) -> Void) {
        requestObserver = observer
    }
}
```
Given that the URL loading system autonomously creates the `URLProtocol` instance, we lack control over its instantiation. As a workaround, we designate `requestObserver` as a static property. Furthermore, we introduce a static function, `observeRequests`, furnishing an external interface for storing the property. Subsequently, we append the following code to the `startLoading` function:"

```swift
override func startLoading() {
    if let requestObserver = URLProtocolStub.requestObserver {
        client?.urlProtocolDidFinishLoading(self)
        return requestObserver(request)
    }
    client?.urlProtocolDidFinishLoading(self)
}

```
The code checks for the presence of a request observer. If it finds one, it completes the loading and triggers the callback.

Now, returning to the test file. Before diving into testing, there's an essential step to finalize the `URLProtocol`. In `setupWithError`, it's crucial to register our `URLProtocolStub` to ensure its functionality.

```swift
override func setUpWithError() throws {
    try super.setUpWithError()
    URLProtocol.registerClass(URLProtocolStub.self)
}
```

Additionally, in `tearDownWithError`, it's necessary to unregister it and also release the observer closure.

```swift
override func tearDownWithError() throws {
    try super.tearDownWithError()
    URLProtocol.unregisterClass(URLProtocolStub.self)
    requestObserver = nil
}
```

For clarity, we encapsulate these steps into two static functions within `URLProtocolStub`, signifying their specific roles.

```swift
private class URLProtocolStub: URLProtocol {
    //...
    static func startInterceptingRequests() {
        URLProtocol.registerClass(URLProtocolStub.self)
    }
        
    static func stopInterceptingRequests() {
        URLProtocol.unregisterClass(URLProtocolStub.self)
        requestObserver = nil
        stub = nil
    }
    //...
}
```
In the test file, the adjustments are made as follows:

```swift
override func setUpWithError() throws {
    try super.setUpWithError()
    URLProtocolStub.startInterceptingRequests()
}

override func tearDownWithError() throws {
    try super.tearDownWithError()
    URLProtocolStub.stopInterceptingRequests()
}
```

Now let's shift to the `test_getFromURL_performsGETRequestWithURL`. The testing process will be like 
(1) Setup the request details(GET request).
(2) Setup a expectation instance that waits for asynchronous execution.
(3) Inject a closure that returns a URLRequest to `URLProtocolStub.observeRequests`. Verify the URL and httMethod inside this closure and fulfill the expectation to end the test.
(4) Fire the request.
(5) Wait for completion.

```swift
func test_getFromURL_performsGETRequestWithURL() {
    let url = URL(string: "http://any-url.com")!
    var request = URLRequest(url: url)
    request.httpMethod = "GET"
    let exp = expectation(description: "Wait for request")

    URLProtocolStub.observeRequests { request in
        XCTAssertEqual(request.url, url)
        XCTAssertEqual(request.httpMethod, "GET")
        exp.fulfill()
    }
   
    makeSUT().load(request: request) { _ in }
        
    wait(for: [exp], timeout: 1.0)
}
```

Run the test and you will see it pass. Our `URLProtocolStub` does perform well behind the scene. 

Now we want to test more scenarios, such as data, error, and url response. How can we do that? 
Similar to what we do for the request callback, we add other properties and functions inside `URLProtocolStub` to store information. Let's add a private structure `Stub` comprised of information we care about, and add a static function as a interface for storage.

```swift
private class URLProtocolStub: URLProtocol {
    private static var stub: Stub?
    private static var requestObserver: ((URLRequest) -> Void)?

    private struct Stub {
        let data: Data?
        let response: URLResponse?
        let error: Error?
    }

    //...
        
    static func stub(data: Data?, response: URLResponse?, error: Error?) {
        stub = Stub(data: data, response: response, error: error)
    }
}
```

Then we add some code to the `startLoading` function to check these info iteratively.

```swift
override func startLoading() {
            
    if let requestObserver = URLProtocolStub.requestObserver {
        client?.urlProtocolDidFinishLoading(self)
        return requestObserver(request)
    }
            
    if let data = URLProtocolStub.stub?.data {
        client?.urlProtocol(self, didLoad: data)
    }
            
    if let response = URLProtocolStub.stub?.response {
        client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)
    }
            
    if let error = URLProtocolStub.stub?.error {
        client?.urlProtocol(self, didFailWithError: error)
    }
            
    client?.urlProtocolDidFinishLoading(self)
}

```
We walk through the property inside the stub and invoke delegate methods in response to each situation.
Moving back to the test file, we write our second test to examine if it returns an error on a failing request.

```swift
func test_getFromURL_failsOnRequestError() {
            
    let requestError = NSError(domain: "any error", code: 1)
        
    URLProtocolStub.stub(data: nil, response: nil, error: requestError)
    let sut = makeSUT()
    let exp = expectation(description: "Wait for completion")

    sut.load(request: anyGetRequest()) { result in
        switch result {
            case .failure(let receivedError):
                XCTAssertNotNil(receivedError)
            default:
                XCTFail("Expected failure, got \(result) instead")
        }
        exp.fulfill()
    }
            
    wait(for: [exp], timeout: 1.0)
}
```
Of course, we can further wrap up the process above to improve its reusability. We encapsulate the logic into a function `resultFor`. According to the result type, we tailor two more functions to address success and failure cases.

```swift
private func resultFor(data: Data?, response: URLResponse?, error: Error?) -> APIClient.Result {
    URLProtocolStub.stub(data: data, response: response, error: error)
    let sut = makeSUT(file: file, line: line)
    let exp = expectation(description: "Wait for completion")
        
    var receivedResult: APIClient.Result!
    sut.load(request: anyGetRequest()) { result in
        receivedResult = result
        exp.fulfill()
    }
        
    wait(for: [exp], timeout: 1.0)
    return receivedResult
}

// For success cases
private func resultValuesFor(data: Data?, response: URLResponse?, error: Error?) -> (data: Data, response: HTTPURLResponse)? {
    let result = resultFor(data: data, response: response, error: error)
        
    switch result {
        case let .success((data, response)):
            return (data, response)
        default:
            XCTFail("Expected success, got \(result) instead")
            return nil
    }
}

// For failure cases   
private func resultErrorFor(data: Data?, response: URLResponse?, error: Error?) -> Error? {
        
let result = resultFor(data: data, response: response, error: error)
        
switch result {
    case let .failure(error):
        return error
    default:
        XCTFail("Expected failure, got \(result) instead")
        return nil
    }
}
```
By doing so, we are capable of testing different types of responding cases. The former test will morph into the following code:

```swift
func test_getFromURL_failsOnRequestError() {
        
    let requestError = NSError(domain: "any error", code: 1)
    let receivedError = resultErrorFor(data: nil, response: nil, error: requestError)
        
    XCTAssertNotNil(receivedError)
}
```
For more test cases and complete URLProtocolStub code, you can go to my GitHub gists to check them out.

That's it! If you have any questions or recommendations, please leave a comment down below. See you at the top! 🧪