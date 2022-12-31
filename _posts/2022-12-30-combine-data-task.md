---
layout: post
title: 'Combine: Handle dataTaskPublisher'
categories: [iOS dev]
tag: [swift, combine]
---

Recently I was playing with Combine and I kind of liked its syntax. Combine offers plenty handy operators to handle asynchronicity, making the processing of URL task results looks elegant. If you are new to Combine, I highly recommend you to read the [official document](https://developer.apple.com/documentation/foundation/urlsession/processing_url_session_data_task_results_with_combine) about processing URL task results. It covers many useful APIs and explains them in a succinct way. URLSession offers a Combine publisher called `dataTaskPublisher(for: URL)` to handle asynchronous data-fetching from the internet. I want to share some ideas of processing results of `dataTaskPublisher` using a chain of operators.

## Convert the Contents via tryMap
First, let's take a look at a typical dataTaskPublisher:
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
We put a url into publisher. At default, it will switch to a global background queue to process this time-consuming task. And after the process completes, the publisher will emits either a tuple containing data and a `URLResponse`, or an error. Then we can switch to main queue to receive the outcome and use `eraseToAnyPublisher` to convert it to an `AnyPublisher` output. You can test if you remove the `eraseToAnyPublisher` what will the output look like. Actually, it will turn out to be a nested publisher structure which is pretty difficult to deal with. So adding `eraseToAnyPublisher` can save us lots of efforts.

```swift
static func handleURLResponse(output: URLSession.DataTaskPublisher.Output) throws -> Data {
    guard let response = output.response as? HTTPURLResponse,
              response.statusCode >= 200 && response.statusCode < 300 else {
        throw URLError(.badServerResponse)
    }
    return output.data
}
```

```swift
static func download(url: URL) -> AnyPublisher<Data, Error> {
    return URLSession.shared.dataTaskPublisher(for: url)
            .tryMap { try handleURLResponse(output: $0) }
            .receive(on: DispatchQueue.main)
            .eraseToAnyPublisher()
}
```
## Decode and Handle Received Values
 Then we can call the download function and subscribe it as follows:

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


```swift
subscription = NetworkManager
            .download(url: url)
            .decode(type: [Model].self, decoder: JSONDecoder())
            .sink(receiveCompletion: NetworkManager.handleCompletion, receiveValue: { [weak self] result in
                guard let self = self else { return }
                self.outputResult = result
                self.subscription?.cancel()
            })
```
## Background Optimization

```swift
static func download<T: Codable>(url: URL, type: T.Type) -> AnyPublisher<T, Error> {
    return URLSession.shared.dataTaskPublisher(for: url)
            .tryMap { try handleURLResponse(output: $0, url: url) }
            .decode(type: T.self, decoder: JSONDecoder())
            .receive(on: DispatchQueue.main)
            .eraseToAnyPublisher()
}
```
