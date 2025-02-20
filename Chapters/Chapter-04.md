# CounterUI

Our Counter sample application product is almost complete. Let’s turn our attention to the SwiftUI view component tree. Let’s review the three components we are going to display for the user: an Increment button, a Decrement button, and a component to display the current value. To display the current value, we will use `ImmutableUI.Selector`. To mutate the current value, we will dispatch actions using `ImmutableUI.Dispatcher`. To pass a `Store` through our component tree, we will use `ImmutableUI.Provider`.

## StoreKey

The components we built in the `ImmutableUI` module use `SwiftUI.Environment` to read a `Store`. What we didn’t define in `ImmutableUI` is what key path we use to set this `Store` on `SwiftUI.Environment`. This decision we left to product engineers. Since we are now product engineers, let’s start by defining a key that will be used across our application to save this `Store`.

Select the `CounterUI` package and add a new Swift file under `Sources/CounterUI`. Name this file `StoreKey.swift`.

```swift
//  StoreKey.swift

import CounterData
import ImmutableData
import ImmutableUI
import SwiftUI

@MainActor fileprivate struct StoreKey : @preconcurrency EnvironmentKey {
  static let defaultValue = ImmutableData.Store(
    initialState: CounterState(),
    reducer: CounterReducer.reduce
  )
}
```

Our `StoreKey` adopts `EnvironmentKey`. Our `defaultValue` creates a new `Store` instance with `CounterState` as the `Store.State` and `CounterAction` as the `Store.Action`.

We are required to provide a `defaultValue` for `EnvironmentKey`,[^1] but our application will provide its own instance through `ImmutableUI.Provider`. The instance we provide through `ImmutableUI.Provider` will then be the `Store` available to `ImmutableUI.Selector` and `ImmutableUI.Dispatcher`. If you wish to enforce that the `defaultValue` is not appropriate to use through `ImmutableUI.Selector` and `ImmutableUI.Dispatcher`, you could also choose to pass a custom `reduce` function that crashes with a `fatalError` to indicate a programmer error.

To compile from Swift 6.0 and Strict Concurrency Checking, we adopt `preconcurrency` from [SE-0423][^2].

Our next step is an extension on `EnvironmentValues`:

```swift
//  StoreKey.swift

extension EnvironmentValues {
  fileprivate var store: ImmutableData.Store<CounterState, CounterAction> {
    get {
      self[StoreKey.self]
    }
    set {
      self[StoreKey.self] = newValue
    }
  }
}
```

When building from Xcode 16.0, we have the option to save ourselves some time with the `Entry` macro. As of this writing, this seems to lead to our `defaultValue` being created on demand many times over our app lifecycle.[^3] For now, we build these the old-fashioned way to save our `defaultValue` as a stored type property: it’s created just once.

We can update our types from our `ImmutableUI` infra to take advantage of `StoreKey`. Our `ImmutableUI` types take an `EnvironmentValues` key path to a `Store` as a parameter. As you can imagine, it would be tedious to have to pass the same exact key path in every place we need to call one of these types. Since our product will use the same key path across our application life cycle, we can update the types from `ImmutableUI` to use this key path without us having to pass it as a parameter. Let’s begin with our `ImmutableUI.Provider`:

```swift
//  StoreKey.swift

extension ImmutableUI.Provider {
  public init(
    _ store: Store,
    @ViewBuilder content: () -> Content
  ) where Store == ImmutableData.Store<CounterState, CounterAction> {
    self.init(
      \.store,
       store,
       content: content
    )
  }
}
```

This extension on `ImmutableUI.Provider` adds a new initializer. The original initializer accepts an `EnvironmentValues` key path as a parameter. Since we defined our `StoreKey` on the expectation that our application uses this key path everywhere across the component tree, we can define this new initializer that uses our `Store` key path every time. This also allows us to keep our `StoreKey` defined as `fileprivate` while still defining a `public` initializer on `ImmutableUI.Provider` that uses our `Store` key path.

Let’s continue with a new initializer for `ImmutableUI.Dispatcher`:

```swift
//  StoreKey.swift

extension ImmutableUI.Dispatcher {
  public init() where Store == ImmutableData.Store<CounterState, CounterAction> {
    self.init(\.store)
  }
}
```

Let’s finish with our `ImmutableUI.Selector` initializers:

