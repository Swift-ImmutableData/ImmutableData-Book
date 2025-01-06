# Next Steps

As they say at FB: *This journey is one-percent finished.*

We covered a lot of ground in this tutorial. We built a new infra library from scratch, we built multiple sample application products, we learned about some external dependencies we can import for specialized data structures to improve performance, and we ran benchmarks to measure how these immutable data structures compare to SwiftData.

We have a lot more work to do. This tutorial is enough to bring to the community for an introduction. Here are the most important tasks for us to finish next:

* Ship an `ImmutableData` and `ImmutableUI` repo. These are the “productized” versions of the infra modules we built in our tutorial; meant for you to add as a dependency in your own products.
* Ship an incremental migration. We built three greenfield sample products. How can we help engineers that have an existing product built on SwiftData? What does a migration look like? How can a product incrementally migrate to `ImmutableData` when it’s not easy or practical to rewrite the whole thing all at once?

As always, please file a new GitHub issue if you encounter any compatibility problems, issues, or bugs with `ImmutableData`. Please let us know if there are any missing pieces: anything that is blocking you or your team from migrating to `ImmutableData`.

Thanks!
