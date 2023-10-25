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

Time to write some tests. We want to verify the behaviors of `URLSessionAPIClient` align with our expectations. First, we want to test it can correctly make a GET request by checking its url and http method. So we create another helper function to provide us a GET request.

```swift
private func anyGetRequest() -> URLRequest {
    return URLRequest(url: .init(string: "http://any-url.com")!)
}
```
Then here comes a problem: how can we make a request using a fake url? We must utilize something to intercept the request and return the response as what we want. And here comes a handy tool called `URLProtocol`, which fulfills what we need.
Don't be fooled by its name. `URLProtocol` is actually a `class` that exists in iOS's URL loading system. When we fire a url session request, the system automatically creates a `URLProtocol` instance to handle the task. All we have to do is to stub `URLProtocol` and intercept the request by implementing required methods.

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
    final class URLSessionAPIClientTests: XCTestCase {

        override func setUpWithError() throws {
            try super.setUpWithError()
            URLProtocolStub.startInterceptingRequests()
        }

        override func tearDownWithError() throws {
            try super.tearDownWithError()
            URLProtocolStub.stopInterceptingRequests()
        }
    }
    ```

    helpers

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