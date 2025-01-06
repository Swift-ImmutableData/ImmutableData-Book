# ImmutableUI

Now that we have our `Store` class for saving the state of our system at a moment in time, we want a set of types to support selecting and transforming that state from SwiftUI view components.

There are three types we focus our effort on for this chapter:

* `Provider`: A SwiftUI view component for passing a `Store` instance through a view component graph.
* `Dispatcher`: A SwiftUI `DynamicProperty` for dispatching Action values from a SwiftUI view component.
* `Selector`: A SwiftUI `DynamicProperty` for selecting a slice of State from a SwiftUI view component.

## Provider

If you have experience with shipping SwiftData in a SwiftUI application, you might be familiar with the `modelContext` environment value.[^1] A typical SwiftUI application might initialize a `ModelContext` instance at app launch, and then make that `ModelContext` available through the view component graph with `SwiftUI.Environment`.

SwiftData supports creating multiple `ModelContext` instances over your app lifecycle. The Flux architecture supported multiple `Store` instances. We choose an approach consistent with the Redux architecture; we will create just one `Store` instance at app launch. We then make that `Store` available through the view component graph with `SwiftUI.Environment`.

Select the `ImmutableUI` package and add a new Swift file under `Sources/ImmutableUI`. Name this file `Provider.swift`. This view component is not very complex; here is all we need:

```swift
//  Provider.swift

import SwiftUI

@MainActor public struct Provider<Store, Content> where Content : View {
  private let keyPath: WritableKeyPath<EnvironmentValues, Store>
  private let store: Store
  private let content: Content
  
  public init(
    _ keyPath: WritableKeyPath<EnvironmentValues, Store>,
    _ store: Store,
    @ViewBuilder content: () -> Content
  ) {
    self.keyPath = keyPath
    self.store = store
    self.content = content()
  }
}

extension Provider : View {
  public var body: some View {
    self.content.environment(self.keyPath, self.store)
  }
}
```

Our initializer takes three parameters:
* A `keyPath` where we save our `store` instance.
* A `store` instance of a generic `Store` type.
* A `content` closure to build a `Content` view component.

All we have to do when computing our `body` is set the `environment` value for `keyPath` to our `store` on the `content` view component graph. This `store` will then be available to use across that view component graph similar to `modelContext`.

Our `Provider` component serves a similar role as the `Provider` component in React Redux.[^2]

## Dispatcher

Now that we have the ability to pass a `Store` type through our `environment`, how do we dispatch an action from a SwiftUI view component when an important event occurs? Add a new Swift file under `Sources/ImmutableUI`. Name this file `Dispatcher.swift`.

This is a simple SwiftUI `DynamicProperty`. Here is all we need:

```swift
//  Dispatcher.swift

import ImmutableData
import SwiftUI

@MainActor @propertyWrapper public struct Dispatcher<Store> : DynamicProperty where Store : ImmutableData.Dispatcher {
  @Environment private var store: Store
  
  public init(_ keyPath: WritableKeyPath<EnvironmentValues, Store>) {
    self._store = Environment(keyPath)
  }
  
  public var wrappedValue: some ImmutableData.Dispatcher<Store.State, Store.Action> {
    self.store
  }
}
```

We initialize our `Dispatcher` property wrapper with a `keyPath`. This `keyPath` should match the `keyPath` we use to initialize our `Provider` component. When our SwiftUI view component is ready to dispatch an Action value, the `wrappedValue` property returns an opaque `ImmutableData.Dispatcher`. We *could* return the `store` instance as a `wrappedValue` of the `Store` type, but we are then “leaking” unnecessary information about the `store` that our view component does not need to know about; this approach keeps our `Dispatcher` focused only on dispatching Action values.

Our `Dispatcher` type serves a similar role as the `useDispatch` hook from React Redux.[^3]

## Selector

Our `Provider` and `Dispatcher` were both quick to build. Our `Selector` type will be more complex. Before we start looking at code, let’s talk about our strategy and goals with this type.

We want our `Selector` to select a slice of our State and apply an arbitrary transform. For a contacts application, an example might be to return the count of all `Person` values. Another example might be to select all `Person` values and filter for names with a matching substring. Another example might be to select all `Person` values and sort by name.

