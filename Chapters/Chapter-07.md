# AnimalsDataClient

When building products with dependencies on complex services like SwiftData, it can be very handy to be able to run things “end-to-end” outside our UI. We can think of this as an “integration test”; this is in addition to — not a replacement for — conventional unit tests.

Before we build our component graph and put these data models on-screen, let’s build a simple command-line executable against our `LocalStore`. Our `AnimalsDataClient` executable will build against our production `LocalStore` class and persist its data on our filesystem. This means we can run our `AnimalsDataClient` executable, mutate our data, and see that new data the next time we run.

To run against SwiftData from a command-line executable, we need a little work to configure our module.[^1] The Workspace from the [`ImmutableData-Samples`](https://github.com/Swift-ImmutableData/ImmutableData-Samples) repo already includes this configuration to save us some time. All we have to do is select the `AnimalsData` package and open `Sources/AnimalsDataClient/main.swift`.

Here is all we need to begin operating on a `LocalStore`:

```swift
//  main.swift

import AnimalsData
import Foundation

func makeLocalStore() throws -> LocalStore<UUID> {
  if let url = Process().currentDirectoryURL?.appending(
    component: "default.store",
    directoryHint: .notDirectory
  ) {
    return try LocalStore<UUID>(url: url)
  }
  return try LocalStore<UUID>()
}

func main() async throws {
  let store = try makeLocalStore()
  
  let animals = try await store.fetchAnimalsQuery()
  print(animals)
  
  let categories = try await store.fetchCategoriesQuery()
  print(categories)
}

try await main()
```

Our `main` function builds a `LocalStore` instance. We can then read from our `LocalStore` and perform mutations.

Try it for yourself: think of this executable as a “playground”. Try adding mutations. Run your executable again and confirm the mutations were persisted.

[^1]: https://forums.developer.apple.com/forums/thread/734540
