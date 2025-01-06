# AnimalsData

One of the realities of the practice of Software Engineering is the inevitable “changing requirements”.[^1] We build an application, customers need a new feature, and we have to build something. Sometimes these customers are external customers: the actual users that paid for our product. Sometimes these customers are internal customers: our product managers or engineering managers.

Our `ImmutableData` architecture looks good so far for “greenfield” products: three new sample products we built from scratch. What happens *after* the product is built? How flexible are these products to adapt to changing requirements?

Our Animals product was built to save data to a local database. Our experiment will be to build a “next-gen” Animals product. The changing requirements are we now need to support saving data to a remote server. We don’t have to cache data locally; the server will be our “source of truth”.

In a world where our SwiftUI application is built directly on SwiftData, this is possible using some advanced techniques with `DataStore`.[^2] This was also possible in Core Data using `NSIncrementalStore`.[^3] We’re going to take a slightly different approach. We’re not going to update our existing `LocalStore` to forward its queries and mutations to a server. We’re going to build a `RemoteStore` and replace our existing `LocalStore` on app launch.

## Category

Our remote server will return JSON. To serialize that JSON to and from `Category` values, we are going to adopt `Codable`. Select the `AnimalsData` package and open `Sources/AnimalsData/Category.swift`. Add the `Codable` conformance to our main declaration:

```swift
//  Category.swift

public struct Category: Hashable, Codable, Sendable {
  public let categoryId: String
  public let name: String
  
  package init(
    categoryId: String,
    name: String
  ) {
    self.categoryId = categoryId
    self.name = name
  }
}
```

## Animal

Let’s update `Animal` to adopt `Codable`. Open `Sources/AnimalsData/Animal.swift`. Add the `Codable` conformance to our main declaration:

```swift
//  Animal.swift

public struct Animal: Hashable, Codable, Sendable {
  public let animalId: String
  public let name: String
  public let diet: Diet
  public let categoryId: String
  
  package init(
    animalId: String,
    name: String,
    diet: Diet,
    categoryId: String
  ) {
    self.animalId = animalId
    self.name = name
    self.diet = diet
    self.categoryId = categoryId
  }
}
```

We also add `Codable` to the `Diet` declaration:

```swift
//  Animal.swift

extension Animal {
  public enum Diet: String, CaseIterable, Hashable, Codable, Sendable {
    case herbivorous = "Herbivore"
    case carnivorous = "Carnivore"
    case omnivorous = "Omnivore"
  }
}
```

## RemoteStore

Our `PersistentSession` performs asynchronous queries and mutations on a type that conforms to `PersistentSessionPersistentStore`. Our `PersistentSession` does not have any *direct* knowledge of SwiftData: that knowledge lives in `LocalStore`. We’re going to build a new type that conforms to `PersistentSessionPersistentStore`. This type will perform asynchronous queries and mutations against a remote server.

There’s no “one right way” to design a remote API. This is an interesting topic, but it’s orthogonal to our main goal of teaching `ImmutableData`. We’re going to build a remote server with an API inspired by GraphQL.[^4] The history of GraphQL is closely tied with the history of Relay.[^5] Relay evolved out of the Flux Architecture with an emphasis on retrieving server data.[^6] Relay and Redux evolved independently of each other, but they both share a common ancestor: Flux.

This isn’t meant as a strong opinion about your own products built from `ImmutableData`. There’s actually nothing in our `RemoteStore` that will need any direct knowledge of the `ImmutableData` architecture. If your server engineers build something that looks like “classic REST”, that doesn’t need to block you on shipping with `ImmutableData`.

Add a new Swift file under `Sources/AnimalsData`. Name this file `RemoteStore.swift`.

Let’s begin with a type to define our Request. This is the outgoing communication from our client to our server.

```swift
//  RemoteStore.swift

import Foundation

package struct RemoteRequest: Hashable, Codable, Sendable {
  package let query: Array<Query>?
  package let mutation: Array<Mutation>?
  
  package init(
    query: Array<Query>? = nil,
    mutation: Array<Mutation>? = nil
  ) {
    self.query = query
    self.mutation = mutation
  }
}
```

Our `RemoteRequest` is constructed with two parameters: an `Array` of `Query` values and an `Array` of `Mutation` values. We will use `Query` values to indicate operations to read data and `Mutation` values to indicate operations to write data.