Let’s begin with a definition:

* We define an *Output Selector* that takes a State as input and returns some data as an *Output*.

Since we expect our global state to transform many times over an application lifecycle, we would like for this `Selector` to rerun its Output Selector when the output might have changed. Since the only way for our `ImmutableData` architecture to transform State is from dispatching an Action, one strategy would be for our `Selector` to rerun its Output Selector every time an Action value is dispatched to the `Store`. Let’s think about the performance impact of this strategy.

Let’s start with a simple example: we plan to build a SwiftUI view component that displays the total number of people in our contacts application. If our `Person` instances are saved as values of a `Dictionary`, it is trivial for us to select the count of all `Person` instances. This operation is constant time: `O(1)`.

Here’s our next example: we plan to build a SwiftUI view component that filters for contact names with a matching substring. We can model this as an Output Selector that performs an amount of work linear with the amount of `Person` instances in our `Dictionary`: `O(n)`.

Here’s our next example: we plan to build a SwiftUI view component that sorts all contacts by name. We can model this Output Selector as an `O(n log n)` operation.

The operations that run in `O(n)` and `O(n log n)` are not trivial; let’s think about how we can try and optimize this work. Let’s think about the sorting operation. This operation runs in `O(n log n)` time. However, we can think of a way to help keep this operation from running more times than necessary. Suppose we run the sort algorithm at time `T(0)` across all `Person` instances. Suppose an Action is then dispatched to the `Store` at time `T(1)`. Should we then rerun the sort algorithm? Let’s think about this sort algorithm in terms of inputs and outputs. The input is a collection of `Person` instances. The output is a sorted array of `Person` instances. If this sort algorithm is a pure function with no side effects, then the output is stable and deterministic; if the input to our sort algorithm at time `T(0)` is the same as our input at time `T(1)`, our sorted array could not have changed. If the algorithm to determine if two inputs are equal runs in linear time (`O(n)`), the option to perform an equality check on our inputs might save us a lot of performance by sorting in `O(n log n)` only when necessary.

* We define a *Dependency Selector* that takes a State as input and returns some data as a *Dependency*.

Instead of rerunning our Output Selector on every Action that is dispatched to our `Store`, we can rerun our Dependency Selectors. If our `Selector` memoizes or caches this set of dependencies, and only reruns our Output Selector when one of those dependencies has changed, we can offer a lot of flexibility for product engineers to optimize performance.

The total set of all Action values in our application might be large, but only a small subset of those Action values might result in any mutation to our State that would cause the result of our Output Selector to ever change. If we rerun our Dependency Selectors on every Action that is dispatched, we are performing work that might be expensive (`O(n)` or greater) when we don’t really need to. Let’s give our `Selector` the ability to filter those Action values.

* We define a `Filter` that takes a State and an Action as input and returns a `Bool` as output indicating this State and Action should cause the Dependency Selectors to rerun.

Our Dependency Selectors and Filter are both powerful tools for product engineers to optimize performance, but we would also prefer for these to be optional. A product engineer should be able to create a `Selector` with no Dependency Selectors and no Filter; this would just rerun the Output Selector on every Action that is dispatched to the `Store`.

Now that we have some strategy written out, we can begin to build our `Selector` type. This will be the most complex type we have built so far, but we’ll go step-by-step and try to keep things as simple as possible.

Our `Selector` type serves a similar role as the `useSelector` hook from React Redux.[^4] Our use of memoization and dependency selectors serves a similar role as the Reselect library.[^5]

---

Our `Selector` type will use the `Streamer` protocol we defined in our last chapter for updating our view components as our State transforms over time. There are three types we will build:

* `Selector`: This is a `public` type for view components to select a slice of State.
* `Listener`: This is a `package` type for `Selector` to begin listening to `Streamer`.
* `AsyncListener`: This is an `internal` type for `Listener` to begin listening to `Streamer`.

Our `AsyncListener` is a “helper”; we could make `Listener` complex enough to listen to `Streamer` on its own, but we can keep our code a little more organized by keeping some work in `AsyncListener`.

Our first step is to define our `DependencySelector` and `OutputSelector` types. Add a new Swift file under `Sources/ImmutableUI`. Name this file `Selector.swift`.

Here are our first two types:

```swift
//  Selector.swift

public struct DependencySelector<State, Dependency> {
  let select: @Sendable (State) -> Dependency
  let didChange: @Sendable (Dependency, Dependency) -> Bool
  
  public init(
    select: @escaping @Sendable (State) -> Dependency,
    didChange: @escaping @Sendable (Dependency, Dependency) -> Bool
  ) {
    self.select = select
    self.didChange = didChange
  }
}

public struct OutputSelector<State, Output> {
  let select: @Sendable (State) -> Output
  let didChange: @Sendable (Output, Output) -> Bool
  
  public init(
    select: @escaping @Sendable (State) -> Output,
    didChange: @escaping @Sendable (Output, Output) -> Bool
  ) {
    self.select = select
    self.didChange = didChange
  }
}
```

Our `DependencySelector` type is initialized with a `select` closure, which returns a `Dependency` from a State, and a `didChange` closure, which returns `true` when two `Dependency` values have changed. In practice, we expect most product engineers to choose for their `Dependency` type to use value inequality (`!=`) to indicate when two `Dependency` values have changed. Our approach gives this infra the flexibility to handle more advanced use cases where product engineers need specialized control over the value returned from `didChange`.

Our `OutputSelector` type follows a similar pattern: a `select` closure, which returns an `Output` from a State, and a `didChange` closure, which returns `true` when two `Output` values have changed. Our `didChange` closure will be needed when we give our type the ability to use `Observable` for updating our view component graph.

Let’s turn our attention to `AsyncListener`. Add a new Swift file under `Sources/ImmutableUI`. Name this file `AsyncListener.swift`.

Our `AsyncListener` will use `Observable` to keep our view component graph updated. For a quick review of `Observable`, please read [SE-0395][^6]. Let’s begin with a small class just for `Observable`:

```swift
//  AsyncListener.swift

import Foundation
import ImmutableData
import Observation

@MainActor @Observable final fileprivate class Storage<Output> {
  var output: Output?
}
```

Let’s add our `AsyncListener` type:

```swift
//  AsyncListener.swift

@MainActor final class AsyncListener<State, Action, each Dependency, Output> where State : Sendable, Action : Sendable, repeat each Dependency : Sendable, Output : Sendable {
  private let label: String?
  private let filter: (@Sendable (State, Action) -> Bool)?
  private let dependencySelector: (repeat DependencySelector<State, each Dependency>)
  private let outputSelector: OutputSelector<State, Output>
  
  private var oldDependency: (repeat each Dependency)?
  private var oldOutput: Output?
  private let storage = Storage<Output>()
  
  init(
    label: String? = nil,
    filter isIncluded: (@Sendable (State, Action) -> Bool)? = nil,
    dependencySelector: repeat DependencySelector<State, each Dependency>,
    outputSelector: OutputSelector<State, Output>
  ) {
    self.label = label
    self.filter = isIncluded
    self.dependencySelector = (repeat each dependencySelector)
    self.outputSelector = outputSelector
  }
}
```

Our `AsyncListener` class is generic across these types:
* `State`: The State type defined by our product and our `Store`.
* `Action` The Action type defined by our product and our `Store`.
* `Dependency`: A parameter pack of types defined by our product.
* `Output`: A type defined by our product.

For a quick review of parameter packs and variadic types, please read [SE-0393][^7] and [SE-0398][^8].

Our `AsyncListener` class is initialized with these parameters:
* `label`: An optional `String` used for debugging and console logging.
* `filter`: An optional `Filter` closure.
* `dependencySelector`: A parameter pack of `DependencySelector` values. An empty pack indicates no dependencies are needed.
* `outputSelector`: An `OutputSelector` for displaying data in a view component.

Since we use this type for displaying data in a view component, we choose to isolate our type to `MainActor`.

In a future version of Swift, we might choose to make `AsyncListener` an `Observable` type and remove our `Storage` class. As of Swift 6.0, we are currently blocked on making a variadic type `Observable`; we use our `Storage` class as a workaround.[^9]

Here is the function for delivering an `Output` value to a view component:

```swift
//  AsyncListener.swift

extension AsyncListener {
  var output: Output {
    guard let output = self.storage.output else { fatalError("missing output") }
    return output
  }
}
```

