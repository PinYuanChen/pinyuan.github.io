---
layout: post
title: 'Using A Property Wrapper to Store Codable Object into UserDefaults'
categories: [iOS dev]
tag: [swift, property-wrapper, storage]
---

Swift 5.1 rolls out the property wrapper feature, which allows us encapsulate logic associated to properties into annotations. By adding the annotation you made in front of a property, you can apply the logic directly to that property, reducing boilerplate and improve reusability. In this article I am gonna share a use case of property wrapper to deal with UserDefaults storage.

Let's say we have a struct representing a user's data, and it conforms to `Codable`.
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

We define a generic `T` that conforms to `Codable` and this `T` is the object we want to store into UserDefaults. At initialization, we pass the key and default value in here, so when the value is empty in UserDefaults, we can get the default value we define. The usage will be like this:

```swift
class UserData {
    @UserDefaultsWrapper(key: "User", defaultValue: User(firstName: "", lastName: "")) var user: User
}

let userData = UserData()
print(userData.user) // User(firstName: "", lastName: "")
userData.user = User(firstName: "John", lastName: "Wick")
print(userData.user) // User(firstName: "John", lastName: "Wick")
```
We put the `@UserDefaultsWrapper` annotation before the declaration of the variable. So whenever we get the value from the `userData.user`, it fetches data from UserDefaults with the key "User". And when you assign a new value to it, it automatically save it into UserDefaults. That's pretty intuitive and helpful. The property wrapper do the retrieving/saving stuff underneath the hood.

We might want our wrapper to be more flexible to deal with `nil`. You can modify the code in this way:

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


```swift
extension UserDefaultsWrapper where T: ExpressibleByNilLiteral {
    init(key: String) {
        self.init(key: key, defaultValue: nil)
    }
}

class UserData {
    @UserDefaultsWrapper(key: "User") var user: User?
}
```

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