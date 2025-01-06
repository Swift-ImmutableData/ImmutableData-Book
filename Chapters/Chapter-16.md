# AnimalsDataClient

Our `AnimalsDataClient` executable was originally built to use `LocalStore` from command-line. Let’s try testing our `RemoteStore`.

Select the `AnimalsData` package and open `Sources/AnimalsDataClient/main.swift`. Here is all we need for now:

```swift
//  main.swift

import AnimalsData
import Foundation
import Services

extension NetworkSession: RemoteStoreNetworkSession {
  
}

func makeRemoteStore() -> RemoteStore<NetworkSession<URLSession>> {
  let session = NetworkSession(urlSession: URLSession.shared)
  return RemoteStore(session: session)
}

func main() async throws {
  let store = makeRemoteStore()
  
  let animals = try await store.fetchAnimalsQuery()
  print(animals)
  
  let categories = try await store.fetchCategoriesQuery()
  print(categories)
}

try await main()
```

If we start with running `AnimalsDataServer`, we can run `AnimalsDataClient` to perform queries and mutations on `RemoteStore`.

Try it for yourself: experiment with mutating an `Animal` value on `RemoteStore`. Run your executables again and confirm the mutations were persisted.