Our `Storage` returns an optional `output`. Before we complete this chapter, we will see how we can guarantee that this will not be `nil` and why a `fatalError` is used to indicate programmer error.

Let’s add two helper functions we will use only for debugging purposes:

```swift
//  AsyncListener.swift

extension UserDefaults {
  fileprivate var isDebug: Bool {
    self.bool(forKey: "com.northbronson.ImmutableUI.Debug")
  }
}

fileprivate func address(of x: AnyObject) -> String {
  var result = String(
    unsafeBitCast(x, to: UInt.self),
    radix: 16
  )
  for _ in 0..<(2 * MemoryLayout<UnsafeRawPointer>.size - result.utf16.count) {
    result = "0" + result
  }
  return "0x" + result
}
```

We will need three more functions for working with our parameter pack values:

```swift
//  AsyncListener.swift

extension ImmutableData.Selector {
  @MainActor fileprivate func selectMany<each T>(_ select: repeat @escaping @Sendable (State) -> each T) -> (repeat each T) where repeat each T : Sendable {
    (repeat self.select(each select))
  }
}

fileprivate struct NotEmpty: Error {}

fileprivate func isEmpty<each Element>(_ element: repeat each Element) -> Bool {
  func _isEmpty<T>(_ t: T) throws {
    throw NotEmpty()
  }
  do {
    repeat try _isEmpty(each element)
  } catch {
    return false
  }
  return true
}

fileprivate struct DidChange: Error {}

fileprivate func didChange<each Element>(
  _ didChange: repeat @escaping @Sendable (each Element, each Element) -> Bool,
  lhs: repeat each Element,
  rhs: repeat each Element
) -> Bool {
  func _didChange<T>(_ didChange: (T, T) -> Bool, _ lhs: T, _ rhs: T) throws {
    if didChange(lhs, rhs) {
      throw DidChange()
    }
  }
  do {
    repeat try _didChange(each didChange, each lhs, each rhs)
  } catch {
    return true
  }
  return false
}
```

Our original `Selector` protocol from the previous chapter returned a slice of State for one closure defined by our product engineer. For our `DependencySelector` values, a product engineer has the option to request that multiple slices of state be used to determine whether or not a dispatched Action should rerun our `OutputSelector`. Our `selectMany` function makes it easy for us to pass a parameter pack of closures and return a parameter pack of slices of State.

We include a `isEmpty` function for confirming if our parameter pack contains no values. We include a `didChange` function for confirming if two parameter packs changed. We use `do-catch` statements, but we could also choose `for-in` and leverage parameter pack iteration for this function.[^10]

Let’s now add a function to update our cached `dependency` values:

```swift
//  AsyncListener.swift

extension AsyncListener {
  private func updateDependency(with store: some ImmutableData.Selector<State>) -> Bool {
#if DEBUG
    if let label = self.label,
       UserDefaults.standard.isDebug {
      print("[ImmutableUI][AsyncListener]: \(address(of: self)) Update Dependency: \(label)")
    }
#endif
    let dependency = store.selectMany(repeat (each self.dependencySelector).select)
    if let oldDependency = self.oldDependency {
      self.oldDependency = dependency
      return didChange(
        repeat (each self.dependencySelector).didChange,
        lhs: repeat each oldDependency,
        rhs: repeat each dependency
      )
    } else {
      self.oldDependency = dependency
      return true
    }
  }
}
```

We begin with an optional `print` statement that is available in debug builds. These statements can be very helpful during debugging so that product engineers can follow along to see how their view component graph is being updated as Action values are dispatched. We enable this with our `com.northbronson.ImmutableUI.Debug` flag passed to `UserDefaults`.

We pass every `select` closure to our `store` and select the current set of `Dependency` values. If we have already cached a previous set of `Dependency` values, we compare the two and return `true` if they have changed. If this is our first time selecting a set of `Dependency` values, we return `true`.

Let’s add a function to update our cached `Output` value:

```swift
//  AsyncListener.swift

extension AsyncListener {
  private func updateOutput(with store: some ImmutableData.Selector<State>) {
#if DEBUG
    if let label = self.label,
       UserDefaults.standard.isDebug {
      print("[ImmutableUI][AsyncListener]: \(address(of: self)) Update Output: \(label)")
    }
#endif
    let output = store.select(self.outputSelector.select)
    if let oldOutput = self.oldOutput {
      self.oldOutput = output
      if self.outputSelector.didChange(oldOutput, output) {
        self.storage.output = output
      }
    } else {
      self.oldOutput = output
      self.storage.output = output
    }
  }
}
```

Similar to our previous function, we begin with an optional `print` statement in debug builds for product engineers to follow along as Action values are dispatched.

We select our current `Output` value from `Store`. If we have already cached a previous `Output` value, we compare the two and set the new value on our `Storage` reference — which is `Observable` — if they changed. This will update our view components that are currently observing this `Selector`. If this is our first time selecting an `Output` value, we set the new value on our `Storage` reference.

Let’s add a new function to update these both together:

```swift
//  AsyncListener.swift

extension AsyncListener {
  func update(with store: some ImmutableData.Selector<State>) {
#if DEBUG
    if let label = self.label,
       UserDefaults.standard.isDebug {
      print("[ImmutableUI][AsyncListener]: \(address(of: self)) Update: \(label)")
    }
#endif
    if self.hasDependency {
      if self.updateDependency(with: store) {
        self.updateOutput(with: store)
      }
    } else {
      self.updateOutput(with: store)
    }
  }
}

extension AsyncListener {
  private var hasDependency: Bool {
    isEmpty(repeat each self.dependencySelector) == false
  }
}
```

If our product engineer provided a set of `DependencySelector` values, we update our cached set of `Dependency` values and then update our `Output` value if those `Dependency` values have changed. If our product engineer did not provide a set of `DependencySelector` values, we go ahead and update our `Output` value.

Our final step is one more function for performing these updates while listening to a stream of values:

```swift
//  AsyncListener.swift

extension AsyncListener {
  func listen<S>(
    to stream: S,
    with store: some ImmutableData.Selector<State>
  ) async throws where S : AsyncSequence, S : Sendable, S.Element == (oldState: State, action: Action) {
    if let filter = self.filter {
      for try await _ in stream.filter(filter) {
        self.update(with: store)
      }
    } else {
      for try await _ in stream {
        self.update(with: store)
      }
    }
  }
}
```

Our `listen` function takes a `stream` parameter which adopts `AsyncSequence`; the `Element` type returned from this `stream` is a tuple with a State and an Action. This matches the form of the `Element` returned by our `Streamer` protocol from our previous chapter.

If our product engineer provided a `Filter` closure, we can use the `AsyncSequence` protocol to filter its elements through this closure. We then call our `update` function for every element returned by this `stream`.

That’s all we need for `AsyncListener`. This type was complex, but will be one of the most important types we need for building a flexible infra that scales to multiple sample product applications. We have two more types to build before our infra is done, but those will be easier and more straight forward.

---

Our `Listener` type will act as a helper between our public `Selector` and our internal `AsyncListener`. We could have put all of the work from `AsyncListener` here in `Listener`, but our code will be a little more readable with this approach.

Add a new Swift file under `Sources/ImmutableUI`. Name this file `Listener.swift`.

```swift
//  Listener.swift

@MainActor final package class Listener<State, Action, each Dependency, Output> where State : Sendable, Action : Sendable, repeat each Dependency : Sendable, Output : Sendable {
  private var id: AnyHashable?
  private var label: String?
  private var filter: (@Sendable (State, Action) -> Bool)?
  private var dependencySelector: (repeat DependencySelector<State, each Dependency>)
  private var outputSelector: OutputSelector<State, Output>
  
  private weak var store: AnyObject?
  private var listener: AsyncListener<State, Action, repeat each Dependency, Output>?
  private var task: Task<Void, any Error>?
  
  package init(
    id: some Hashable,
    label: String? = nil,
    filter isIncluded: (@Sendable (State, Action) -> Bool)? = nil,
    dependencySelector: repeat DependencySelector<State, each Dependency>,
    outputSelector: OutputSelector<State, Output>
  ) {
    self.id = AnyHashable(id)
    self.label = label
    self.filter = isIncluded
    self.dependencySelector = (repeat each dependencySelector)
    self.outputSelector = outputSelector
  }
  
  package init(
    label: String? = nil,
    filter isIncluded: (@Sendable (State, Action) -> Bool)? = nil,
    dependencySelector: repeat DependencySelector<State, each Dependency>,
    outputSelector: OutputSelector<State, Output>
  ) {
    self.id = nil
    self.label = label
    self.filter = isIncluded
    self.dependencySelector = (repeat each dependencySelector)
    self.outputSelector = outputSelector
  }
  
  deinit {
    self.task?.cancel()
  }
}
```