```swift
//  StoreKey.swift

extension ImmutableUI.Selector {
  public init(
    id: some Hashable,
    label: String? = nil,
    filter isIncluded: (@Sendable (Store.State, Store.Action) -> Bool)? = nil,
    dependencySelector: repeat DependencySelector<Store.State, each Dependency>,
    outputSelector: OutputSelector<Store.State, Output>
  ) where Store == ImmutableData.Store<CounterState, CounterAction> {
    self.init(
      \.store,
       id: id,
       label: label,
       filter: isIncluded,
       dependencySelector: repeat each dependencySelector,
       outputSelector: outputSelector
    )
  }
}

extension ImmutableUI.Selector {
  public init(
    label: String? = nil,
    filter isIncluded: (@Sendable (Store.State, Store.Action) -> Bool)? = nil,
    dependencySelector: repeat DependencySelector<Store.State, each Dependency>,
    outputSelector: OutputSelector<Store.State, Output>
  ) where Store == ImmutableData.Store<CounterState, CounterAction> {
    self.init(
      \.store,
       label: label,
       filter: isIncluded,
       dependencySelector: repeat each dependencySelector,
       outputSelector: outputSelector
    )
  }
}
```

Each sample application product will be required to provide its own key path to access a `Store` instance from `EnvironmentValues`. While we are not required to build these new initializers, this is the convention we follow for our sample application products through this tutorial. You are welcome to follow this convention for your own sample application products.

## Dispatch

Our component tree will need to dispatch an Action when the user taps a button. We will use the `ImmutableUI.Dispatcher` type to dispatch this action. When we built the `ImmutableUI.Dispatcher`, we returned an opaque `ImmutableData.Dispatcher` as a `wrappedValue`. When we built the `ImmutableData.Dispatcher` protocol, we included a function for dispatching “thunk” closures. We will see in our next sample application product how we use these thunks to dispatch async operations.

While we *could* use `ImmutableUI.Dispatcher` to dispatch thunk operations directly from our component tree, we will see that our recommended approach for that will be to use Listeners in our data model layer.

Let’s build a new property wrapper just for our product. This property wrapper will make use of `ImmutableUI.Dispatcher`, but will only give product engineers the ability to dispatch Action values.

Add a new Swift file under `Sources/CounterUI`. Name this file `Dispatch.swift`.

```swift
//  Dispatch.swift

import CounterData
import ImmutableData
import ImmutableUI
import SwiftUI

@MainActor @propertyWrapper struct Dispatch : DynamicProperty {
  @ImmutableUI.Dispatcher() private var dispatcher
  
  init() {
    
  }
  
  var wrappedValue: (CounterAction) throws -> () {
    self.dispatcher.dispatch
  }
}
```

For our `wrappedValue`, we return the `dispatch` function from our `dispatcher`. We do not give product engineers the option to dispatch a thunk operation from components with this type.

Product engineers are not required to build their own `Dispatch` property wrapper, but we do recommend most product engineers follow this pattern.

## Select

Our component tree will need to display the current value. We will use the `ImmutableUI.Selector` type to select this value from our State. The `ImmutableUI.Selector` offers a lot of power and flexibility to product engineers to customize the behavior for their product, but all this flexibility might not always be necessary or desirable. A “simpler” API would reduce the friction on product engineers. Let’s see an example of how we can customize the behavior of `ImmutableUI.Selector` to help improve the developer experience for our product engineers building the Counter application.

The `ImmutableUI.DependencySelector` and `ImmutableUI.OutputSelector` give product engineers the ability to define the slices of State they need to be their dependencies and output. These types also offer product engineers the ability to define their own `didChange` closures for indicating two values have changed. When two `Dependency` values have changed, we compute a new `Output` instance. When two `Output` values have changed, we use `Observable` to compute a new `body` in our component.

The ability for product engineers to have control over what logic is used to determine when two `Dependency` or `Output` values have changed is very powerful, but can introduce “ceremony” throughout our product. What our product engineers usually want is the ability to define some “default” behavior to save time from writing repetitive boilerplate code. Let’s see how we can build this for our product.

Add a new Swift file under `Sources/CounterUI`. Name this file `Select.swift`.

```swift
//  Select.swift

import CounterData
import ImmutableData
import ImmutableUI
import SwiftUI

extension ImmutableUI.DependencySelector {
  init(select: @escaping @Sendable (State) -> Dependency) where Dependency : Equatable {
    self.init(select: select, didChange: { $0 != $1 })
  }
}

extension ImmutableUI.OutputSelector {
  init(select: @escaping @Sendable (State) -> Output) where Output : Equatable {
    self.init(select: select, didChange: { $0 != $1 })
  }
}
```

Our first step is to add some default behavior to `ImmutableUI.DependencySelector` and `ImmutableUI.OutputSelector`. These new initializers are available when `Dependency` and `Output` adopt `Equatable`. For this product, we decided that a reasonable default behavior is that we want to use value equality to determine whether or not two values have changed. Rather than require product engineers to pass this value equality operator to *every* `ImmutableUI.DependencySelector` and `ImmutableUI.OutputSelector`, we build two new initializers that pass those value equality operators for us.

