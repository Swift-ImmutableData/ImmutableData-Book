# Quakes.app

There are only a few steps left before we can launch our application. Select `Quakes.xcodeproj` and open `QuakesApp.swift`. We can delete the original “Hello World” template. Let’s begin by defining our `QuakesApp` type:

```swift
//  QuakesApp.swift

import ImmutableData
import ImmutableUI
import QuakesData
import QuakesUI
import Services
import SwiftUI

@main @MainActor struct QuakesApp {
  @State private var store = Store(
    initialState: QuakesState(),
    reducer: QuakesReducer.reduce
  )
  @State private var listener = Listener(
    localStore: Self.makeLocalStore(),
    remoteStore: Self.makeRemoteStore()
  )
  
  init() {
    self.listener.listen(to: self.store)
  }
}
```

We construct our `QuakesApp` with a `Store` and a `Listener`. Our `Store` is constructed with `QuakesState` and `QuakesReducer`. Our `Listener` will be constructed with a `LocalStore` and a `RemoteStore`.

Here is where we construct our `LocalStore`:

```swift
//  QuakesApp.swift

extension QuakesApp {
  private static func makeLocalStore() -> LocalStore {
    do {
      return try LocalStore()
    } catch {
      fatalError("\(error)")
    }
  }
}
```

Here is where we construct our `RemoteStore`:

```swift
//  QuakesApp.swift

extension NetworkSession: @retroactive RemoteStoreNetworkSession {
  
}

extension QuakesApp {
  private static func makeRemoteStore() -> RemoteStore<NetworkSession<URLSession>> {
    let session = NetworkSession(urlSession: URLSession.shared)
    return RemoteStore(session: session)
  }
}
```

We saw `NetworkSession` when we built our `QuakesDataClient` executable. It’s a lightweight wrapper around `URLSession` for requesting and serializing JSON. Our `RemoteStore` needs a type that adopts `RemoteStoreNetworkSession`: we can extend `NetworkSession` to conform to `RemoteStoreNetworkSession`. The `retroactive` attribute can silence a compiler warning.[^1] It’s a legit warning, but we control `NetworkSession` and `QuakesApp`; there’s not much of a risk of this breaking anything for our tutorial.

Here is our `body` property:

```swift
//  QuakesApp.swift

extension QuakesApp: App {
  var body: some Scene {
    WindowGroup {
      Provider(self.store) {
        Content()
      }
    }
  }
}
```

We can now build and run (`⌘ R`). Our application is now a working clone of the original Quakes sample application from Apple. We preserved the core functionality: we launch our app and fetch `Quake` values from the USGS server. We can filter and sort `Quake` values. Our `List` and `Map` components stay updated with the same subset of `Quake` values. We also save our global state on our filesystem in a local database. Launching the app a second time loads the `Quake` values that were previously cached. We also have the option to delete `Quake` values from our local database.

You can also create a new window (`⌘ N`) and watch as edits to global state from one window are reflected in the other. The global state is the complete set of `Quake` values: this is consistent. The local state is tracked independently: users can sort and filter on one window without affecting the sort and filter options of other windows.

Similar to our Animals product, we can also use Launch Arguments to see the `print` statements we added for debugging:

```text
-com.northbronson.QuakesData.Debug 1
-com.northbronson.QuakesUI.Debug 1
-com.northbronson.ImmutableUI.Debug 1
-com.apple.CoreData.SQLDebug 1
-com.apple.CoreData.ConcurrencyDebug 1
```

It’s easy to save thousands of `Quake` values in global state. Turning on `com.northbronson.QuakesData.Debug` will print *a lot* of data. If you were interested in trying to reduce the amount of data printed, you could try and update `QuakesData.Listener` to print only the *difference* between the state of our system before and after our Reducer returned. Instead of printing the old state and the new state, think about an algorithm to compare two state values and only print the diff.

Go ahead and try launching the original sample project from Apple. The original sample project fetched all the earthquake values recorded during the current day. a typical day might be about 300 to 400 earthquake values. To test performance with more data, we hacked the original sample project to fetch all the earthquake values recording during the current month. Now, we are fetching an order of magnitude more earthquake values: about 10K. Depending on your network connection, the work to download that data from USGS is not the major bottleneck. The `URLSession` operation from the original sample project is asynchronous and concurrent. Our `main` thread is not blocked: we can still scroll around our `Map` component, even if we don’t have any data to display. Where things get difficult is when those earthquake values have returned from USGS. We then iterate through and insert new `PersistentModel` reference in our `ModelContext`. This work happens on `main`, and it’s slow. *This* is the bottleneck that leads to apps freezing and “beachballs” spinning.

To be fair, there are more advanced techniques that can perform asynchronous and concurrent `ModelContext` operations without blocking `main`.[^2] Similar to CoreData, we can then attempt to “sync” a `ModelContext` tied to `main` with a `ModelContext` performing asynchronous and concurrent operations. This can help improve performance, but it’s complex code; we don’t want our product engineers to have to think about this. Even if we *were* able to factor all this code down into shared infra and we *were* able to write all the automated tests and we *were* able to defend against all the edge-cases, we would still be stuck with a programming model we don’t agree with: product engineers would be performing imperative mutations on mutable objects from their component tree.

Migrating to `ImmutableData` fixes both these problems for us: it’s an “abstraction layer” that makes it easy to keep expensive SwiftData operations asynchronous and concurrent, and it delivers immutable value types to our component tree.

[^1]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0364-retroactive-conformance-warning.md
[^2]: https://fatbobman.com/en/posts/concurret-programming-in-swiftdata/