We will define two possible queries:

```swift
//  RemoteStore.swift

extension RemoteRequest {
  package enum Query: Hashable, Codable, Sendable {
    case categories
    case animals
  }
}
```

This is easy: one option to request `Animal` values and one option to request `Category` values.

We will define four possible mutations:

```swift
//  RemoteStore.swift

extension RemoteRequest {
  package enum Mutation: Hashable, Codable, Sendable {
    case addAnimal(
      name: String,
      diet: Animal.Diet,
      categoryId: String
    )
    case updateAnimal(
      animalId: String,
      name: String,
      diet: Animal.Diet,
      categoryId: String
    )
    case deleteAnimal(animalId: String)
    case reloadSampleData
  }
}
```

This should look familiar: these look a lot like the two queries and four mutations we defined on `PersistentSessionPersistentStore`.

There are two more constructors that will make things easier for us when we only need one `Query` or one `Mutation`:

```swift
//  RemoteStore.swift

extension RemoteRequest {
  fileprivate init(query: Query) {
    self.init(
      query: [
        query
      ]
    )
  }
}

extension RemoteRequest {
  fileprivate init(mutation: Mutation) {
    self.init(
      mutation: [
        mutation
      ]
    )
  }
}
```

Let’s build a type to define our Response. This is the incoming communication from our server to our client.

```swift
//  RemoteStore.swift

package struct RemoteResponse: Hashable, Codable, Sendable {
  package let query: Array<Query>?
  package let mutation: Array<Mutation>?
  
  package init(
    query: Array<Query>? = nil,
    mutation: Array<Mutation>? = nil
  ) {
    self.query = query
    self.mutation = mutation
  }
}
```

Our `RemoteResponse` is constructed with two parameters: an `Array` of `Query` values and an `Array` of `Mutation` values.

We will define two possible queries:

```swift
//  RemoteStore.swift

extension RemoteResponse {
  package enum Query: Hashable, Codable, Sendable {
    case categories(categories: Array<Category>)
    case animals(animals: Array<Animal>)
  }
}
```

We will define four possible mutations:

```swift
//  RemoteStore.swift

extension RemoteResponse {
  package enum Mutation: Hashable, Codable, Sendable {
    case addAnimal(animal: Animal)
    case updateAnimal(animal: Animal)
    case deleteAnimal(animal: Animal)
    case reloadSampleData(
      animals: Array<Animal>,
      categories: Array<Category>
    )
  }
}
```

We also want some `Error` types if the response from our server is missing data:

```swift
//  RemoteStore.swift

extension RemoteResponse.Query {
  package struct Error: Swift.Error {
    package enum Code: Equatable {
      case categoriesNotFound
      case animalsNotFound
    }
    
    package let code: Self.Code
  }
}

extension RemoteResponse.Mutation {
  package struct Error: Swift.Error {
    package enum Code: Equatable {
      case animalNotFound
      case sampleDataNotFound
    }
    
    package let code: Self.Code
  }
}
```

Similar to `QuakesData.RemoteStore`, we’re going to define a protocol for our network session. We don’t want our `RemoteStore` to explicitly depend on `URLSession` or any type that performs “real” networking; we want the ability to inject a test-double to return stub data in tests.

```swift
//  RemoteStore.swift

public protocol RemoteStoreNetworkSession: Sendable {
  func json<T>(
    for request: URLRequest,
    from decoder: JSONDecoder
  ) async throws -> T where T : Decodable
}
```

Here is the main declaration of our `RemoteStore`:

```swift
//  RemoteStore.swift

final public actor RemoteStore<NetworkSession>: PersistentSessionPersistentStore where NetworkSession : RemoteStoreNetworkSession {
  private let session: NetworkSession
  
  public init(session: NetworkSession) {
    self.session = session
  }
}
```

We’re going to write a utility to serialize our `RemoteRequest` to a `URLRequest` that can be forwarded to our `NetworkSession`:

```swift
//  RemoteStore.swift

extension RemoteStore {
  package struct Error : Swift.Error {
    package enum Code: Equatable {
      case urlError
      case requestError
    }
    
    package let code: Self.Code
  }
}

extension RemoteStore {
  private static func networkRequest(remoteRequest: RemoteRequest) throws -> URLRequest {
    guard
      let url = URL(string: "http://localhost:8080/animals/api")
    else {
      throw Error(code: .urlError)
    }
    var networkRequest = URLRequest(url: url)
    networkRequest.httpMethod = "POST"
    networkRequest.setValue("application/json", forHTTPHeaderField: "Content-Type")
    networkRequest.httpBody = try {
      do {
        return try JSONEncoder().encode(remoteRequest)
      } catch {
        throw Error(code: .requestError)
      }
    }()
    return networkRequest
  }
}
```

In the next chapter, we will use Vapor to build a server that runs on `localhost`. The `URL` endpoint will accept a `POST` request and return JSON.

We can now add the functions to conform to `PersistentSessionPersistentStore`. Let’s begin with `fetchCategoriesQuery`:

```swift
//  RemoteStore.swift

extension RemoteStore {
  public func fetchCategoriesQuery() async throws -> Array<Category> {
    let remoteRequest = RemoteRequest(query: .categories)
    let networkRequest = try Self.networkRequest(remoteRequest: remoteRequest)
    let remoteResponse: RemoteResponse = try await self.session.json(
      for: networkRequest,
      from: JSONDecoder()
    )
    guard
      let query = remoteResponse.query,
      let categories = {
        let element = query.first { element in
          if case .categories = element {
            return true
          }
          return false
        }
        if case .categories(categories: let categories) = element {
          return categories
        }
        return nil
      }()
    else {
      throw RemoteResponse.Query.Error(code: .categoriesNotFound)
    }
    return categories
  }
}
```

This might look like a lot of code, but we can think through things step-by-step:

* We construct a `RemoteRequest` with `Query.categories`.
* We transform our `RemoteRequest` to a `URLRequest`.
* We forward our `URLRequest` to our `NetworkSession` and `await` a `RemoteResponse`.
* We look inside our `RemoteResponse` for a `Query.categories` value. If we found a `Query.categories` value, we return the `Array` of `Category` values returned by our server. If this values was missing, we throw an `Error`. We can assume that our server would return at most one `Query.categories` value — we don’t need to code around two different `Query.categories` values returned in the same `RemoteResponse`.

Here is a similar pattern for `fetchAnimalsQuery`:

```swift
//  RemoteStore.swift

extension RemoteStore {
  public func fetchAnimalsQuery() async throws -> Array<Animal> {
    let remoteRequest = RemoteRequest(query: .animals)
    let networkRequest = try Self.networkRequest(remoteRequest: remoteRequest)
    let remoteResponse: RemoteResponse = try await self.session.json(
      for: networkRequest,
      from: JSONDecoder()
    )
    guard
      let query = remoteResponse.query,
      let animals = {
        let element = query.first { element in
          if case .animals = element {
            return true
          }
          return false
        }
        if case .animals(animals: let animals) = element {
          return animals
        }
        return nil
      }()
    else {
      throw RemoteResponse.Query.Error(code: .animalsNotFound)
    }
    return animals
  }
}
```

On app launch, we will perform a `fetchCategoriesQuery` *and* a `fetchAnimalsQuery`. This will be two different network requests. A legit optimization would be to add more code in our `Listener` to call only one query on app launch; something like an `appLaunchQuery`. We can then make one network request for `Query.categories` and `Query.animals` at the same time: our server would return them both together.

Mutations will follow a similar pattern. Here is `addAnimalMutation`:

```swift
//  RemoteStore.swift

extension RemoteStore {
  public func addAnimalMutation(
    name: String,
    diet: Animal.Diet,
    categoryId: String
  ) async throws -> Animal {
    let remoteRequest = RemoteRequest(
      mutation: .addAnimal(
        name: name,
        diet: diet,
        categoryId: categoryId
      )
    )
    let networkRequest = try Self.networkRequest(remoteRequest: remoteRequest)
    let remoteResponse: RemoteResponse = try await self.session.json(
      for: networkRequest,
      from: JSONDecoder()
    )
    guard
      let mutation = remoteResponse.mutation,
      let animal = {
        let element = mutation.first { element in
          if case .addAnimal = element {
            return true
          }
          return false
        }
        if case .addAnimal(animal: let animal) = element {
          return animal
        }
        return nil
      }()
    else {
      throw RemoteResponse.Mutation.Error(code: .animalNotFound)
    }
    return animal
  }
}
```

Here is `updateAnimalMutation`:

```swift
//  RemoteStore.swift

extension RemoteStore {
  public func updateAnimalMutation(
    animalId: String,
    name: String,
    diet: Animal.Diet,
    categoryId: String
  ) async throws -> Animal {
    let remoteRequest = RemoteRequest(
      mutation: .updateAnimal(
        animalId: animalId,
        name: name,
        diet: diet,
        categoryId: categoryId
      )
    )
    let networkRequest = try Self.networkRequest(remoteRequest: remoteRequest)
    let remoteResponse: RemoteResponse = try await self.session.json(
      for: networkRequest,
      from: JSONDecoder()
    )
    guard
      let mutation = remoteResponse.mutation,
      let animal = {
        let element = mutation.first { element in
          if case .updateAnimal = element {
            return true
          }
          return false
        }
        if case .updateAnimal(animal: let animal) = element {
          return animal
        }
        return nil
      }()
    else {
      throw RemoteResponse.Mutation.Error(code: .animalNotFound)
    }
    return animal
  }
}
```

Here is `deleteAnimalMutation`:

```swift
//  RemoteStore.swift

extension RemoteStore {
  public func deleteAnimalMutation(animalId: String) async throws -> Animal {
    let remoteRequest = RemoteRequest(
      mutation: .deleteAnimal(animalId: animalId)
    )
    let networkRequest = try Self.networkRequest(remoteRequest: remoteRequest)
    let remoteResponse: RemoteResponse = try await self.session.json(
      for: networkRequest,
      from: JSONDecoder()
    )
    guard
      let mutation = remoteResponse.mutation,
      let animal = {
        let element = mutation.first { element in
          if case .deleteAnimal = element {
            return true
          }
          return false
        }
        if case .deleteAnimal(animal: let animal) = element {
          return animal
        }
        return nil
      }()
    else {
      throw RemoteResponse.Mutation.Error(code: .animalNotFound)
    }
    return animal
  }
}
```

Here is `reloadSampleDataMutation`:

```swift
//  RemoteStore.swift

extension RemoteStore {
  public func reloadSampleDataMutation() async throws -> (
    animals: Array<Animal>,
    categories: Array<Category>
  ) {
    let remoteRequest = RemoteRequest(
      mutation: .reloadSampleData
    )
    let networkRequest = try Self.networkRequest(remoteRequest: remoteRequest)
    let remoteResponse: RemoteResponse = try await self.session.json(
      for: networkRequest,
      from: JSONDecoder()
    )
    guard
      let mutation = remoteResponse.mutation,
      let (animals, categories) = {
        let element = mutation.first { element in
          if case .reloadSampleData = element {
            return true
          }
          return false
        }
        if case .reloadSampleData(animals: let animals, categories: let categories) = element {
          return (animals, categories)
        }
        return nil
      }()
    else {
      throw RemoteResponse.Mutation.Error(code: .sampleDataNotFound)
    }
    return (animals, categories)
  }
}
```

Over the course of this tutorial, we’ve presented some strong opinions and value-statements about state-management. We feel strongly about these opinions, but it’s important to remember that some of this code was “arbitrary” in the sense that the designs and patterns are orthogonal to our goal of teaching `ImmutableData`. The `ImmutableData` has strong opinions about *how* components should affect transformations on global state, but we don’t always have strong opinions about *what* that transformation should look like.

Our `LocalStore` was built to persist data to our filesystem using SwiftData. We could have chosen Core Data, SQLite, or something else; it is an implementation detail that would not need to influence how we go about building apps on `ImmutableData`. The decision to choose SwiftData — and the code we wrote to interact with SwiftData — is not meant to sound like an opinion about “the right way” to use `ImmutableData`.

Similarly, our `RemoteStore` is built on `URLSession` and an endpoint that presents a GraphQL-inspired API. These are arbitrary; your own products might look very different, and that’s ok.

[^1]: https://martinfowler.com/distributedComputing/soft.pdf
[^2]: https://developer.apple.com/videos/play/wwdc2024/10138/
[^3]: https://nshipster.com/nsincrementalstore/
[^4]: https://graphql.org/learn/
[^5]: https://engineering.fb.com/2015/09/14/core-infra/graphql-a-data-query-language/
[^6]: https://engineering.fb.com/2015/09/14/core-infra/relay-declarative-data-for-react-applications/
