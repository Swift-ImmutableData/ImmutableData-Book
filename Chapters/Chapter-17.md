# Animals.app

We’re almost ready to launch our next-gen Animals product. All we have to do is make some changes on app launch. Select `Animals.xcodeproj` and open `AnimalsApp.swift`.

Here is our new `AnimalsApp`:

```swift
//  AnimalsApp.swift

import AnimalsData
import AnimalsUI
import ImmutableData
import ImmutableUI
import Services
import SwiftUI

@main @MainActor struct AnimalsApp {
  @State private var store = Store(
    initialState: AnimalsState(),
    reducer: AnimalsReducer.reduce
  )
  @State private var listener = Listener(store: Self.makeRemoteStore())
  
  init() {
    self.listener.listen(to: self.store)
  }
}

extension NetworkSession: @retroactive RemoteStoreNetworkSession {
  
}

extension AnimalsApp {
  private static func makeRemoteStore() -> RemoteStore<NetworkSession<URLSession>> {
    let session = NetworkSession(urlSession: URLSession.shared)
    return RemoteStore(session: session)
  }
}

extension AnimalsApp: App {
  var body: some Scene {
    WindowGroup {
      Provider(self.store) {
        Content()
      }
    }
  }
}
```

If we start with running `AnimalsDataServer`, we can now build and run our application (`⌘ R`). Our Animals application will now use `RemoteStore` to save its state to our HTTP server.

This is great! We just made a big leap in what this app is capable of: we added the ability to persist data to a remote server. Let’s also think about what we *didn’t* do:

* Other than building `RemoteStore` and a few small changes to add `Codable` to data models, we didn’t have to change anything in our `AnimalsData` module.
* We didn’t have to change anything in our `AnimalsUI` module.
* We just changed a few lines of code in `AnimalsApp` to migrate from `LocalStore` to `RemoteStore`.

To be fair, there are also some open questions we might have about scaling to more complex products that depend on fetching data from a remote server:

* If we run our `AnimalsDataServer`, run our Animals application, then run our `AnimalsDataClient` to perform a mutation from command-line, our Animals application is now “stale”: we don’t display the current data. More complex products could introduce a solution similar to GraphQL subscriptions.[^1] If our Vapor server delivered a web-socket connection, we could use that for a “two-way” communication: when our source-of-truth is mutated remotely, we can then push the important changes back to our client. Relay has supported GraphQL subscriptions for many years to help solve this problem.[^2]
* In a complex product, we might have multiple components that fetch similar data. An application might “over-fetch” too much data; this can reduce performance and drain battery life. If multiple components need the same data, we would like the option to make this fetch just one time. We would like more control over how and when data is cached. This is one of the problems that Relay was built to help solve.[^3]
* Waiting for a network response can be slow. If our user performs an action — like deleting an `Animal` value — that should lead to a state mutation on our server, do we need to wait for our server to return? What if we just performed that state mutation directly on the client without waiting for our server to respond? These are known as “optimistic updates”, which we saw examples of when we built our Quakes product. Managing optimistic updates can be challenging: if our server fails to perform the mutation, we need some way to “roll back” the changes that happened locally. Relay manages a lot of this work and reduces the amount of code product engineers need to be responsible for.[^4]

The Flux architecture was meant to be general and agnostic; Flux *could* have been used to build applications that fetched data from a remote server, but it also could have been used to build applications that saved all data locally. While React and Flux were being used internally at FB, product engineers began to solve the same problems over and over again shipping products that depended on a remote server. Writing repetitive code slows down product engineers and increases the chance of shipping bugs. Relay was built to solve these problems in just one place: the infra.

Redux evolved from Flux, but independent of Relay. Redux and Relay solved different problems. Relay took the principles of Flux and shipped a complex framework for fetching data from a remote server. Redux took the principles of Flux and shipped a lightweight framework with stronger opinions about immutability.

Like Flux and Redux, `ImmutableData` is lightweight. We built the infra ourselves in two chapters, and we saw that infra deploy to three different sample application products. Like Flux and Redux, `ImmutableData` is general and agnostic. We built three different sample application products with different needs: our first application saved data locally in-memory, our second application saved data in a persistent database, our third application fetched data from a remote server and saved data locally in a persistent database, and we migrated our second application to save data to a remote server.

Unlike Relay, `ImmutableData` is not built with opinions about a remote server, or what kind of data that remote server would return. We built our Quakes product and our Animals product with remote data, but we wrote this code as a *product engineer*; this was product code, not infra code.

In time, the `ImmutableData` architecture can continue evolving. A “next-gen” `ImmutableData` would probably look similar to Relay: the infra would ship with opinions about how a remote server works and how the data is structured. From those opinions, infra could then be written to solve for the repetitive problems that product engineers want solutions for: subscribing to remote changes, optimized caching logic, managing optimistic updates, and more.

[^1]: https://graphql.org/learn/subscriptions/
[^2]: https://relay.dev/docs/guided-tour/updating-data/graphql-subscriptions/
[^3]: https://relay.dev/docs/guided-tour/reusing-cached-data/
[^4]: https://relay.dev/docs/tutorial/mutations-updates/#improving-the-ux-with-an-optimistic-updater
