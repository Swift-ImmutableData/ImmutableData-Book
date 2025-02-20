# Counter.app

The final step to complete our Counter application is to build a `SwiftUI.App` entry point.

Select `Counter.xcodeproj` and open `CounterApp.swift`. We can delete the original “Hello World” template. Let’s begin by defining our `CounterApp` type:

```swift
//  CounterApp.swift

import CounterData
import CounterUI
import ImmutableData
import ImmutableUI
import SwiftUI

@main @MainActor struct CounterApp {
  @State private var store = Store(
    initialState: CounterState(),
    reducer: CounterReducer.reduce
  )
}
```

In our previous chapter, we saw that we are required to provide a `defaultValue` for `StoreKey`. While you *could* use this `defaultValue` as the global state for your application, we strongly recommend creating a `Store` in your `App` entry point. In our next sample application products, we also see how this is where we configure our Listeners to manage complex asynchronous operations on behalf of our `Store`.

Now that we have a `Store` instance, let’s add a `body` to complete our application:

```swift
//  CounterApp.swift

extension CounterApp : App {
  var body: some Scene {
    WindowGroup {
      Provider(self.store) {
        Content()
      }
    }
  }
}
```

We deliver our `Store` instance through our component tree using the `Provider` type. We don’t have to provide an explicit key path because this is using the extension initializer we defined in our previous chapter.

That should be all we need for now. Go ahead and build and run (`⌘ R`) and watch our Counter application run on your computer. This window should function like what we saw in the Previews we built in our previous chapter: Our Increment Button and Decrement Button update our Value component. What we didn’t see from `Preview` was multiple windows. Go ahead and create a new window (`⌘ N`) to see them both working together. Change the value in the first window and watch it update in the second window.

As previously discussed, we focus our attention in this tutorial on building macOS applications. This saves us some time and keeps us moving forward, but the `ImmutableData` infra is meant to be multiplatform: any platform where SwiftUI is available. Nothing about `CounterApp` really *needs* macOS; we could deploy this app to iOS, iPadOS, tvOS, visionOS, or watchOS. Optimizing SwiftUI applications to display on multiple platforms is an important topic, but is mostly a discussion about *presentation* of data; this is orthogonal to our main goal: teaching the `ImmutableData` architecture. We continue to focus on macOS for this tutorial, but we encourage you to deploy `ImmutableData` to all platforms supported by your product.

This sample application is very simple; all of our state is *global state*: just an integer value. In our next sample application products, we will see examples of much more complex global states along with examples of *local state* which we save in our view component tree directly.

Finally, we want to discuss some very important topics for front-end engineering: internationalization, localization, and accessibility. Internationalization and localization is the process of presenting data in formats and languages most appropriate for users.[^1] Accessibility is the process of presenting data in a way that everyone can understand your product, including people with disabilities that might use assistive technologies.[^2] Building applications for *all* users is an important goal, but is mostly a discussion about *presentation* of data. This can be learned orthogonally to the `ImmutableData` architecture. We encourage you to explore the resources available from Apple for learning more about these best practices.

[^1]: https://developer.apple.com/localization/
[^2]: https://developer.apple.com/accessibility/