Now that we have these new initializers, we can also simplify the initializers on `ImmutableUI.Selector`:

```swift
//  Select.swift

extension ImmutableUI.Selector {
  init(
    id: some Hashable,
    label: String? = nil,
    filter isIncluded: (@Sendable (Store.State, Store.Action) -> Bool)? = nil,
    dependencySelector: repeat @escaping @Sendable (Store.State) -> each Dependency,
    outputSelector: @escaping @Sendable (Store.State) -> Output
  ) where Store == ImmutableData.Store<CounterState, CounterAction>, repeat each Dependency : Equatable, Output : Equatable {
    self.init(
      id: id,
      label: label,
      filter: isIncluded,
      dependencySelector: repeat DependencySelector(select: each dependencySelector),
      outputSelector: OutputSelector(select: outputSelector)
    )
  }
}

extension ImmutableUI.Selector {
  init(
    label: String? = nil,
    filter isIncluded: (@Sendable (Store.State, Store.Action) -> Bool)? = nil,
    dependencySelector: repeat @escaping @Sendable (Store.State) -> each Dependency,
    outputSelector: @escaping @Sendable (Store.State) -> Output
  ) where Store == ImmutableData.Store<CounterState, CounterAction>, repeat each Dependency : Equatable, Output : Equatable {
    self.init(
      label: label,
      filter: isIncluded,
      dependencySelector: repeat DependencySelector(select: each dependencySelector),
      outputSelector: OutputSelector(select: outputSelector)
    )
  }
}
```

Now, we don’t have to specify a value equality operator when creating a `ImmutableUI.Selector` instance; our product defines its own default behavior which is appropriate for the domain of this product.

Not every product engineer is going to want to use these defaults; some product engineers might need the flexibility and control of the original initializers. We expect that value equality would be a reasonable default behavior for most product engineers and this is the convention we follow in our sample application products. You are welcome to follow this convention for your own sample application products.

In our previous chapter, we defined the `selectValue` function to select and return our `value` from our `CounterState`. This will be the `outputSelector` we pass when creating `ImmutableUI.Selector` for displaying the value in our component tree. In complex products, we might need to display the same slice of State across multiple view component subgraphs. To help make things easier for product engineers and reduce the amount of code that needs to be duplicated, we will define custom Selector types for our products. Each Selector type knows how to select one specific slice of State and return that slice to a view component.

Here is our custom Selector:

```swift
//  Select.swift

@MainActor @propertyWrapper struct SelectValue : DynamicProperty {
  @ImmutableUI.Selector(outputSelector: CounterState.selectValue()) var wrappedValue
  
  init() {
    
  }
}
```

Our sample application product will only display `value` in one place; but this is good practice for us before we move on to more complex products. We will follow this convention throughout our sample application products. We encourage you to follow this convention in your own products.

Our use of custom Selector types serves a similar role as custom hooks in React.[^4]

## Content

You might expect that building our view component tree would be a lot of work. The truth is, most of the “heavy lifting” has already been completed; this was by design. For the most part, learning the `ImmutableData` architecture does not mean learning SwiftUI all over again. Most of our work building component trees will look and feel very familiar to what you already know; this was also by design.

The biggest philosophical difference you must embrace is transforming user events to Action values. This is what we mean by *thinking declaratively*. Instead of performing an imperative mutation on global state when a button is tapped, we dispatch an Action to our `Store`.

Let’s start by building our component. Add a new Swift file under `Sources/CounterUI`. Name this file `Content.swift`. Here is the declaration:

```swift
//  Content.swift

import CounterData
import ImmutableData
import ImmutableUI
import SwiftUI

@MainActor public struct Content {
  @SelectValue private var value
  
  @Dispatch private var dispatch
  
  public init() {
    
  }
}
```

Our `Content` uses the property wrappers we built earlier in this chapter. The `Dispatch` wrapper returns our `dispatch` function. The `SelectValue` wrapper returns our `value`.

Our next step is two helper functions for dispatching Action values:

```swift
//  Content.swift

extension Content {
  private func didTapIncrementButton() {
    do {
      try self.dispatch(.didTapIncrementButton)
    } catch {
      print(error)
    }
  }
}

extension Content {
  private func didTapDecrementButton() {
    do {
      try self.dispatch(.didTapDecrementButton)
    } catch {
      print(error)
    }
  }
}
```

Our `dispatch` function can throw an error. For this product, we `print` that `error` to our console. A discussion on error handling for SwiftUI is orthogonal to our goal of teaching the `ImmutableData` architecture. In your own products, you might choose to display an alert to the user. For our tutorial, we `print` to console in the interest of time to keep things concentrated on `ImmutableData` as much as possible. If you did choose to display an alert to the user, the state to manage that alert could be saved as “component” state using `SwiftUI.State`; you don’t need to rethink how your global state is built.

