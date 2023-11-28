---
layout: post
title: 'Combine: Handle dataTaskPublisher Results'
categories: [iOS dev]
tag: [swift, combine]
---

Recently I was playing with Combine and I kind of liked its succinct syntax. Combine offers plenty of handy operators to handle asynchronicity, making the processing of URL task results looks elegant. If you are new to Combine, I highly recommend you read the [official document](https://developer.apple.com/documentation/foundation/urlsession/processing_url_session_data_task_results_with_combine) about processing URL task results. It covers many helpful APIs and you can quickly get the gist of how to use Combine.

In the past, we use URLSession to make requests and do all our work in a handler closure, which is a little bit messy. Now URLSession offers a Combine publisher called `dataTaskPublisher(for: URL)` to handle asynchronous data fetching from the internet. On top of that, you can use many operators to make your code looks organized and neat. I want to share some ideas for processing the results of `dataTaskPublisher` using a chain of operators.

## Convert the Contents via tryMap
First, let's take a look at a typical `dataTaskPublisher`:

```swift
class NetworkManager {
    static func download(url: URL) -> AnyPublisher<Data, Error> {
        return URLSession.shared.dataTaskPublisher(for: url)
            .tryMap { output -> Data in
                guard let response = output.response as? HTTPURLResponse,
                      response.statusCode >= 200 && response.statusCode < 300 else {
                    throw URLError(.badServerResponse)
                }
                
                return output.data
            }
            .receive(on: DispatchQueue.main)
            .eraseToAnyPublisher()
    }
}
```
We declare a `NetworkManager` and a static download function to handle requests. Inside the function, we put a url into a `dataTaskPublisher`. At default, it will process the task in a global background queue. And after completing the process, the publisher will emit either a tuple containing raw data and a `URLResponse`, or an error.

At this point, you can either use map(_:) or tryMap(_:) operators to handle the emission. The official document explains their differences: 

>
You can use the `map(_:)` operator to convert the contents of this tuple to another type. If you want to inspect the response before inspecting the data, use `tryMap(_:)` and throw an error if the response is unacceptable.
>

I use `tryMap(:)` and expect if there is any response error I can catch them properly. Some native iOS developers may confuse `map(_:)` here with the array's function `map`. `Map` in Combine is similar to RxSwift that they both convert data into your designated types rather than return an array. You can see the API underneath the hood looks like this: 

```swift
public func map<T>(_ transform: @escaping (Self.Output) -> T) -> Publishers.Map<Self, T>
```

while the definition of `Swift.Collection.Array` is:

```swift
@inlinable public func map<T>(_ transform: (Element) throws -> T) rethrows -> [T]
```

 Let's move back to our example. After mapping the data, we switch to the main queue to receive the outcome and use `eraseToAnyPublisher` to convert it into an `AnyPublisher` output. You can test if you remove the `eraseToAnyPublisher`. Then you will see the output turn out to be a gross nested publisher which is pretty difficult to deal with. So adding `eraseToAnyPublisher` can save us lots of effort.

 We can make our code more readable and reusable by getting the function in `tryMap(_:)` outside. We define a static function called `handleURLResponse` which does the same job as what we formerly did in `tryMap(_:)`.

```swift
static func handleURLResponse(output: URLSession.DataTaskPublisher.Output) throws -> Data {
    guard let response = output.response as? HTTPURLResponse,
              response.statusCode >= 200 && response.statusCode < 300 else {
        throw URLError(.badServerResponse)
    }
    return output.data
}
```

then you can replace the closure in `tryMap(_:)` with our defined function.

```swift
static func download(url: URL) -> AnyPublisher<Data, Error> {
    return URLSession.shared.dataTaskPublisher(for: url)
            .tryMap { try handleURLResponse(output: $0) }
            .receive(on: DispatchQueue.main)
            .eraseToAnyPublisher()
}
```

## Decode and Handle Received Values
 Let's create a `DataService` class to fetch data through NetworkManager.

```swift
class DataService {
    
    @Published var outputResult = [Model]()
    var subscription: AnyCancellable?
    
    init() {
        fetchData()
    }
    
    func fetchData() {
        
        guard let url = URL(string: "Put in your URL string") else { return }
        
        subscription = NetworkManager
            .download(url: url)
            .decode(type: [Model].self, decoder: JSONDecoder())
            .sink { completion in
                switch completion {
                case .finished:
                    break
                case .failure(let error):
                    print(error.localizedDescription)
                }
            } receiveValue: { [weak self] result in
                guard let self = self else { return }
                self.outputResult = result
                self.subscription?.cancel()
            }
    }
}
```

Normally, we will define a `struct` model to contain the API returned data, usually, it is a JSON or a XML file. You can use `decode(type:decoder:)` to do the job. It allows us to convert the data to our own types whichever conforms to the `Decodable` protocol. Then we hand the result to the `sink` operator. 

Combine provides us with two types of sink operators: `sink(receiveValue:)` and `sink(receiveCompletion:receiveValue:)`. The latter one has one more completion block. You can capture the error in its failure case or just break it when it is finished. That looks a little bit verbose. So we can also define a function outside to handle it.

```swift
static func handleCompletion(completion: Subscribers.Completion<Error>) {
    switch completion {
        case .finished:
            break
        case .failure(let error):
            print(error.localizedDescription)
    }
}
```

that makes our code looks neat

```swift
subscription = NetworkManager
            .download(url: url)
            .decode(type: [Model].self, decoder: JSONDecoder())
            .sink(receiveCompletion: NetworkManager.handleCompletion,
                  receiveValue: { [weak self] result in
                guard let self = self else { return }
                self.outputResult = result
                self.subscription?.cancel()
            })
```
## Background Optimization
Our `NetworkManager.download(url:)` function switches from the background to the main queue and sinks the data to the decoder. Here we can do a little optimization by putting the time-consuming decoding job in the background. We can define a generic which conforms to Codable and do the decoding inside the download function:

```swift
static func download<T: Codable>(url: URL, type: T.Type) -> AnyPublisher<T, Error> {
    return URLSession.shared.dataTaskPublisher(for: url)
            .tryMap { try handleURLResponse(output: $0, url: url) }
            .decode(type: T.self, decoder: JSONDecoder())
            .receive(on: DispatchQueue.main)
            .eraseToAnyPublisher()
}
```

Whenever you need to use it, just pass in the model type you've already defined:

```swift
subscription = NetworkManager
            .download(url: url, type: [Model].self)
            .sink(receiveCompletion: NetworkManager.handleCompletion,
                  receiveValue: { [weak self] result in
                // the same
            })
```
