---
layout: post
title: 'Using A Property Wrapper to Store Codable Object into UserDefaults'
categories: [iOS dev]
tag: [swift, property-wrapper, storage]
---



```swift
struct User: Codable {
    var firstName: String
    var lastName: String
}
```

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
            guard let data = userDefaults.object(forKey: key) as? Data else {
                return defaultValue
            }

            let value = try? JSONDecoder().decode(T.self, from: data)
            return value ?? defaultValue
        }
        set {
            let data = try? JSONEncoder().encode(newValue)
            userDefaults.set(data, forKey: key)
        }
    }
}
```

```swift
class UserData {
    @UserDefaultsWrapper(key: "User", defaultValue: User(firstName: "", lastName: "")) var user: User
}

let userData = UserData()
userData.user = User(firstName: "John", lastName: "Wick")
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