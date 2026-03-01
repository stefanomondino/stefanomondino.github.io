---
title: "Protocol values in Codable objects"
date: 2025-07-11
draft: false
tags: ["Codable", "Architecture", "Swift"]
description: "Dealing with mixed-type arrays in Swift Decodable is quite hard - here's an advanced trick that will lead you to a scalable and future-proof solution"
image: "images/protocol-values-in-codable-objects.png"
---

When dealing with JSON APIs, it's quite common to encounter complex properties—both single objects and arrays of objects. As discussed in a previous post about enum alternatives, the typical strategy involves creating Swift objects conforming to `Decodable` with a property for every JSON value.

For more complex objects, developers declare properties with `Decodable` types (for single values) or arrays of `Decodable` objects (for lists). However, this approach breaks down when JSON objects need to conform to a protocol rather than a concrete type—a scenario that requires a more sophisticated solution.

## The Problem: A List of Animals

Consider a veterinary hospital app that fetches a list of animal patients. Each animal has a `type` property and a `name`, but other properties vary by animal type:

```json
{
  "items": [
    {
      "type": "dog",
      "name": "Bingo",
      "goodBoy": true,
      "weight": 10.0
    },
    {
      "type": "cat",
      "name": "Mr. Fluffy",
      "breed": "ragdoll",
      "age": 4
    }
  ]
}
```

Declaring protocol-based arrays in `Decodable` objects fails:

```swift
struct Response: Decodable {
    let items: [any Animal] // This will not compile
}
```

The `Decodable` system requires knowing concrete types in advance. Without explicitly declaring which types exist and where they're located, the system cannot map JSON to protocol-conforming values—a critical limitation in modular applications where animal types might scatter across different feature modules.

## The Core Mapping Strategy

Solving this requires:

1. **Extracting a discriminator property** from the current JSON element to determine its type
2. **Iterating over array elements** individually rather than decoding the entire array at once
3. **Dynamically selecting the appropriate concrete type** based on the extracted discriminator
4. **Accumulating results** into an array of protocol-conforming values

A raw implementation demonstrates the complexity:

```swift
struct Response: Decodable {
    private enum CodingKeys: String, CodingKey {
        case items
    }
    let items: [any Animal]
    init(from decoder: any Decoder) throws {
        struct Extractor: Codable {
            let type: AnimalType
        }
        let container = try decoder.container(keyedBy: CodingKeys.self)
        var subContainer = try container.nestedUnkeyedContainer(forKey: .items)
        var items: [any Animal] = []
        while subContainer.isAtEnd == false {
            var subcontainerCopy = subContainer
            let extractor = try subContainer.decode(Extractor.self)
            switch extractor.type {
            case .cat: items.append(try subcontainerCopy.decode(Cat.self))
            case .dog: items.append(try subcontainerCopy.decode(Dog.self))
            default: break
            }
        }
        self.items = items
    }
}
```

**Key complications:**

- Cyclomatic complexity increases with each new animal type
- Requires knowing all concrete types in the Response context—problematic for modular architectures
- The switch statement must be duplicated for single values, optional values, and arrays
- Logic isn't reusable across different polymorphic scenarios

## The Solution: Decodable Polymorphism

A standardized approach requires three components:

1. **`Polymorphic` protocol**: Declares that a type participates in polymorphic decoding
2. **`TypeExtractor` protocol**: Implements the logic to identify and extract the correct concrete type
3. **`@Polymorph` property wrapper**: Simplifies declaring polymorphic properties in `Decodable` objects

### The Polymorphic Protocol

```swift
public protocol Polymorphic: Decodable, Sendable {
    associatedtype Extractor: TypeExtractor
    static var typeExtractor: Extractor { get }
}
```

### The TypeExtractor Protocol

```swift
public protocol TypeExtractor: Decodable, Equatable, Sendable {
    associatedtype ObjectType: Sendable
    func itemType(from availableTypes: [any Polymorphic.Type]) -> (any Polymorphic.Type)?
    func extract(from container: SingleValueDecodingContainer,
                 availableTypes: [any Polymorphic.Type]) throws -> ObjectType?
    func extract(from container: inout UnkeyedDecodingContainer,
                 availableTypes: [any Polymorphic.Type]) throws -> ObjectType?
    static func extract(from decoder: any Decoder) throws -> ObjectType?
    static func extract(from decoder: any Decoder) throws -> [ObjectType]?
}
```

