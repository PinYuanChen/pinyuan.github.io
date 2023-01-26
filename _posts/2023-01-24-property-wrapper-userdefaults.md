---
layout: post
title: 'Using A Property Wrapper to Store Codable Object into UserDefaults'
categories: [iOS dev]
tag: [swift, property-wrapper, storage]
---

Swift 5.1 introduces the property wrapper feature, which allows for encapsulating logic associated with properties into annotations. By adding the annotation in front of a property, you can apply the logic directly to the property, reducing boilerplate and improving reusability. In this article, I will share a use case for property wrappers in dealing with UserDefaults storage.

 For example, let's consider a struct representing a user's data that conforms to  `Codable`.

```swift
struct User: Codable {
    var firstName: String
    var lastName: String
}
```

Before using property wrappers, if we needed to store or read this struct in or from UserDefaults, we would have to create JSONEncoder/JSONDecoder and handle the process ourselves. To avoid this repetitive pattern, we could put this logic into a computed property like so:

```swift
var currentUser: User {
    get {
        // read from UserDefaults
    }
    set {
        // store into UserDefaults
    }
}
```

But with property wrappers, we can make this logic more reusable by defining a wrapper specifically for this scenario. The following code shows how we can put this logic into a wrapper called `UserDefaultsWrapper`:

```swift
@propertyWrapper
struct UserDefaultsWrapper<T: Codable> {
    private let key: String
    private let defaultValue: T
    private let userDefaults = UserDefaults.standard
    
    init(key: String, defaultValue: T) {
        self.key = key
        self.defaultValue = defaultValue
    }
    
    var wrappedValue: T {
        get {
            guard let data = userDefaults.object(forKey: key) as? Data,
                  let value = try? JSONDecoder().decode(T.self, from: data) else {
                return defaultValue
            }
            
            return value
        }
        set {
            let data = try? JSONEncoder().encode(newValue)
            userDefaults.set(data, forKey: key)
        }
    }
}
```

We define a generic `T`, which represents the object we want to store in UserDefaults and conforms to `Codable`. At initialization, we pass in the key and a default value, so when the value is empty in UserDefaults, we can get the default value we defined. The usage will be like this:

```swift
class UserData {
    @UserDefaultsWrapper(key: "User", defaultValue: User(firstName: "", lastName: "")) var user: User
}

let userData = UserData()
print(userData.user) // User(firstName: "", lastName: "")
userData.user = User(firstName: "John", lastName: "Wick")
print(userData.user) // User(firstName: "John", lastName: "Wick")
```
We put the `@UserDefaultsWrapper` annotation before the declaration of the variable. So whenever we access userData.user, it fetches data from UserDefaults with the key "User". And when you assign a new value to the variable, it automatically saves the new value into UserDefaults, which is pretty intuitive and helpful.

To make the wrapper more flexible in dealing with `nil`, you can modify the code in the following way:

```swift
@propertyWrapper
struct UserDefaultsWrapper<T: Codable> {
    ...
    private var defaultValue: T?
    ...

    init(key: String, defaultValue: T? = nil) {
        self.key = key
        self.defaultValue = defaultValue
    }
    
    var wrappedValue: T? {
        get {
            guard let data = userDefaults.object(forKey: key) as? Data,
                  let value = try? JSONDecoder().decode(T.self, from: data) else {
                return defaultValue
            }
            
            return value
        }
        set {
            switch newValue {
            case .none:
                userDefaults.removeObject(forKey: key)
            default:
                let data = try? JSONEncoder().encode(newValue)
                userDefaults.set(data, forKey: key)
            }
        }
    }
}
```
By changing the `wrappedValue` and the `defaultValue` into optionals, we can get `nil` when there is nothing stored in UserDefaults. Inside the `wrappedValue`'s setter, we use a `switch` statement to check if the `newValue` is equal to `nil`, and if it is, we remove the object from UserDefaults.
And since we assign `nil` to default value at initialization, we don't have to give a default value anymore when initializing the property.

```swift
class UserData {
    @UserDefaultsWrapper(key: "User") var user: User?
}
```

Someone may not be pleased to see the `wrappedValue` and the `defaultValue` set to optionals. John Sundell offers an [alternative](https://www.swiftbysundell.com/articles/property-wrappers-in-swift/) to suit your needs. You can specify `T` as a type that conforms to the `ExpressibleByNilLiteral` protocol, which means that `T` is the same as an `Optional` type, since only the Optional type conforms to this protocol.

```swift
extension UserDefaultsWrapper where T: ExpressibleByNilLiteral {
    init(key: String) {
        self.init(key: key, defaultValue: nil)
    }
}
```

One more thing to keep in mind is to avoid a crash when assigning `nil` to the property. Sundell uses a protocol to check if the value is nil before setting it.

```swift
private protocol AnyOptional {
    var isNil: Bool { get }
}

extension Optional: AnyOptional {
    var isNil: Bool { self == nil }
}

@propertyWrapper
struct UserDefaultsWrapper<T: Codable> {
    ...
    private var defaultValue: T
    ...
    var wrappedValue: T {
        get {
            // getter
        }
        set {
            if let optional = newValue as? AnyOptional,
               optional.isNil {
                userDefaults.removeObject(forKey: key)
            } else {
                let data = try? JSONEncoder().encode(newValue)
                userDefaults.set(data, forKey: key)
            }
        }
    }
}
```

In this way, you can get and set `nil` from the property without adding optional marks to the `wrappedValue` and the `defaultValue`.


The property wrapper feature allows us to do a lot of customization to properties' attached behaviors while still being able to access them like normal unwrapped properties. I believe the property wrapper will become an important feature, since we can see it being heavily used in SwiftUI and Combine. However, we should avoid abusing it by hiding too much complicated logic inside, as others might find it difficult to understand what it's doing under the hood. Besides, be mindful of side effects such as race conditions when using it.

That's it! If you have any questions or recommendations, please leave a comment down below. See you at the top! 👹