There’s a lot of code, but it’s relatively straight forward. Let’s compare the `Listener` to `AsyncListener`:

Both types are generic across these types:
* `State`: The State type defined by our product and our `Store`.
* `Action` The Action type defined by our product and our `Store`.
* `Dependency`: A parameter pack of types defined by our product.
* `Output`: A type defined by our product.

Both types are initialized with these parameters:
* `label`: An optional `String` used for debugging and console logging.
* `filter`: An optional `Filter` closure.
* `dependencySelector`: A parameter pack of `DependencySelector` values (which could be empty).
* `outputSelector`: An `OutputSelector` for displaying data in a view component.

The difference is that our `Listener` accepts an optional `id` parameter. We will see in our next steps where this is needed.

Here is the function for delivering an `Output` value to a view component:

```swift
//  Listener.swift

extension Listener {
  package var output: Output {
    guard let output = self.listener?.output else { fatalError("missing output") }
    return output
  }
}
```

Similar to our approach in `AsyncListener`, we use a `fatalError` to indicate a programmer error when our `AsyncListener` instance has not been created. Our `Selector` will show us how we can guarantee the `listener` property has been created before we request `output`.

Over the course of our app lifecycle, a `Listener` instance might need to rebuild its `AsyncListener` instance. For example, a product engineer might decide at runtime that the `outputSelector` would need to change. We might display a product detail page which selects a `Product` instance from our State with a `productId` value. If that `productId` value changes as the user selects a new product, we need a way to rebuild our `AsyncListener`: the bug would be if we continue to run our `outputSelector` with the “stale” `productId`. Our `id` property will serve a similar purpose to the `id` value available to `SwiftUI.View`.[^11]

Here are two methods to update our `Listener` if an `id` has changed:

```swift
//  Listener.swift

extension Listener {
  package func update(
    id: some Hashable,
    label: String? = nil,
    filter isIncluded: (@Sendable (State, Action) -> Bool)? = nil,
    dependencySelector: repeat DependencySelector<State, each Dependency>,
    outputSelector: OutputSelector<State, Output>
  ) {
    let id = AnyHashable(id)
    if self.id != id {
      self.id = id
      self.label = label
      self.filter = isIncluded
      self.dependencySelector = (repeat each dependencySelector)
      self.outputSelector = outputSelector
      self.store = nil
    }
  }
}

extension Listener {
  package func update(
    label: String? = nil,
    filter isIncluded: (@Sendable (State, Action) -> Bool)? = nil,
    dependencySelector: repeat DependencySelector<State, each Dependency>,
    outputSelector: OutputSelector<State, Output>
  ) {
    if self.id != nil {
      self.id = nil
      self.label = label
      self.filter = isIncluded
      self.dependencySelector = (repeat each dependencySelector)
      self.outputSelector = outputSelector
      self.store = nil
    }
  }
}
```

If our `id` has changed, we then remove the reference to the previous `store` instance we were listening to. If our `id` is consistent, no action is taken. We don’t rebuild our `AsyncListener` instance here; this will be the next step:

```swift
//  Listener.swift

import ImmutableData

extension Listener {
  package func listen(to store: some ImmutableData.Selector<State> & ImmutableData.Streamer<State, Action> & AnyObject) {
    if self.store !== store {
      self.store = store
      
      let listener = AsyncListener<State, Action, repeat each Dependency, Output>(
        label: self.label,
        filter: self.filter,
        dependencySelector: repeat each self.dependencySelector,
        outputSelector: self.outputSelector
      )
      listener.update(with: store)
      self.listener = listener
      
      let stream = store.makeStream()
      
      self.task?.cancel()
      self.task = Task {
        try await listener.listen(
          to: stream,
          with: store
        )
      }
    }
  }
}
```