Protocol extensions provide default implementations for the extraction methods, eliminating boilerplate:

```swift
extension TypeExtractor {
    func itemType(from availableTypes: [any Polymorphic.Type]) -> (any Polymorphic.Type)? {
        availableTypes.first(where: { $0.typeExtractor as? Self == self })
    }

    func extract(from container: SingleValueDecodingContainer,
                 availableTypes: [any Polymorphic.Type]) throws -> ObjectType? {
        if let type = itemType(from: availableTypes) {
            return try container.decode(type) as? ObjectType
        } else {
            return nil
        }
    }

    func extract(from container: inout UnkeyedDecodingContainer,
                 availableTypes: [any Polymorphic.Type]) throws -> ObjectType? {
        if let type = itemType(from: availableTypes) {
            return try container.decode(type) as? ObjectType
        } else {
            return nil
        }
    }

    static func extract(from decoder: any Decoder) throws -> ObjectType? {
        let container = try decoder.singleValueContainer()
        let extractor = try container.decode(Self.self)

        if let object = try extractor
            .extract(from: container,
                     availableTypes: decoder.availableTypes(for: extractor)) {
            return object
        } else {
            return nil
        }
    }

    static func extract(from decoder: any Decoder) throws -> [ObjectType]? {
        var container = try decoder.unkeyedContainer()
        var objects: [Self.ObjectType] = []
        while !container.isAtEnd {
            var decodingCopy = container
            let extractor = try container.decode(Self.self)
            if let value = try? extractor
                .extract(from: &decodingCopy,
                         availableTypes: decoder.availableTypes(for: extractor)) {
                objects.append(value)
            }
        }
        return objects
    }
}
```

### Concrete TypeExtractor Implementation

```swift
struct AnimalType: TypeExtractor {
    enum CodingKeys: String, CodingKey {
        case value = "type"
    }

    typealias ObjectType = Animal

    var value: String

    init(_ value: String) {
        self.value = value
    }
}

extension AnimalType {
    static var dog: Self { .init("dog") }
    static var cat: Self { .init("cat") }
}
```

### Protocol Definition and Conformance

```swift
protocol Animal: Polymorphic where Extractor == AnimalType {}

struct Dog: Animal {
    static var typeExtractor: AnimalType { .dog }
    let name: String
    let goodBoy: Bool
}

struct Cat: Animal {
    static var typeExtractor: AnimalType { .cat }
    let name: String
    let breed: String
    let age: Int
}
```

### The @Polymorph Property Wrapper

```swift
@propertyWrapper
public struct Polymorph<Extractor: TypeExtractor, Value: Sendable>: Decodable, Sendable {
    private var value: Value?
    public var wrappedValue: Value {
        get {
            return value.unsafelyUnwrapped
        }
        set {
            value = newValue
        }
    }

    public init(_ value: Value) {
        self.value = value
    }

    public init(from decoder: any Decoder) throws {
        if Extractor.ObjectType.self == Value.self {
            guard let value: Extractor.ObjectType = try Extractor.extract(from: decoder) else {
                throw PolymorphError.missingMapping
            }
            self.value = value as? Value
        } else if Value.self == Optional<Extractor.ObjectType>.self {
            let value: Optional<Extractor.ObjectType> = try Extractor.extract(from: decoder)
            self.value = value as? Value
        } else if Value.self == [Extractor.ObjectType].self {
            let value: [Extractor.ObjectType]? = try Extractor.extract(from: decoder)
            self.value = value as? Value
        } else if Value.self == Optional<[Extractor.ObjectType]>.self {
            let value: [Extractor.ObjectType]? = try Extractor.extract(from: decoder)
            self.value = value as? Value
        } else {
            throw PolymorphError.misconfiguredMapping
        }
    }
}

public enum PolymorphError: Swift.Error {
    case missingMapping
    case misconfiguredMapping
}
```

