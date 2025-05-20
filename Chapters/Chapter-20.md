# Next Steps

As they say at FB: *This journey is one-percent finished.*

We covered a lot of ground in this tutorial. We built a new infra library from scratch, we built multiple sample application products, we learned about some external dependencies we can import for specialized data structures to improve performance, and we ran benchmarks to measure how these immutable data structures compare to SwiftData.

If you’re ready to experiment with the `ImmutableData` architecture in your own products, you have a few different options available:

* If you have the ability to import a new external dependency in your product, you can import the [`ImmutableData`](https://github.com/Swift-ImmutableData/ImmutableData) repo package. This is a “standalone” package version of the infra we built in this Programming Guide.
* If you have a product that deploys to older operating systems, you might be blocked on importing `ImmutableData` because of the dependencies on `Observable` and variadic types. The [`ImmutableData-Legacy`](https://github.com/Swift-ImmutableData/ImmutableData-Legacy) repo package is a version of the `ImmutableData` infra that deploys to older operating systems.
* If you are blocked on importing *any* new external dependency, you can follow the steps in this Programming Guide to build your own version of the `ImmutableData` architecture from scratch.

If you are building a new product from scratch, the sample application products built in this Programming Guide can help you begin to think creatively how `ImmutableData` can be used for your own product domain.

If you are attempting to bring `ImmutableData` to an *existing* product built on a legacy architecture, we recommend reading our [`ImmutableData-FoodTruck`](https://github.com/Swift-ImmutableData/ImmutableData-FoodTruck) tutorial. The `ImmutableData-FoodTruck` tutorial is an *incremental* migration: we show you how the `ImmutableData` architecture can “coexist” along with a legacy architecture built on imperative logic and mutable data models.

As always, please file a new GitHub issue if you encounter any compatibility problems, issues, or bugs with `ImmutableData`. Please let us know if there are any missing pieces: anything that is blocking you or your team from migrating to `ImmutableData`.

Thanks!
