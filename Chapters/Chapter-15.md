# AnimalsDataServer

Before we run our Animals application with `RemoteStore`, we’re going to actually need an HTTP server to read and write data.

There are many options and languages to choose from when building a server. Our goal is to teach `ImmutableData`; many of the decisions that go into building scalable web services are outside the scope of this tutorial.

To keep things simple, we’re going to take a couple of shortcuts:

* We will use Swift to build our HTTP server.
* We will run our HTTP server on `localhost`.

This isn’t going to scale to millions of daily active users, and that’s ok. We’re just unblocking ourselves on testing our Animals application running against a real server.

Engineers are already using Swift to build server-side applications.[^1] One of the popular repos for engineering on server-side is Vapor.[^2] We will use Vapor to build an HTTP server, run our server on `localhost`, and test our Animals application.

Up to this point, the code we wrote has been almost all new. We refrained from introducing external dependencies and repos. We don’t really want there to be anything “magic” about what we are building. We built the `ImmutableData` infra and we built three sample application products against that infra. Building an HTTP server in Swift is a very specialized task. Learning how to build this technology ourselves might be interesting, but it should not block our goal of teaching `ImmutableData`. We’re going to use Vapor to move fast.

Our Animals product has the ability to read data with queries and write data with mutations. We would like our server to persist that data across launches. The Vapor ecosystem ships Fluent for persisting data in a database. We’re going to take a shortcut. Instead of learning how Fluent works, let’s just use our `LocalStore`.

Select the `AnimalsData` package and open `Sources/AnimalsDataServer/main.swift`. Let’s begin with some utilities on `LocalStore` for transforming a `RemoteRequest` to a `RemoteResponse`:

```swift
//  main.swift

import AnimalsData
import Foundation
import Vapor

extension LocalStore {
  fileprivate func response(request: RemoteRequest) throws -> RemoteResponse {
    RemoteResponse(
      query: try self.response(query: request.query),
      mutation: try self.response(mutation: request.mutation)
    )
  }
}
```

Our `RemoteRequest` was sent with an `Array` of `Query` values and an `Array` of `Mutation` values. We’re going to build a `RemoteResponse` with the data needed for our `RemoteStore`.

Here is how we build the `query` and the `mutation` of our `RemoteResponse`:

```swift
//  main.swift

extension LocalStore {
  private func response(query: Array<RemoteRequest.Query>?) throws -> Array<RemoteResponse.Query>? {
    try query?.map { query in try self.response(query: query) }
  }
}

extension LocalStore {
  private func response(mutation: Array<RemoteRequest.Mutation>?) throws -> Array<RemoteResponse.Mutation>? {
    try mutation?.map { mutation in try self.response(mutation: mutation) }
  }
}
```

We need to transform a `RemoteRequest.Query` value to a `RemoteResponse.Query`:

```swift
//  main.swift

extension LocalStore {
  private func response(query: RemoteRequest.Query) throws -> RemoteResponse.Query {
    switch query {
    case .animals:
      let animals = try self.fetchAnimalsQuery()
      return .animals(animals: animals)
    case .categories:
      let categories = try self.fetchCategoriesQuery()
      return .categories(categories: categories)
    }
  }
}
```

We need to transform a `RemoteRequest.Mutation` value to a `RemoteResponse.Mutation`:

```swift
//  main.swift

extension LocalStore {
  private func response(mutation: RemoteRequest.Mutation) throws -> RemoteResponse.Mutation {
    switch mutation {
    case .addAnimal(name: let name, diet: let diet, categoryId: let categoryId):
      let animal = try self.addAnimalMutation(name: name, diet: diet, categoryId: categoryId)
      return .addAnimal(animal: animal)
    case .updateAnimal(animalId: let animalId, name: let name, diet: let diet, categoryId: let categoryId):
      let animal = try self.updateAnimalMutation(animalId: animalId, name: name, diet: diet, categoryId: categoryId)
      return .updateAnimal(animal: animal)
    case .deleteAnimal(animalId: let animalId):
      let animal = try self.deleteAnimalMutation(animalId: animalId)
      return .deleteAnimal(animal: animal)
    case .reloadSampleData:
      let (animals, categories) = try self.reloadSampleDataMutation()
      return .reloadSampleData(animals: animals, categories: categories)
    }
  }
}
```

Let’s construct a `LocalStore`:

```swift
//  main.swift

func makeLocalStore() throws -> LocalStore<UUID> {
  if let url = Process().currentDirectoryURL?.appending(
    component: "default.store",
    directoryHint: .notDirectory
  ) {
    return try LocalStore<UUID>(url: url)
  }
  return try LocalStore<UUID>()
}
```

Let’s build our Vapor server:

```swift
//  main.swift

func main() async throws {
  let localStore = try makeLocalStore()
  let app = try await Application.make(.detect())
  app.post("animals", "api") { request in
    let response = Response()
    let remoteRequest = try request.content.decode(RemoteRequest.self)
    print(remoteRequest)
    let remoteResponse = try await localStore.response(request: remoteRequest)
    print(remoteResponse)
    try response.content.encode(remoteResponse, as: .json)
    return response
  }
  try await app.execute()
  try await app.asyncShutdown()
}

try await main()
```

