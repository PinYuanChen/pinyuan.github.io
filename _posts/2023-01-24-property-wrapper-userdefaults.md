---
layout: post
title: 'Using A Property Wrapper to Store Codable Object into UserDefaults'
categories: [iOS dev]
tag: [swift, property-wrapper, storage]
---

Swift 5.1 rolls out the property wrapper feature, which allows us encapsulate logic associated to properties into annotations. By adding the annotation you made in front of a property, you can apply the logic directly to the property, reducing boilerplate and improve reusability. In this article, I am going to share a use case of property wrapper to deal with UserDefaults storage.

Let's say we have a struct representing an user's data, and it conforms to `Codable`.
```swift
struct User: Codable {
    var firstName: String
    var lastName: String
}
```

Before using a property wrapper, if we need to store/read this struct into/from UserDefaults, we have to create JSONEncoder/JSONDecoder and handle the process. Of course you can avoid this repetitive pattern by putting them into computed property like this:

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

But we can make it more reusable by defining a property wrapper to deal with the same scenario. The following code is how we put the logic into a wrapper called `UserDefaultsWrapper`:

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

We define a generic `T`, the object we want to store into UserDefaults, which conforms to `Codable`. At initialization, we pass in the key and default value so when the value is empty in UserDefaults, we can get the default value we define. The usage will be like this:

```swift
class UserData {
    @UserDefaultsWrapper(key: "User", defaultValue: User(firstName: "", lastName: "")) var user: User
}

let userData = UserData()
print(userData.user) // User(firstName: "", lastName: "")
userData.user = User(firstName: "John", lastName: "Wick")
print(userData.user) // User(firstName: "John", lastName: "Wick")
```
We put the `@UserDefaultsWrapper` annotation before the declaration of the variable. So whenever we access the `userData.user`, it fetches data from UserDefaults with the key "User". And when you assign a new value to the variable, it automatically save the new value into UserDefaults, which is pretty intuitive and helpful.

We might want our wrapper to be more flexible on dealing with `nil`. You can modify the code in this way:

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
By changing the `wrappedValue` and the `defaultValue` into optionals, we can get `nil` when there is nothing inside. Inside the `wrappedValue`'s setter, we use `switch` to check if the newValue is equal to `nil`, and if so, we remove the object.
And since we assign `nil` to default value at initialization, we don't have to give a default value anymore when initializing the property.

```swift
class UserData {
    @UserDefaultsWrapper(key: "User") var user: User?
}
```

Someone may not pleased to see the `wrappedValue` and the `defaultValue` set to optionals. John Sundell offers an [alternative](https://www.swiftbysundell.com/articles/property-wrappers-in-swift/) to suit your needs. You can specify the `T` to the type conforming to the `ExpressibleByNilLiteral` protocol, which means the same the `T` is the `Optional` type, since only the Optional type conforms to this protocol.

```swift
extension UserDefaultsWrapper where T: ExpressibleByNilLiteral {
    init(key: String) {
        self.init(key: key, defaultValue: nil)
    }
}
```

There is one more thing you need to take care of. You have to avoid crash when assign `nil` to the property. Sundell uses a protocol to check if the value is `nil` before setting the value.

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

The property wrapper feature allows us to do lots of customization to properties' attached behaviors while we can access them like normal unwrapped properties at the same time. I believe the property wrapper will become an important feature since we can see it heavily used in `SwiftUI` and `Combine`. However, we should avoid abusing it by hiding too much complicated logic inside for others might find it obscure to understand what it have done underneath the hood. You also have to watch out side effects such as race condition when using it. 

That's it! If you have any questions or recommendations, please leave a comment down below. See you at the top! 👹