### Handling Optional Values

When using optional wrapped values, the decoder needs to recognize that the property wrapper itself is always present, even if the underlying value is `nil`:

```swift
public extension KeyedDecodingContainer {
    func decode<Extractor, Value>(_ type: Polymorph<Extractor, Value>.Type,
                                   forKey key: K) throws -> Polymorph<Extractor, Value>
                                   where Value: ExpressibleByNilLiteral {
        if let value = try self.decodeIfPresent(type, forKey: key) {
            return value
        }
        return Polymorph()
    }
}
```

### Declaring Available Types

The decoder needs to know which concrete types are available for a given extractor. This is handled through the decoder's `userInfo` dictionary:

```swift
public extension Decoder {
    func availableTypes<Extractor: TypeExtractor>(for extractor: Extractor) -> [any Polymorphic.Type] {
        (userInfo[Extractor.codingReference] as? [any Polymorphic.Type]) ?? []
    }
}

public extension JSONDecoder {
    func set<Extractor: TypeExtractor>(types: [any Polymorphic.Type],
                                        for extractor: Extractor.Type = Extractor.self) {
        userInfo[Extractor.codingReference] = types
    }
}

public extension PropertyListDecoder {
    func set<Extractor: TypeExtractor>(types: [any Polymorphic.Type],
                                        for extractor: Extractor.Type = Extractor.self) {
        userInfo[Extractor.codingReference] = types
    }
}

public extension TypeExtractor {
    fileprivate static var codingReference: CodingUserInfoKey {
        .init(rawValue: "\(ObjectIdentifier(Self.self))").unsafelyUnwrapped
    }
    static func set(types: [any Polymorphic.Type], in decoder: JSONDecoder) {
        decoder.userInfo[Self.codingReference] = types
    }
}
```

## Complete Usage Example

```swift
let json = """
{
  "items": [
    {
      "type": "dog",
      "name": "Bingo",
      "goodBoy": true,
      "weight": 10.0
    },
    {
      "type": "cat",
      "name": "Mr. Fluffy",
      "breed": "ragdoll",
      "age": 4
    }
  ]
}
"""

protocol Animal: Polymorphic where Extractor == AnimalType {}

struct AnimalType: TypeExtractor {
    enum CodingKeys: String, CodingKey {
        case value = "type"
    }

    typealias ObjectType = Animal

    var value: String

    init(_ value: String) {
        self.value = value
    }
}

extension AnimalType {
    static var dog: Self { .init("dog") }
    static var cat: Self { .init("cat") }
}

struct Dog: Animal {
    static var typeExtractor: AnimalType { .dog }
    let name: String
    let goodBoy: Bool
}

struct Cat: Animal {
    static var typeExtractor: AnimalType { .cat }
    let name: String
    let breed: String
    let age: Int
}

struct Response: Decodable {
    @Polymorph<AnimalType, [any Animal]> let items
}

let data = Data(json.utf8)
let decoder = JSONDecoder()
decoder.set(types: [Dog.self, Cat.self], for: AnimalType.self)
let response = try decoder.decode(Response.self, from: data)
print(response.items)
```

## Advantages

This approach provides several benefits:

- **Eliminated boilerplate**: The switch statement and complex decoding logic moves into reusable protocol extensions
- **Modular scalability**: New animal types can be added in different modules without modifying the Response object
- **Uniform syntax**: A single `@Polymorph` wrapper handles single values, optionals, arrays, and optional arrays
- **Reusability**: The same pattern works for any polymorphic scenario—animals, chat messages, content modules, etc.
- **Future-proof**: Adding new types requires only creating a new struct conforming to the protocol and registering it with the decoder

## Conclusion

Handling polymorphic objects in Swift's `Decodable` system requires sophisticated techniques, but a well-structured abstraction eliminates complexity. By encapsulating the extraction logic in protocols and leveraging property wrappers, developers can maintain clean, scalable code while handling complex API responses.

The complete implementation is available as an open-source Swift Package Manager library called [CodableKitten](https://github.com/stefanomondino/CodableKitten), which provides ready-to-use components for polymorphic decoding in real-world applications.
