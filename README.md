# The ImmutableData Programming Guide

“What is the best design pattern for SwiftUI apps?”

We hear this question a lot. Compared to the days when AppKit and UIKit were the dominant frameworks for product engineering in the Apple Ecosystem, Apple has been relatively un-opinionated about what kind of design pattern engineers should choose “by default” for their SwiftUI applications.

Many engineers in the SwiftUI community are currently evangelizing a “MVVM” design pattern. Other engineers are making the argument that SwiftUI is really encouraging a “MVC” design pattern. You might have also heard discussion of a “MV” design pattern. These design patterns share a fundamental philosophy: the state of your application is managed from your view components using imperative logic on mutable model objects. To put it another way, these design patterns start with a fundamental assumption of *mutability* that drives the programming model that product engineers must opt-in to when building graphs of view components. The “modern and declarative” programming model product engineers have transitioned to for SwiftUI is then paired with a “legacy and imperative” programming model for managing shared mutable state.

Over the course of this project, we present what we think is a better way. Drawing on over a decade of experience shipping products at scale using declarative UI frameworks, we present a new application architecture for SwiftUI. Using the Flux and Redux architectures as a philosophical prior art, we can design an architecture using modern Swift and specialized for modern SwiftUI. This architecture encourages *declarative* thinking instead of *imperative* thinking, *functional* programming instead of object-oriented programming, and *immutable* model values instead of mutable model objects.

At the core of the architecture is a unidirectional data flow:

```mermaid
flowchart LR
  accTitle: Data Flow in the ImmutableData Framework
  accDescr: Data flows from action to state, and from state to view, in one direction only.
  A[Action] --> B[State] --> C[View]
```

All global state data flows through the application following this basic pattern, and a strict separation of concerns is enforced. The actions *declare* what has occurred, whether user input, a server response, or a change in a device’s sensors, but they have no knowledge of the state or view layers. The state layer *reacts* to the “news” described by the action and updates the state accordingly. All logic for making changes to the state is contained within the state layer, but it knows nothing of the view layer. The views then *react* to the changes in the state layer as the new state flows through the component tree. Again, however, the view layer knows nothing about the state layer. By maintaining this strict unidirectional data flow and separation of concerns, our application code becomes easier to test, easier to reason about, easier to explain to new team members, and easier to update when new features are required.

Further, avoiding complexity like two-way data bindings, or the spaghetti engendered by mutability, allows our code to become clean, fast, and maintainable. This is the key difference between this application framework (and the ideas behind it) and other presentations of actions, state, and view previously shown at WWDC.[^1] By avoiding direct mutations called from outside the state layer and embracing immutability instead, complexity vanishes and our code becomes much more robust.

We call this framework and architecture `ImmutableData`. We present `ImmutableData` as a free and open-source project with free and open-source documentation. Over the course of this tutorial, we will show you, step-by-step, how the `ImmutableData` infra is built. Once the infra is ready, we will then build, step-by-step, multiple sample applications using SwiftUI to display and transform state through the `ImmutableData` architecture.

## Requirements

Our goal is to teach a new way of thinking about state management and data flow for SwiftUI. Our goal *is not* to teach Swift Programming or the basics of SwiftUI. You should have a strong competency in Swift 6.0 before beginning this tutorial. You should also have a working familiarity with SwiftUI. A working familiarity with SwiftData would be helpful, but is not required.

Inspired by Matt Gallagher, our project will make heavy use of modules and access control to keep our code organized.[^2] A working familiarity with Swift Package Manager will be helpful, but our use of Swift Package APIs will be kept at a relatively basic level.

The `ImmutableData` infra deploys to the following platforms:
* iOS 17.0+
* iPadOS 17.0+
* macOS 14.0+
* tvOS 17.0+
* visionOS 1.0+
* watchOS 10.0+

Building the `ImmutableData` tutorial requires Xcode 16.0+ and macOS 14.5+.

Please file a GitHub issue if you encounter any compatibility problems.

## Organization

*The ImmutableData Programming Guide* is inspired by “long-form” documentation like [*Programming with Objective-C*][^3] and [*The Swift Programming Language*][^4].

This guide includes the following chapters:

### Part 0: Overview
* [Chapter 00](Chapters/Chapter-00.md): We discuss the history and evolution of Flux, Redux, and SwiftUI. In what ways did SwiftUI evolve in a similar direction as React? How can our `ImmutableData` architecture use ideas from React to improve product engineering for SwiftUI?
### Part 1: Infra
* [Chapter 01](Chapters/Chapter-01.md): We build the `ImmutableData` module for managing the global state of our application.
* [Chapter 02](Chapters/Chapter-02.md): We build the `ImmutableUI` module for making our global state available to SwiftUI view components.
### Part 2: Products
* [Chapter 03](Chapters/Chapter-03.md): We build the data models of our Counter application: a simple SwiftUI app to increment and decrement an integer.
* [Chapter 04](Chapters/Chapter-04.md): We build the component tree of our Counter application.
* [Chapter 05](Chapters/Chapter-05.md): We build and run our Counter application.
* [Chapter 06](Chapters/Chapter-06.md): We build the data models of our Animals application: a SwiftUI app to store a collection of data models with persistence to a local database.
* [Chapter 07](Chapters/Chapter-07.md): We build a command-line utility for testing the data models of our Animals application without any component tree.
* [Chapter 08](Chapters/Chapter-08.md): We build the component tree of our Animals application.
* [Chapter 09](Chapters/Chapter-09.md): We build and run our Animals application.
* [Chapter 10](Chapters/Chapter-10.md): We build the data models of our Quakes application: a SwiftUI app to fetch a collection of data models from a remote server with persistence to a local database.
* [Chapter 11](Chapters/Chapter-11.md): We build a command-line utility for testing the data models of our Quakes application without any component tree.
* [Chapter 12](Chapters/Chapter-12.md): We build the component tree of our Quakes application.
* [Chapter 13](Chapters/Chapter-13.md): We build and run our Quakes application.
* [Chapter 14](Chapters/Chapter-14.md): We update the data models of our Animals application to support persistence to a remote server.
* [Chapter 15](Chapters/Chapter-15.md): We build an HTTP server for testing our new Animals application.
* [Chapter 16](Chapters/Chapter-16.md): We build a command-line utility for testing the data models of our new Animals application without any component tree.
* [Chapter 17](Chapters/Chapter-17.md): We build and run our new Animals application.
### Part 3: Performance
* [Chapter 18](Chapters/Chapter-18.md): We learn about specialized data structures that can improve the performance of our applications when working with large amounts of data that is copied many times.
* [Chapter 19](Chapters/Chapter-19.md): We run benchmarks to measure how the performance of immutable collection values compare to SwiftData.
### Part 4: Next Steps
* [Chapter 20](Chapters/Chapter-20.md): Here are some final thoughts about what’s coming next.

## Companion Repos

You can find more repos on our `ImmutableData` GitHub organization:

* [`ImmutableData-Samples`](https://github.com/Swift-ImmutableData/ImmutableData-Samples) includes empty Swift packages, empty Xcode projects, and an empty Xcode workspace. This is the recommended way to complete our tutorial. The workspace provides some basic setup (like adding dependencies between packages) that will let you focus on our tutorial.
* [`ImmutableData-Benchmarks`](https://github.com/Swift-ImmutableData/ImmutableData-Benchmarks) includes benchmarks to measure performance. These benchmarks will be discussed in Chapter 19.
* [`ImmutableData`](https://github.com/Swift-ImmutableData/ImmutableData) is the “standalone” repo package of the `ImmutableData` infra. This is for product engineers that want to use the infra in their products without building the infra from scratch.
* [`ImmutableData-Legacy`](https://github.com/Swift-ImmutableData/ImmutableData-Legacy) is a version of the `ImmutableData` infra that deploys to older operating systems.
* [`ImmutableData-FoodTruck`](https://github.com/Swift-ImmutableData/ImmutableData-FoodTruck) is an incremental migration tutorial. We begin with an app from Apple built on a legacy architecture. We then show how the `ImmutableData` infra can be used to incrementally migrate individual components and product surfaces *without* changing the behavior of components and product surfaces built on the legacy infra. We show how `ImmutableData` can “coexist” with a legacy architecture.

## License

Copyright 2024 Rick van Voorden and Bill Fisher

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

[^1]: https://developer.apple.com/videos/play/wwdc2019/226
[^2]: https://www.cocoawithlove.com/blog/app-submodules.html
[^3]: https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/
[^4]: https://www.swift.org/documentation/tspl/
