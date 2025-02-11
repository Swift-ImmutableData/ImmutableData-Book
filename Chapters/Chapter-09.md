# Animals.app

Our `AnimalsData` and `AnimalsUI` modules are complete. We just need an `App` to tie everything together.

Select `Animals.xcodeproj` and open `AnimalsApp.swift`. We can delete the original “Hello World” template. Let’s begin by defining our `AnimalsApp` type:

```swift
//  AnimalsApp.swift

import AnimalsData
import AnimalsUI
import ImmutableData
import ImmutableUI
import SwiftUI

@main @MainActor struct AnimalsApp {
  @State private var store = Store(
    initialState: AnimalsState(),
    reducer: AnimalsReducer.reduce
  )
  @State private var listener = Listener(store: Self.makeLocalStore())
  
  init() {
    self.listener.listen(to: self.store)
  }
}
```

We construct our `AnimalsApp` with a `Store` and a `Listener`. Our `Store` is constructed with `AnimalsState` and `AnimalsReducer`. Our `Listener` will be constructed with a `LocalStore` as its `PersistentStore`. We pass our `Store` to our `Listener`; when action values are dispatched to our `Store` and our `AnimalsReducer` returns, this `Listener` will perform asynchronous side effects.

Here is where we construct our `LocalStore`:

```swift
//  AnimalsApp.swift

extension AnimalsApp {
  private static func makeLocalStore() -> LocalStore<UUID> {
    do {
      return try LocalStore<UUID>()
    } catch {
      fatalError("\(error)")
    }
  }
}
```

If we fail to construct a `LocalStore`, we crash our sample product. A full discussion about how and why SwiftData might fail to initialize is outside the scope of this tutorial. You can explore the documentation and determine for yourself if your own products should perform any work here to “fail gracefully” and inform the user about what just happened.

Our final step is `body`:

```swift
//  AnimalsApp.swift

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

We can now build and run (`⌘ R`). Our application is now a working clone of the original Animals sample application from Apple. We preserved the core functionality: `Animal` values can be created, edited, and deleted. We also save our global state on our filesystem in a persistent database. You can also create a new window (`⌘ N`) and watch as edits from one window are reflected in the other.

Both applications are built from SwiftUI, and both application leverage SwiftData to manage their persistent database. The big difference here is that the original sample application built SwiftUI components that respond to user events with imperative logic to mutate SwiftData model objects. Our new sample application built SwiftUI components that respond to user events with declarative logic; the imperative logic to mutate SwiftData model objects is abstracted out of our component tree. Instead of passing mutable model objects to our component tree — and living with the unpredictable nature of shared mutable state — we pass immutable model values — our component tree can’t mutate shared state unpredictably.

Let’s experiment with some of the tools we built for improved debugging. Inspired by Antoine van der Lee, we’re going to leverage launch arguments from Xcode to enable some of the `print` statements we built earlier.[^1] Let’s begin with our `AnimalsUI` module. We added a `debugPrint` function that we call when components construct a `body` property. Let’s add a launch argument to our scheme to see how this looks:

```text
-com.northbronson.AnimalsUI.Debug 1
```

When we enable this argument and run our application, we now see logging from SwiftUI when our `body` properties are built:

```text
Content: @self, @identity, _selectedCategoryId, _selectedAnimalId changed.
CategoryList: @self, @identity, _selectedCategoryId changed.
CategoryList.Container: @self, @identity, _categories, @16, _status, @184, _selectedCategoryId, _dispatch changed.
CategoryList.Presenter: @self, @identity, _isAlertPresented, _selectedCategoryId changed.
AnimalList.Container: @self, @identity, _animals, @16, _category, @184, _status, @320, _selectedAnimalId, _dispatch changed.
AnimalList.Presenter: @self, @identity, _isSheetPresented, _selectedAnimalId changed.
AnimalDetail: @self changed.
AnimalDetail.Container: @self, @identity, _animal, @16, _category, @152, _status, @288, _dispatch changed.
AnimalDetail.Presenter: @self, @identity, _isAlertPresented, _isSheetPresented changed.
CategoryList.Container: \Storage<Optional<Status>>.output changed.
CategoryList.Presenter: @self changed.
CategoryList.Container: \Storage<Array<Category>>.output changed.
CategoryList.Presenter: @self changed.
```

While we perform user events and transform our global state, we can watch as our component tree is rebuilt. To optimize for performance, we can choose to reduce the amount of unnecessary time spent computing `body` properties. Our `ImmutableData` architecture helps us here: our component tree is built against Selectors that return data scoped to the needs of our component. Our Selectors use Filters and Dependencies to reduce the amount of times we then return new data to our component.

If we select the `Fish` category and add a new `Animal` value to the `Mammal` category, we *do not* see our `AnimalList` component compute its `body` again. If we select the `Fish` category and add a new `Animal` values to the `Fish` category, we *do* see our `AnimalList` component compute its `body` again.

Let’s enable the logging we built in `AnimalsData`. Here is our new launch argument:

```text
-com.northbronson.AnimalsData.Debug 1
```

When we enable this argument and run our application, we now see logging from `AnimalsData`:

```text
[AnimalsData][Listener] Old State: AnimalsState(categories: AnimalsData.AnimalsState.Categories(data: [:], status: nil), animals: AnimalsData.AnimalsState.Animals(data: [:], status: nil, queue: [:]))
[AnimalsData][Listener] Action: ui(AnimalsData.AnimalsAction.UI.categoryList(AnimalsData.AnimalsAction.UI.CategoryList.onAppear))
[AnimalsData][Listener] New State: AnimalsState(categories: AnimalsData.AnimalsState.Categories(data: [:], status: Optional(AnimalsData.Status.waiting)), animals: AnimalsData.AnimalsState.Animals(data: [:], status: nil, queue: [:]))
[AnimalsData][Listener] Old State: AnimalsState(categories: AnimalsData.AnimalsState.Categories(data: [:], status: Optional(AnimalsData.Status.waiting)), animals: AnimalsData.AnimalsState.Animals(data: [:], status: nil, queue: [:]))
[AnimalsData][Listener] Action: data(AnimalsData.AnimalsAction.Data.persistentSession(AnimalsData.AnimalsAction.Data.PersistentSession.didFetchCategories(result: AnimalsData.AnimalsAction.Data.PersistentSession.FetchCategoriesResult.success(categories: [AnimalsData.Category(categoryId: "Invertebrate", name: "Invertebrate"), AnimalsData.Category(categoryId: "Mammal", name: "Mammal"), AnimalsData.Category(categoryId: "Reptile", name: "Reptile"), AnimalsData.Category(categoryId: "Fish", name: "Fish"), AnimalsData.Category(categoryId: "Bird", name: "Bird"), AnimalsData.Category(categoryId: "Amphibian", name: "Amphibian")]))))
[AnimalsData][Listener] New State: AnimalsState(categories: AnimalsData.AnimalsState.Categories(data: ["Bird": AnimalsData.Category(categoryId: "Bird", name: "Bird"), "Fish": AnimalsData.Category(categoryId: "Fish", name: "Fish"), "Reptile": AnimalsData.Category(categoryId: "Reptile", name: "Reptile"), "Invertebrate": AnimalsData.Category(categoryId: "Invertebrate", name: "Invertebrate"), "Mammal": AnimalsData.Category(categoryId: "Mammal", name: "Mammal"), "Amphibian": AnimalsData.Category(categoryId: "Amphibian", name: "Amphibian")], status: Optional(AnimalsData.Status.success)), animals: AnimalsData.AnimalsState.Animals(data: [:], status: nil, queue: [:]))
```

For every action dispatched to our `Store`, our `Listener` instance is now logging the global state, the action value, and the state returned from our Reducer. This can log *a lot* of data when application state grows large and complex, but it can be a very useful tool for investigating unexpected behaviors.

Let’s enable the logging we built in `ImmutableUI`. Here is our new launch argument:

```text
-com.northbronson.ImmutableUI.Debug 1
```

When we enable this argument and run our application, we now see logging from `Listener`:

```text
[ImmutableUI][AsyncListener]: 0x0000600000514870 Update: SelectCategoriesValues
[ImmutableUI][AsyncListener]: 0x0000600000514870 Update Dependency: SelectCategoriesValues
[ImmutableUI][AsyncListener]: 0x0000600000514870 Update Output: SelectCategoriesValues
[ImmutableUI][AsyncListener]: 0x0000600001c34ee0 Update: SelectCategoriesStatus
[ImmutableUI][AsyncListener]: 0x0000600001c34ee0 Update Output: SelectCategoriesStatus
[ImmutableUI][AsyncListener]: 0x00006000005150e0 Update: SelectAnimalsValues(categoryId: nil)
[ImmutableUI][AsyncListener]: 0x00006000005150e0 Update Dependency: SelectAnimalsValues(categoryId: nil)
[ImmutableUI][AsyncListener]: 0x00006000005150e0 Update Output: SelectAnimalsValues(categoryId: nil)
[ImmutableUI][AsyncListener]: 0x000060000190e980 Update: SelectCategory(categoryId: nil)
[ImmutableUI][AsyncListener]: 0x000060000190e980 Update Output: SelectCategory(categoryId: nil)
[ImmutableUI][AsyncListener]: 0x0000600001c35ab0 Update: SelectAnimalsStatus
[ImmutableUI][AsyncListener]: 0x0000600001c35ab0 Update Output: SelectAnimalsStatus
[ImmutableUI][AsyncListener]: 0x00006000006300a0 Update: SelectAnimal(animalId: nil)
[ImmutableUI][AsyncListener]: 0x00006000006300a0 Update Output: SelectAnimal(animalId: nil)
[ImmutableUI][AsyncListener]: 0x000060000190ba80 Update: SelectCategory(animalId: nil)
[ImmutableUI][AsyncListener]: 0x000060000190ba80 Update Output: SelectCategory(animalId: nil)
[ImmutableUI][AsyncListener]: 0x0000600001c20070 Update: SelectAnimalStatus(animalId: nil)
[ImmutableUI][AsyncListener]: 0x0000600001c20070 Update Output: SelectAnimalStatus(animalId: nil)
[ImmutableUI][AsyncListener]: 0x0000600001c34ee0 Update: SelectCategoriesStatus
[ImmutableUI][AsyncListener]: 0x0000600001c34ee0 Update Output: SelectCategoriesStatus
[ImmutableUI][AsyncListener]: 0x0000600001c35ab0 Update: SelectAnimalsStatus
[ImmutableUI][AsyncListener]: 0x0000600001c35ab0 Update Output: SelectAnimalsStatus
[ImmutableUI][AsyncListener]: 0x00006000006300a0 Update: SelectAnimal(animalId: nil)
[ImmutableUI][AsyncListener]: 0x00006000006300a0 Update Output: SelectAnimal(animalId: nil)
[ImmutableUI][AsyncListener]: 0x000060000190ba80 Update: SelectCategory(animalId: nil)
[ImmutableUI][AsyncListener]: 0x000060000190ba80 Update Output: SelectCategory(animalId: nil)
[ImmutableUI][AsyncListener]: 0x0000600001c20070 Update: SelectAnimalStatus(animalId: nil)
[ImmutableUI][AsyncListener]: 0x0000600001c20070 Update Output: SelectAnimalStatus(animalId: nil)
[ImmutableUI][AsyncListener]: 0x000060000190e980 Update: SelectCategory(categoryId: nil)
[ImmutableUI][AsyncListener]: 0x000060000190e980 Update Output: SelectCategory(categoryId: nil)
[ImmutableUI][AsyncListener]: 0x0000600001c35ab0 Update: SelectAnimalsStatus
[ImmutableUI][AsyncListener]: 0x0000600001c35ab0 Update Output: SelectAnimalsStatus
[ImmutableUI][AsyncListener]: 0x0000600000514870 Update: SelectCategoriesValues
[ImmutableUI][AsyncListener]: 0x0000600000514870 Update Dependency: SelectCategoriesValues
[ImmutableUI][AsyncListener]: 0x0000600000514870 Update Output: SelectCategoriesValues
[ImmutableUI][AsyncListener]: 0x000060000190ba80 Update: SelectCategory(animalId: nil)
[ImmutableUI][AsyncListener]: 0x000060000190ba80 Update Output: SelectCategory(animalId: nil)
[ImmutableUI][AsyncListener]: 0x000060000190e980 Update: SelectCategory(categoryId: nil)
[ImmutableUI][AsyncListener]: 0x000060000190e980 Update Output: SelectCategory(categoryId: nil)
[ImmutableUI][AsyncListener]: 0x00006000006300a0 Update: SelectAnimal(animalId: nil)
[ImmutableUI][AsyncListener]: 0x00006000006300a0 Update Output: SelectAnimal(animalId: nil)
[ImmutableUI][AsyncListener]: 0x0000600001c20070 Update: SelectAnimalStatus(animalId: nil)
[ImmutableUI][AsyncListener]: 0x0000600001c20070 Update Output: SelectAnimalStatus(animalId: nil)
[ImmutableUI][AsyncListener]: 0x0000600001c34ee0 Update: SelectCategoriesStatus
[ImmutableUI][AsyncListener]: 0x0000600001c34ee0 Update Output: SelectCategoriesStatus
```

These look like a lot of selectors to launch our application, but most of these run in constant time. The only selectors here that run above-constant time are `SelectCategoriesValues` and `SelectAnimalsValues`, and `SelectAnimalsValues` returns an empty `Array` in constant time when we pass `nil` for `Category.ID`. Our first `SelectCategoriesValues` returns an empty `Array`; there is no time spent sorting because there are no `Category` values before our asynchronous fetch operations has completed. The good news here is displaying our components performs just one `O(n log n)` operation.

Our focus is on teaching the `ImmutableData` architecture. We do teach how to leverage SwiftData, but a complete tutorial on debugging and optimizing SwiftData is outside the scope of this tutorial. We would like to recommend two more launch arguments you might be interested in:

```text
-com.apple.CoreData.SQLDebug 1
-com.apple.CoreData.ConcurrencyDebug 1
```

As of this writing, these launch arguments — which were originally built for CoreData — enable extra logging and stricter concurrency checking.[^2] Try these for yourself as you build products on SwiftData to help defend against bugs before they ship in your production application.

We completed two sample products: Counter and Animals. Both applications are built from `ImmutableData`. We specify our product domain when we construct our `Store`, but the `ImmutableData` infra *itself* did not change. Our infra deployed to both products without additional work. We’re going to use that same infra — without making changes to that infra — to begin a new sample product in our next chapter.

[^1]: https://www.avanderlee.com/xcode/overriding-userdefaults-launch-arguments/
[^2]: https://useyourloaf.com/blog/debugging-core-data/
