---
title: "Un'alternativa scalabile agli enum Decodable"
date: 2025-04-23
draft: false
tags: ["Codable", "Swift", "Architecture"]
description: "Un approccio scalabile per il decoding degli enum in Swift. Codice più pulito, maggiore flessibilità e migliore manutenibilità per codebase in crescita."
image: "images/a-scalable-alternative-to-decodable-enums.png"
---

I protocolli `Decodable` e `Encodable` di Swift offrono un modo comodo per mappare dati JSON in oggetti Swift nativi. Tuttavia, gli approcci base basati su enum possono diventare problematici con l'evolversi dei progetti e l'introduzione di nuovi valori nelle API.

## Il problema con gli enum Decodable standard

Quando si mappano le risposte API con gli enum, nuovi valori non dichiarati causano il fallimento completo del decoding. Per esempio, se un'app per conferenze mappa le categorie degli eventi come enum basato su `String`:

```swift
enum Category: String, Codable {
    case talk
    case workshop
}
```

L'aggiunta di una nuova categoria "roundtable" rompe l'intero processo di decoding, facendo fallire persino il JSON valido che contiene quel nuovo valore.

## Limiti della soluzione tradizionale

La soluzione comune aggiunge un case `unknown`:

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

Questo approccio ha tre criticità:

1. **Perdita di dati**: I valori originali vengono scartati, complicando il debugging e la comunicazione con il server
2. **Duplicazione del codice**: Ogni enum richiede una logica di decoding personalizzata
3. **Problemi di scalabilità**: Gli enum non possono essere estesi tra moduli diversi in architetture modulari

## Un'alternativa basata su struct

Al posto degli enum, si usano struct che preservano il valore originale:

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

## Una soluzione generica riutilizzabile

Si crea un tipo generico `ExtensibleIdentifier`:

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

Il parametro generico `Tag` garantisce la type safety tra contesti diversi senza memorizzare valori.

## Typealias per comodità

Si semplificano le dichiarazioni:

```swift
public typealias StringIdentifier<Tag> = ExtensibleIdentifier<String, Tag>

struct Event: Codable {
    typealias Category = StringIdentifier<Self>
    let title: String
    let category: Category
}
```

## Syntactic sugar

Si abilitano le espressioni letterali tramite conformance ai protocolli:

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

I valori possono ora essere estesi tra i moduli:

```swift
public extension Event.Category {
    static var roundtable: Self { "roundtable" }
}
```

## Vantaggi rispetto agli enum

- **Preserva i dati originali**: mantiene i valori grezzi per debugging e comunicazione con il server
- **Estensibilità modulare**: i casi possono essere definiti in framework diversi
- **Nessun fallimento del decoding**: i valori sconosciuti non rompono il processo
- **Type safety**: identificatori diversi con valori identici rimangono tipi distinti
- **Riutilizzabilità**: l'implementazione generica elimina la duplicazione del codice

## Compromessi

- Perdita del supporto al protocollo `CaseIterable`
- Gli switch statement richiedono il case `default`
- Minori garanzie a compile-time sui valori possibili

## Quando usare gli enum

Gli enum rimangono appropriati per valori universalmente stabili: modalità tema (light/dark/system), formati orari (AM/PM) o unità di velocità (KMH/MPH) che difficilmente si espanderanno.

## Conclusione

Questo approccio basato su struct offre una flessibilità superiore per l'integrazione con le API in applicazioni in crescita. Sfruttando il sistema avanzato di tipi di Swift, è possibile costruire soluzioni più sicure e manutenibili che scalano in architetture modulari. L'implementazione è disponibile nella libreria [CodableKitten](https://github.com/stefanomondino/CodableKitten/blob/main/Sources/CodableKitten/ExtensibleIdentifier/ExtensibleIdentifier.swift), attualmente in sviluppo.
