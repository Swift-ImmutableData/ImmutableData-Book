# QuakesDataClient

Similar to our `AnimalsDataClient` executable, we can build a `QuakesDataClient` executable for testing against a “real” SwiftData database. Select the `QuakesData` package and open `Sources/QuakesDataClient/main.swift`.

Here is our work to construct a `LocalStore` and a `RemoteStore`:

```swift
//  main.swift

import Foundation
import QuakesData
import Services

extension NetworkSession: RemoteStoreNetworkSession {
  
}

func makeLocalStore() throws -> LocalStore {
  if let url = Process().currentDirectoryURL?.appending(
    component: "default.store",
    directoryHint: .notDirectory
  ) {
    return try LocalStore(url: url)
  }
  return try LocalStore()
}

func makeRemoteStore() -> RemoteStore<NetworkSession<URLSession>> {
  let session = NetworkSession(urlSession: URLSession.shared)
  return RemoteStore(session: session)
}

func main() async throws {
  let localStore = try makeLocalStore()
  let remoteStore = makeRemoteStore()
  
  let localQuakes = try await localStore.fetchLocalQuakesQuery()
  print(localQuakes)
  
  let remoteQuakes = try await remoteStore.fetchRemoteQuakesQuery(range: .allHour)
  print(remoteQuakes)
}

try await main()
```

The `Services` module is provided along with our Workspace from the [`ImmutableData-Samples`](https://github.com/Swift-ImmutableData/ImmutableData-Samples) repo. This module provides a `NetworkSession` class that simplifies fetching and serializing data models from a remote server. Our goal with this tutorial is to teach the `ImmutableData` architecture. Networking code is interesting, but does not really need to block that main goal. You are welcome to explore other solutions for networking in your own products. We provide the `Services` module just to keep things easy for our tutorial.

Once we have a `NetworkSession` instance, we use that to create a `RemoteStore`. We create a `LocalStore` using a similar pattern to what we saw in `AnimalsDataClient`.

Our `main` function constructs a `LocalStore` and a `RemoteStore`. We now have the option to begin performing queries and mutations. Try it for yourself. You can fetch earthquakes from USGS, insert those earthquakes in a local database, then run your executable again and confirm the mutations were persisted.
