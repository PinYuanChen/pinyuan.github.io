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

This post is going to cover how to write unit tests for requests without actually firing networking task. I will probably write a post about end-to-end tests in the future, but right now let's just focus on the unit test topic.

The following is a typical API protocol that defines the result and the load function.
```swift
public protocol APIClient {
    typealias Result = Swift.Result<(Data, HTTPURLResponse), Error>
    
    func load(request: URLRequest, completion: @escaping (Result) -> Void)
}
```

We create a class named `URLSessionAPIClient` and make it conform to `APIClient` protocol.
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
It holds a `URLSession` property and use it to make requests. You can either inject a  `URLSession` instance or use the default `shared` one. Then we move on to create a new test file called `URLSessionAPIClientTests`. We adopt the factory method mentioned [before](https://pinyuanchen.github.io/posts/xctestcase-sut/) to create instances for our tests. 

```swift
final class URLSessionAPIClientTests: XCTestCase {
    // helpers
    private func makeSUT() -> APIClient {
        let sut = URLSessionAPIClient()
        return sut
    }
}
```

Time to write some tests. We want to verify the behaviors of `URLSessionAPIClient` align with our expectations. First, we want to test it can correctly make a GET request by checking its url and http method.

```swift
func test_getFromURL_performsGETRequestWithURL() { }    
```

Here comes a problem: how can we make a request using a fake url? We must utilize something to intercept the request and return the response as what we want. And here comes a handy tool called `URLProtocol`, which fulfills what we need.
Don't be fooled by its name. `URLProtocol` is actually a `class` that exists in iOS's URL loading system. When we fire a url session request, the system automatically creates a `URLProtocol` instance to handle the task. All we have to do is to stub `URLProtocol` and intercept the request by implementing required methods.

Upon sub classing `URLProtocol`, we need to implement four required methods.
>
class func canInit(with request: URLRequest)
class func canonicalRequest(for request: URLRequest) -> URLRequest
func startLoading()
func stopLoading()
>

The core section is `startLoading`, at this step we will check if we have anything stubbed beforehand and return them as responses. That is, we intercept the request here.  We implement the four requirements like the following:

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

For `canInit`, we simply return `true` to pass the validation of request eligibility, and so does the `canonicalRequest`. Since we don't need to do anything on `stopLoading`, we just leave it empty. Now we are gonna focus on `startLoading` method. We need a property to store the request and verify it later. We can simply use a closure property to achieve that.

```swift
private class URLProtocolStub: URLProtocol {
    private static var requestObserver: ((URLRequest) -> Void)?

    static func observeRequests(observer: @escaping (URLRequest) -> Void) {
        requestObserver = observer
    }
}
```
Since the URL loading system creates the `URLProtocol` instance underneath the hood, we cannot control its creation so we declare `requestObserver` as a static property. We also setup a static function `observeRequests` to provide a interface for outsiders to store the property. Then in the `startLoading` function we add the code:

```swift
override func startLoading() {
    if let requestObserver = URLProtocolStub.requestObserver {
        client?.urlProtocolDidFinishLoading(self)
        return requestObserver(request)
    }
    client?.urlProtocolDidFinishLoading(self)
}

```
The code detect if the request observer exists, then it finish the loading immediately and evoke the callback. 

Let's move back to the test file. Before you begin testing, there is one more key step to settle down the URLProtocol. In `setupWithError`, we need to register our `URLProtocolStub` to make it works.

```swift
override func setUpWithError() throws {
    try super.setUpWithError()
    URLProtocol.registerClass(URLProtocolStub.self)
}
```

Besides, in `tearDownWithError` we need to unregister it and also release the observer closure. 

```swift
override func tearDownWithError() throws {
    try super.tearDownWithError()
    URLProtocol.unregisterClass(URLProtocolStub.self)
    requestObserver = nil
}
```

To make them more readable, we wrap them up into two static functions, identifying their usages, in `URLProtocolStub`.

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
In the test file, we modify the calling as below:

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
Moving back to the test file, we write our second test to examine it returns an error on a failing request.

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


```swift
private class URLProtocolStub: URLProtocol {
        private static var stub: Stub?
        private static var requestObserver: ((URLRequest) -> Void)?
        
        private struct Stub {
            let data: Data?
            let response: URLResponse?
            let error: Error?
        }
        
        static func observeRequests(observer: @escaping (URLRequest) -> Void) {
            requestObserver = observer
        }
        
        static func stub(data: Data?, response: URLResponse?, error: Error?) {
            stub = Stub(data: data, response: response, error: error)
        }
        
        static func startInterceptingRequests() {
            URLProtocol.registerClass(URLProtocolStub.self)
        }
        
        static func stopInterceptingRequests() {
            URLProtocol.unregisterClass(URLProtocolStub.self)
            requestObserver = nil
            stub = nil
        }
        
        override class func canInit(with request: URLRequest) -> Bool {
            return true
        }
        
        override class func canonicalRequest(for request: URLRequest) -> URLRequest {
            return request
        }
        
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
        
        override func stopLoading() {}
    }
    ```

    ```swift
    
    private func resultFor(data: Data?, response: URLResponse?, error: Error?) -> HTTPClient.Result {
        
        URLProtocolStub.stub(data: data, response: response, error: error)
        let sut = makeSUT()
        let exp = expectation(description: "Wait for completion")
        
        var receivedResult: HTTPClient.Result!
        sut.load(request: anyGetRequest()) { result in
            receivedResult = result
            exp.fulfill()
        }
        
        wait(for: [exp], timeout: 1.0)
        return receivedResult
    }
    
    private func anyGetRequest() -> URLRequest {
        return URLRequest(url: .init(string: "http://any-url.com")!)
    }
    ```

    test
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
    
    func test_getFromURL_failsOnRequestError() {
        
        let requestError = NSError(domain: "any error", code: 1)
        let receivedError = resultFor(data: nil, response: nil, error: requestError)
        
        XCTAssertNotNil(receivedError)
    }
    ```