Our `listen` function might be called many times over an app lifecycle. If the identity of the `store` is consistent, we do not need to do any work; we continue listening to the same `store`.

We pass our optional `label`, optional `filter`, `dependencySelector`, and `outputSelector` to a new `AsyncListener` instance and give it our `store` to set its initial state with `update`. We then begin a new `stream` and pass that to our `AsyncListener` instance in a new `Task`.

---

Here’s the final type we need for our infra. Add a new Swift file under `Sources/ImmutableUI`. Name this file `Selector.swift`.

```swift
//  Selector.swift

import ImmutableData
import SwiftUI

@MainActor @propertyWrapper public struct Selector<Store, each Dependency, Output> where Store : ImmutableData.Selector, Store : ImmutableData.Streamer, Store : AnyObject, repeat each Dependency : Sendable, Output : Sendable {
  @Environment private var store: Store
  @State private var listener: Listener<Store.State, Store.Action, repeat each Dependency, Output>
  
  private let id: AnyHashable?
  private let label: String?
  private let filter: (@Sendable (Store.State, Store.Action) -> Bool)?
  private let dependencySelector: (repeat DependencySelector<Store.State, each Dependency>)
  private let outputSelector: OutputSelector<Store.State, Output>
  
  public init(
    _ keyPath: WritableKeyPath<EnvironmentValues, Store>,
    id: some Hashable,
    label: String? = nil,
    filter isIncluded: (@Sendable (Store.State, Store.Action) -> Bool)? = nil,
    dependencySelector: repeat DependencySelector<Store.State, each Dependency>,
    outputSelector: OutputSelector<Store.State, Output>
  ) {
    self._store = Environment(keyPath)
    self.listener = Listener(
      id: id,
      label: label,
      filter: isIncluded,
      dependencySelector: repeat each dependencySelector,
      outputSelector: outputSelector
    )
    self.id = AnyHashable(id)
    self.label = label
    self.filter = isIncluded
    self.dependencySelector = (repeat each dependencySelector)
    self.outputSelector = outputSelector
  }
  
  public init(
    _ keyPath: WritableKeyPath<EnvironmentValues, Store>,
    label: String? = nil,
    filter isIncluded: (@Sendable (Store.State, Store.Action) -> Bool)? = nil,
    dependencySelector: repeat DependencySelector<Store.State, each Dependency>,
    outputSelector: OutputSelector<Store.State, Output>
  ) {
    self._store = Environment(keyPath)
    self.listener = Listener(
      label: label,
      filter: isIncluded,
      dependencySelector: repeat each dependencySelector,
      outputSelector: outputSelector
    )
    self.id = nil
    self.label = label
    self.filter = isIncluded
    self.dependencySelector = (repeat each dependencySelector)
    self.outputSelector = outputSelector
  }
  
  public var wrappedValue: Output {
    self.listener.output
  }
}
```

This code should look very similar to what we built for our `Listener` type, with the exception of the `keyPath` parameter for reading a `Store` from `SwiftUI.Environment`. Our `Selector` is a property wrapper; we return the `Output` from `Listener` for our `wrappedValue`.

Selecting an `Output` value from our `Store` can be expensive work. SwiftUI view components are intended to be lightweight; the system might create and dispose your view component many times over the course of an app lifecycle. You should always try to avoid performing expensive work when a view component is initialized or a `body` is computed.[^12] While the value of the view component might be short-lived, the “lifetime” of the view is tied to its identity.[^13] By saving our `Listener` instance as a `SwiftUI.State` variable, we then tie the lifetime of our `Listener` instance to the lifetime of the identity of the view. If our `Output` is an expensive operation — like an `O(n log n)` sorting operation — we can cache the result of that operation even while the value of our view component is created many times by the system.

This is an important optimization, but then raises a different problem: if the lifetime of our `Listener` instance is tied to the lifetime of the identity of the view, then how do we update our `Listener` instance when something changed in how it should compute its `Output`? Let’s think back to our example of a product detail page which selects a `Product` instance with a `productId`. If the user selects a different `productId`, we want to be sure that our `Output` now reflects the correct `Product` instance — not the old one.

We do have the option to set a new `id` on our view component like we saw from SwiftUI Lab; this would also refresh *all* the component state, which might not always be what we want.