Let’s turn our attention to the component tree:

```swift
//  Content.swift

extension Content : View {
  public var body: some View {
    VStack {
      Button("Increment") {
        self.didTapIncrementButton()
      }
      Text("Value: \(self.value)")
      Button("Decrement") {
        self.didTapDecrementButton()
      }
    }
    .frame(
      minWidth: 256,
      minHeight: 256
    )
  }
}

#Preview {
  Content()
}
```

In macOS 15.0, there seems to be a known issue in `SwiftUI.Stepper` that is causing unexpected behavior when using `Observable`.[^5] We can work around this with a custom component.

You can now see this component tree live from Xcode in `Preview`. This is actually the first time we’ve seen our `ImmutableData` in action from a live demo. Pretty cool! Tapping the Increment Button dispatches the `didTapIncrementButton` value to our `Store`. Tapping the Decrement Button performs the inverse operation and dispatches the `didTapDecrementButton` value to our `Store`. Our `CounterReducer` transforms our `CounterState` to increment our `value`. Our component tree is updated when `value` changes because we built our `ImmutableUI.Selector` infra on `Observable`.

The `Content` component from `Preview` is using the `StoreKey.defaultValue` we defined earlier in this chapter. For your products, you might choose to keep `StoreKey.defaultValue` as a legit option that product engineers can choose to use from `Preview`. In the products through this tutorial, we will always use `Provider` to set one `Store` instance on `Environment` at the root level of the component tree.

The `Preview` macro will be very helpful to us as we build our component trees. Let’s see an example of how we can have more control over the `Store` instance used in `Preview`.

As we previously discussed, you might choose to pass a Reducer function to your `StoreKey.defaultValue` instance that crashes with a `fatalError` to indicate programmer error: product engineers should always use the `Store` instance that is passed through `Provider`. Let’s see an example of a Preview that passes a `Store` through `Provider`:

```swift
//  Content.swift

#Preview {
  @Previewable @State var store = ImmutableData.Store(
    initialState: CounterState(),
    reducer: CounterReducer.reduce
  )
  
  Provider(store) {
    Content()
  }
}
```

Our `CounterReducer.reduce` function does not throw errors, but we still perform a `do-catch` statement in `Content` because the `ImmutableData.Dispatcher` protocol says the `dispatch` function may choose to throw. Our `Content` component logs these errors to console, but we can’t see these errors in our app; they don’t exist.

Let’s add a custom Preview just for tracking our error logging:

```swift
//  Content.swift

fileprivate struct CounterError : Swift.Error {
  let state: CounterState
  let action: CounterAction
}

#Preview {
  @Previewable @State var store = ImmutableData.Store(
    initialState: CounterState(),
    reducer: { (state: CounterState, action: CounterAction) -> (CounterState) in
      throw CounterError(
        state: state,
        action: action
      )
    }
  )
  
  Provider(store) {
    Content()
  }
}
```

Instead of creating a `Store` instance with `CounterReducer.reduce`, we define a custom Reducer function that *does* throw errors. When we run this Preview from Xcode, we can confirm that errors are printed to console when our Button components are tapped.

Passing custom reducers to a Preview can be a legit strategy to improve testability, but our advice is to try and save this technique for special circumstances. For this example, we build a custom Reducer because our production Reducer never throws. We make sure to include a Preview with our production Reducer; we want product engineers to have the ability to test against the same Reducer our users will see in the final product.

---

Here is our `CounterUI` package:

```text
CounterUI
└── Sources
    └── CounterUI
        ├── Content.swift
        ├── Dispatch.swift
        ├── Select.swift
        └── StoreKey.swift
```

Unlike our previous chapters, we don’t have a very strong opinion on unit testing your view component tree built from `ImmutableData`. We consider unit testing SwiftUI to be orthogonal to our goal of teaching the `ImmutableData` architecture. For this tutorial, we prefer “integration-style” testing using `Preview`. If you are interested in learning more about unit testing for SwiftUI, we recommend following Jon Reid and Quality Coding to learn what might be possible for your products.[^6]

[^1]: https://developer.apple.com/documentation/swiftui/environmentkey/
[^2]: https://github.com/apple/swift-evolution/blob/main/proposals/0423-dynamic-actor-isolation.md
[^3]: https://developer.apple.com/forums/thread/770298
[^4]: https://react.dev/learn/reusing-logic-with-custom-hooks
[^5]: https://developer.apple.com/forums/thread/763442
[^6]: https://qualitycoding.org
