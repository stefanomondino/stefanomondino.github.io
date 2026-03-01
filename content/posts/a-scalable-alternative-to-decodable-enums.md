---
title: "A scalable alternative to Decodable enums"
date: 2025-04-23
draft: false
tags: ["Codable", "Swift", "Architecture"]
description: "Discover a scalable approach to decoding enums in Swift. Cleaner code, better flexibility, and improved maintainability for growing codebases."
image: "images/a-scalable-alternative-to-decodable-enums.png"
---

Swift's `Decodable` and `Encodable` protocols provide convenient ways to map JSON data into native Swift objects. However, basic enum-based approaches can become problematic as projects evolve and APIs introduce new values.

## The Problem with Standard Decodable Enums

When mapping API responses with enums, new undeclared values cause complete decoding failures. For instance, if a conference app maps event categories as a `String`-based enum:

```swift
enum Category: String, Codable {
    case talk
    case workshop
}
```

Adding a new "roundtable" category breaks the entire decoding process, failing even for valid JSON containing that new value.

## Traditional Workaround Limitations

The common solution adds an `unknown` case:

```swift
enum Category: String, Codable {
    case talk
    case workshop
    case unknown

    init(from decoder: Decoder) throws {
        let rawValue = try decoder.singleValueContainer().decode(String.self)
        self = Category(rawValue: rawValue) ?? .unknown
    }
}
```

This approach has three critical drawbacks:

1. **Data Loss**: Original values are discarded, complicating debugging and server communication
2. **Code Duplication**: Every enum requires custom decoding logic
3. **Scalability Issues**: Enums cannot be extended across different modules in modular architectures

## A Struct-Based Alternative

Instead of enums, use structs that preserve the original value:

```swift
extension Event {
    public struct Category: Codable {
        public let value: String

        public init(from decoder: Decoder) throws {
            value = try decoder.singleValueContainer().decode(String.self)
        }

        public func encode(to encoder: Encoder) throws {
            var container = encoder.singleValueContainer()
            try container.encode(value)
        }
    }
}

public extension Event.Category {
    static var talk: Self { .init(value: "talk") }
    static var workshop: Self { .init(value: "workshop") }
}
```

## Building a Reusable Generic Solution

Create a generic `ExtensibleIdentifier` type:

```swift
public struct ExtensibleIdentifier<Value: Hashable & Sendable & Codable, Tag>:
    Hashable, Sendable, Codable {

    public var value: Value

    public init(_ value: Value) {
        self.value = value
    }

    public init(from decoder: Decoder) throws {
        let value = try decoder.singleValueContainer().decode(Value.self)
        self.init(value)
    }

    public func encode(to encoder: any Encoder) throws {
        var container = encoder.singleValueContainer()
        try container.encode(value)
    }

    public func hash(into hasher: inout Hasher) {
        hasher.combine(value)
        hasher.combine(ObjectIdentifier(Tag.self))
    }

    public static func == (lhs: Self, rhs: Self) -> Bool {
        lhs.value == rhs.value
    }
}
```

The `Tag` generic parameter ensures type safety across different contexts without storing values.

## Using Typealiases for Convenience

Simplify declarations:

```swift
public typealias StringIdentifier<Tag> = ExtensibleIdentifier<String, Tag>

struct Event: Codable {
    typealias Category = StringIdentifier<Self>
    let title: String
    let category: Category
}
```

## Adding Syntactic Sugar

Enable literal expressions through protocol conformances:

```swift
extension ExtensibleIdentifier: ExpressibleByStringLiteral where Value == String {
    public init(stringLiteral value: String) {
        self.init(value)
    }
}

public extension Event.Category {
    static var talk: Self { "talk" }
    static var workshop: Self { "workshop" }
}
```

Values can now be extended across modules:

```swift
public extension Event.Category {
    static var roundtable: Self { "roundtable" }
}
```

## Advantages Over Enums

- **Preserves Original Data**: Maintains raw values for debugging and server communication
- **Modular Extensibility**: Cases can be defined across different frameworks
- **No Decoding Failures**: Unknown values don't break the entire process
- **Type Safety**: Different identifiers with identical values remain distinct types
- **Reusability**: Generic implementation eliminates code duplication

## Trade-offs

- Loss of `CaseIterable` protocol support
- Switch statements require `default` cases
- Less compile-time guarantees about possible values

## When to Use Enums Instead

Enums remain appropriate for universally stable values: theme modes (light/dark/system), time formats (AM/PM), or speed units (KMH/MPH) that are unlikely to expand.

## Conclusion

This struct-based approach provides superior flexibility for API integration in growing applications. By leveraging Swift's advanced type system, developers can build safer, more maintainable solutions that scale across modular architectures. The implementation is available in the [CodableKitten](https://github.com/stefanomondino/CodableKitten/blob/main/Sources/CodableKitten/ExtensibleIdentifier/ExtensibleIdentifier.swift) library currently under development.