If we build and run our executable, we can see our server is now running on `localhost`:

```shell
$ swift run AnimalsDataServer
[Vapor] Server started on http://127.0.0.1:8080
```

On first launch, we construct a new `LocalStore` with the sample data we built in our previous chapters. We can now run `curl` from shell and confirm the `Category` values are correct:

```shell
$ curl http://localhost:8080/animals/api -X POST -d '{"query": [ {"categories": {}} ]}' -H "Content-Type: application/json" --silent | python3 -m json.tool --indent 2
{
  "query": [
    {
      "categories": {
        "categories": [
          {
            "categoryId": "Mammal",
            "name": "Mammal"
          },
          {
            "name": "Bird",
            "categoryId": "Bird"
          },
          {
            "categoryId": "Amphibian",
            "name": "Amphibian"
          },
          {
            "categoryId": "Invertebrate",
            "name": "Invertebrate"
          },
          {
            "categoryId": "Fish",
            "name": "Fish"
          },
          {
            "categoryId": "Reptile",
            "name": "Reptile"
          }
        ]
      }
    }
  ]
}
```

We pipe our output to `python3` for pretty-printing; the original response from Vapor did not include this whitespace.

Here are the `Animal` values:

```shell
$ curl http://localhost:8080/animals/api -X POST -d '{"query": [ {"animals": {}} ]}' -H "Content-Type: application/json" --silent | python3 -m json.tool --indent 2
{
  "query": [
    {
      "animals": {
        "animals": [
          {
            "diet": "Herbivore",
            "name": "Southern gibbon",
            "categoryId": "Mammal",
            "animalId": "Bibbon"
          },
          {
            "diet": "Carnivore",
            "categoryId": "Amphibian",
            "name": "Newt",
            "animalId": "Newt"
          },
          {
            "animalId": "Cat",
            "categoryId": "Mammal",
            "diet": "Carnivore",
            "name": "Cat"
          },
          {
            "name": "House sparrow",
            "animalId": "Sparrow",
            "diet": "Omnivore",
            "categoryId": "Bird"
          },
          {
            "categoryId": "Mammal",
            "animalId": "Kangaroo",
            "name": "Red kangaroo",
            "diet": "Herbivore"
          },
          {
            "name": "Dog",
            "animalId": "Dog",
            "categoryId": "Mammal",
            "diet": "Carnivore"
          }
        ]
      }
    }
  ]
}
```

We can also experiment with a mutation:

```shell
$ curl http://localhost:8080/animals/api -X POST -d '{"mutation": [ {"addAnimal": {"name": "Eagle", "diet": "Carnivore", "categoryId": "Bird"}} ]}' -H "Content-Type: application/json" --silent | python3 -m json.tool --indent 2
{
  "mutation": [
    {
      "addAnimal": {
        "animal": {
          "animalId": "467D0044-4BB9-4C28-A10B-E87A2C328034",
          "diet": "Carnivore",
          "categoryId": "Bird",
          "name": "Eagle"
        }
      }
    }
  ]
}
```

If we stop running our server and run our server again, we can request all the `Animal` values to confirm our mutation was saved:

```shell
$ curl http://localhost:8080/animals/api -X POST -d '{"query": [ {"animals": {}} ]}' -H "Content-Type: application/json" --silent | python3 -m json.tool --indent 2
{
  "query": [
    {
      "animals": {
        "animals": [
          {
            "animalId": "Bibbon",
            "diet": "Herbivore",
            "categoryId": "Mammal",
            "name": "Southern gibbon"
          },
          {
            "animalId": "Newt",
            "name": "Newt",
            "diet": "Carnivore",
            "categoryId": "Amphibian"
          },
          {
            "name": "Cat",
            "diet": "Carnivore",
            "animalId": "Cat",
            "categoryId": "Mammal"
          },
          {
            "diet": "Omnivore",
            "categoryId": "Bird",
            "animalId": "Sparrow",
            "name": "House sparrow"
          },
          {
            "animalId": "Kangaroo",
            "name": "Red kangaroo",
            "diet": "Herbivore",
            "categoryId": "Mammal"
          },
          {
            "diet": "Carnivore",
            "animalId": "Dog",
            "categoryId": "Mammal",
            "name": "Dog"
          },
          {
            "diet": "Carnivore",
            "name": "Eagle",
            "animalId": "467D0044-4BB9-4C28-A10B-E87A2C328034",
            "categoryId": "Bird"
          }
        ]
      }
    }
  ]
}
```

With a minimal amount of new code, we not only have an HTTP server running to deliver `Category` and `Animal` values, we also leverage our existing `LocalStore` for writing to a persistent database on our filesystem.

[^1]: https://www.swift.org/documentation/server/
[^2]: https://www.swift.org/getting-started/vapor-web-server/
