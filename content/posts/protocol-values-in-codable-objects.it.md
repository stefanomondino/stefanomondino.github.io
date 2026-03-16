---
title: "Valori protocol-based in oggetti Codable"
date: 2025-07-11
draft: false
tags: ["Codable", "Architecture", "Swift"]
description: "Gestire array di tipi misti con Decodable in Swift è piuttosto difficile: ecco un trucco avanzato per una soluzione scalabile e a prova di futuro."
image: "images/protocol-values-in-codable-objects.png"
---

Lavorando con le API JSON, è abbastanza comune incontrare proprietà complesse: sia singoli oggetti che array di oggetti. Come discusso in un articolo precedente sulle alternative agli enum, la strategia tipica prevede la creazione di oggetti Swift conformi a `Decodable` con una proprietà per ogni valore JSON.

Per oggetti più complessi, gli sviluppatori dichiarano proprietà con tipi `Decodable` (per valori singoli) o array di oggetti `Decodable` (per liste). Questo approccio però si rompe quando gli oggetti JSON devono conformarsi a un protocollo piuttosto che a un tipo concreto, uno scenario che richiede una soluzione più sofisticata.

## Il problema: una lista di animali

Si consideri un'app per un ospedale veterinario che recupera una lista di pazienti animali. Ogni animale ha una proprietà `type` e un `name`, ma le altre proprietà variano per tipo di animale:

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

Dichiarare array basati su protocollo in oggetti `Decodable` non funziona:

```swift
struct Response: Decodable {
    let items: [any Animal] // Non compila
}
```

Il sistema `Decodable` richiede di conoscere i tipi concreti in anticipo. Senza dichiarare esplicitamente quali tipi esistono e dove si trovano, il sistema non può mappare JSON in valori conformi a un protocollo, una limitazione critica in applicazioni modulari dove i tipi animale potrebbero essere distribuiti in moduli diversi.

## La strategia di mapping

Per risolvere il problema occorre:

1. **Estrarre una proprietà discriminante** dall'elemento JSON corrente per determinarne il tipo
2. **Iterare sugli elementi dell'array** singolarmente invece di decodificare l'intero array in una volta
3. **Selezionare dinamicamente il tipo concreto** appropriato in base al discriminante estratto
4. **Accumulare i risultati** in un array di valori conformi al protocollo

Un'implementazione grezza mostra la complessità:

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

**Complicazioni principali:**

- La complessità ciclomatica aumenta ad ogni nuovo tipo di animale
- Richiede di conoscere tutti i tipi concreti nel contesto di Response, problematico in architetture modulari
- Lo switch statement va duplicato per valori singoli, opzionali e array
- La logica non è riutilizzabile in scenari polimorfici diversi

## La soluzione: Decodable Polymorphism

Un approccio standardizzato richiede tre componenti:

1. **Protocollo `Polymorphic`**: dichiara che un tipo partecipa al decoding polimorfico
2. **Protocollo `TypeExtractor`**: implementa la logica per identificare ed estrarre il tipo concreto corretto
3. **Property wrapper `@Polymorph`**: semplifica la dichiarazione di proprietà polimorfiche negli oggetti `Decodable`

### Il protocollo Polymorphic

```swift
public protocol Polymorphic: Decodable, Sendable {
    associatedtype Extractor: TypeExtractor
    static var typeExtractor: Extractor { get }
}
```

### Il protocollo TypeExtractor

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

Le estensioni del protocollo forniscono implementazioni di default per i metodi di estrazione, eliminando il boilerplate:

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

### Implementazione concreta di TypeExtractor

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

### Definizione del protocollo e conformance

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

### Il property wrapper @Polymorph

```swift
@propertyWrapper
public struct Polymorph<Extractor: TypeExtractor, Value: Sendable>: Decodable, Sendable {
    private var value: Value?
    public var wrappedValue: Value {
        get { return value.unsafelyUnwrapped }
        set { value = newValue }
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

### Gestione dei valori opzionali

Con valori wrapped opzionali, il decoder deve riconoscere che il property wrapper è sempre presente anche se il valore sottostante è `nil`:

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

### Dichiarazione dei tipi disponibili

Il decoder deve sapere quali tipi concreti sono disponibili per un dato extractor. Questo viene gestito attraverso il dizionario `userInfo` del decoder:

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
```

## Esempio completo

```swift
struct Response: Decodable {
    @Polymorph<AnimalType, [any Animal]> let items
}

let data = Data(json.utf8)
let decoder = JSONDecoder()
decoder.set(types: [Dog.self, Cat.self], for: AnimalType.self)
let response = try decoder.decode(Response.self, from: data)
print(response.items)
```

## Vantaggi

- **Boilerplate eliminato**: la logica di switch e decoding complesso si sposta in estensioni di protocollo riutilizzabili
- **Scalabilità modulare**: nuovi tipi animale possono essere aggiunti in moduli diversi senza modificare Response
- **Sintassi uniforme**: un unico wrapper `@Polymorph` gestisce valori singoli, opzionali, array e array opzionali
- **Riutilizzabilità**: lo stesso pattern funziona per qualsiasi scenario polimorfico: animali, messaggi di chat, moduli di contenuto
- **A prova di futuro**: aggiungere nuovi tipi richiede solo la creazione di una nuova struct conforme al protocollo e la sua registrazione nel decoder

## Conclusione

Gestire oggetti polimorfici nel sistema `Decodable` di Swift richiede tecniche sofisticate, ma una buona astrazione elimina la complessità. Incapsulando la logica di estrazione in protocolli e sfruttando i property wrapper, è possibile mantenere codice pulito e scalabile nella gestione di risposte API complesse.

L'implementazione completa è disponibile come libreria Swift Package Manager open-source chiamata [CodableKitten](https://github.com/stefanomondino/CodableKitten), che fornisce componenti pronti all'uso per il decoding polimorfico in applicazioni reali.