To offer product engineers flexibility and control, we create a `Selector` instance with an optional `id`. As we saw in `Listener`, changing the `id` will rerun the Dependency Selectors and Output Selector with the new values from our product engineer.

We’re missing one more step: before a view component is about to compute its `body`, we want our `Selector` to make sure the `Listener` has the current parameters from our product engineer and is ready to return an `Output`. Similar to an approach from Dave DeLong,[^14] we use a `SwiftUI.DynamicProperty`.[^15] The SwiftUI infra will run our `update` function before every `body` is computed. Here is what that looks like for us:

```swift
//  Selector.swift

extension Selector: @preconcurrency DynamicProperty {
  public mutating func update() {
    if let id = self.id {
      self.listener.update(
        id: id,
        label: self.label,
        filter: self.filter,
        dependencySelector: repeat each self.dependencySelector,
        outputSelector: self.outputSelector
      )
    } else {
      self.listener.update(
        label: self.label,
        filter: self.filter,
        dependencySelector: repeat each self.dependencySelector,
        outputSelector: self.outputSelector
      )
    }
    self.listener.listen(to: self.store)
  }
}
```

As of this writing, `DynamicProperty` is not explicitly isolated to `MainActor` by SwiftUI. To work around this from Swift 6.0 and compile with Strict Concurrency Checking, we adopt the `preconcurrency` workaround from [SE-0423][^16].

The first step is to pass the current parameters (`id`, `label`, `filter`, `dependencySelector`, and `outputSelector`) from our product engineer to our `Listener` instance. Then, our `Listener` can `listen` to our `store`.

Because SwiftUI guarantees the `update` will be called before `body` is computed, we can guarantee that our `Listener` has already called `update` and `listen` before `body` is computed. This is how we can use `fatalError` in `AsyncListener` if `Output` is missing to indicate a programmer error; `Output` will never be missing at the time `body` is being computed.

One future optimization we might be interested in is to reduce the amount of `Listener` instances that are allocated and disposed. Every `Selector` instance currently allocates a new `Listener` instance,[^17] but these are then disposed if we already have a “long-lived” `Listener` instance that is tied to the view component identity. Our `Listener` initializer is lightweight by design; the performance we lose from this approach is a tradeoff we make to keep the code moving forward and simple for the first version of our infra. If the extra object allocations do affect performance in measurable amounts, we always have the option to revisit this code in the future.

---

Here is our `ImmutableUI` package, including the tests available on our `chapter-02` branch:

```text
ImmutableUI
├── Sources
│   └── ImmutableUI
│       ├── AsyncListener.swift
│       ├── Dispatcher.swift
│       ├── Listener.swift
│       ├── Provider.swift
│       └── Selector.swift
└── Tests
    └── ImmutableUITests
        └── ListenerTests.swift
```

Our infra is now in-place and we are ready to begin building sample application products. Some of the work we completed so far might seem a little abstract without seeing how product engineers put this to work in a production application. Building three sample application products together from this infra should help to demonstrate how to “think in `ImmutableData`” when it’s time to bring the `ImmutableData` architecture to your own products.

[^1]: https://developer.apple.com/documentation/swiftui/environmentvalues/modelcontext
[^2]: https://react-redux.js.org/api/provider
[^3]: https://react-redux.js.org/api/hooks#usedispatch
[^4]: https://react-redux.js.org/api/hooks#useselector
[^5]: https://redux.js.org/usage/deriving-data-selectors#writing-memoized-selectors-with-reselect
[^6]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0395-observability.md
[^7]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0393-parameter-packs.md
[^8]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0398-variadic-types.md
[^9]: https://github.com/swiftlang/swift/issues/73690
[^10]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0408-pack-iteration.md
[^11]: https://swiftui-lab.com/swiftui-id/
[^12]: https://developer.apple.com/videos/play/wwdc2020/10040
[^13]: https://developer.apple.com/videos/play/wwdc2021/10022
[^14]: https://davedelong.com/blog/2021/04/03/core-data-and-swiftui/
[^15]: https://developer.apple.com/documentation/swiftui/dynamicproperty
[^16]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0423-dynamic-actor-isolation.md
[^17]: https://developer.apple.com/documentation/swiftui/state#Store-observable-objects
