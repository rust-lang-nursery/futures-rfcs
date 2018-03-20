# Summary
[summary]: #summary

Prepare for a push toward futures 1.0 this year by releasing a futures 0.2
that's structured for faster iteration.

The main goals of this release are:

- Separation of `futures` into multiple, smaller crates, which are reexported
  in a `futures` facade crate. This will make it possible to gradually evolve
  `futures` libraries without major ecosystem breakage.
- A few substantial changes, potentially including [making task handling
  explicit].
- Emptying the queue of minor breaking changes, including removing deprecated
  features and minor API tweaks (renamings, generalizations, etc.).

[making task handling explicit]: (https://github.com/rust-lang-nursery/futures-rfcs/pull/2)

This breakage is intended to overlap with library changes resulting from the
release of the `tokio` crate (formerly `tokio-core`) and `mio` 0.7.  The `tokio`
changes will result in changes to many popular libraries in the Rust async
ecosystem. By releasing `futures` 0.2 at or around the same time, we hope to
minimize the number of breaking changes to the ecosystem.

# Motivation
[motivation]: #motivation

The futures team would like to iterate toward and publish a 1.0 in 2018. We
aren't going to be able to do that in one big step, in part because we are
anticipating further changes to work well with async/await (see [@withoutboats's
blog series] for more on that). part of the motivation for this RFC is to kick
off that iteration, and also make space for further iterations by breaking up
the crates -- in particular, splitting up the combinators and the different core
traits to allow for more independent revisions.

[@withoutboats's blog series]: https://boats.gitlab.io/blog/post/2018-01-30-async-iii-moving-forward/

## Crate Separation
[motivation-separation]: #motivation-separation

When first publishing new functionality in a library, it is common for APIs to
change in response to user feedback. API inconsistencies, missing features,
unexpected edge-case behavior, and potential user-experience improvements are
often discovered through real-world testing and experimentation.

Unfortunately, popular crates like `futures` appear as public dependencies of
other libraries. Making a breaking change to anything in the `futures` crate
breaks libraries which exposed their own `Future` types or took `Future` types
as arguments.

Separating the `futures` crate into multiple independently-versioned
libraries will allow breaking changes to combinators or other less stable
parts of the library. These changes can be made more frequently without the
ecosystem-wide breakage that would result from breakage of the `Future` or
`Stream` traits.

For example, imagine that the `asyncprint` crate exposes a function called
`asyncprint` which takes a type `F: Future<Item=String, Error=io::Error>`,
completes it, and prints the result. Internally, the implementation of
`asyncprint` uses `and_then`, but it doesn't expose the `AndThen` type
publicly.

Imagine `AndThen` gets a breaking change update makes it slightly easier to use
or more flexible. If `Future` and `AndThen` are both defined in the same crate,
`futures`, then `futures` now has a major version increase. The new crate's
`Future` trait is incompatible with the old crate's `Future` trait, even though
no changes were made to `Future`. In this case, `asyncprint` cannot update to
the new version of `futures` without breaking users, since the new version of
`Future` is incompatible with types that implemented the old `Future`.

However, if `Future` is defined in a separate crate from `AndThen`,
`asyncprint` can update to the new version and use the new `AndThen` without
breaking users.

This is the goal for breaking apart `futures` into separate crates: after this
change, it will be possible to make breaking changes to parts of `futures`
without causing major ecosystem breakage.

**Note: this RFC proposes a more aggressive "sharding" of crates than we
expect in 1.0, to make space for iteration**. The details are given below.

## Small, Breaking API Improvements
[motivation-small-breakage]: #motivation-small-breakage

0.2 breakage would make it possible to resolve a [number of minor issues] that
require breaking changes to fix, as well as to clear out a number of old
deprecations and renamings.

[number of minor issues]: https://github.com/alexcrichton/futures-rs/issues?utf8=%E2%9C%93&q=is%3Aissue+0.2

## Timing in Coordination with Tokio 0.1 and Mio 0.7
[motivation-timing]: #motivation-timing

There is an impending release of the `tokio` crate which will use `mio` 0.7.
Transitioning to use these new APIs will already require some amount of library
breakage or deprecation and replacement. Specifically, functions which took
an explicit handle to a `tokio` reactor core will no longer need to do so.
Functions which formerly accepted the old `tokio-core` handles will not be
compatible with the new `mio` 0.7-based `tokio` crate.

Releasing `futures` 0.2 at the same as `tokio` would consolidate the API
transition period.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `futures` crate is working toward 1.0 this year! To get there, we're going
to need to go through a few iterations, including exploration of the space
around async/await.

To make these transitions as painless as we can, we're doing a couple things today:

- We're releasing futures 0.2 *in tandem* with the new Tokio crate, which is
  already going to result in an ecosystem shift.

- In this release, we split the `futures` crate into a number of smaller
  libraries to allow for easier API evolution and validation as we work toward
  1.0. The actual 1.0 release will contain a smaller set of stable crates.

Library authors should update to use the appropriate crates:

- `futures-core` for the `Future`, `Stream`, `IntoFuture`, `IntoStream` traits
  and their dependencies.
- `futures-io` which includes updated `AsyncRead` and `AsyncWrite` traits.
- `futures-sink` for the `Sink` trait which is used for sending values
  asynchronously.
- `futures-executor` for the `Executor` trait which allows spawning futures
  to run asynchronously.
- `futures-channel` for channels and other synchronization primitives.
- `futures-util` which offers a number of convenient combinator functions,
  some of which are included in `FutureExt` and `StreamExt` extension traits.

By minimizing the number of `futures` crates in the public API of your library,
you will be able to receive more frequent updates without causing breaking
changes for your users.

Application developers can either depend on individual crates or on the
`futures` crate, which provides convenience reexports of many of the
`futures-xxx` crates. Expect that this crate will publish more frequent
semver-breaking changes, but that updating will not normally result in
significant or widespread breakage.

These crates offer easier-to-use and easier-to-evolve APIs than the futures 0.1
series.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Below you will find a *partial* list of some of the breaking changes we hope to
include in the new `futures` 0.2 release. Note that this list is not exact,
and does not aim to be so: this list is merely intended to motivate and
explain some of the desired changes to the `futures` create. Please leave
comments about specific issues on [the `futures` repo](https://github.com/alexcrichton/futures-rs),
as detailed discussion could easily overwhelm the RFC thread.

## Crate Separation

The `futures` crate will be *temporarily* split into several different
libraries; before reaching 1.0, we will reduce the crate count.

### `futures-core`

`futures-core` will include `Future`, `Stream`, `IntoFuture`
traits and their dependencies. This is a central crate that includes only
essential items, designed to minimize future breakage. Combinator methods
will be removed from the `Future` and `Stream` and moved to the
`FutureExt` and `StreamExt` traits in the `futures-util` crate. `poll`
will be the only remaining method on the `Future` and `Stream` traits.
This separation allows combinator functions to evolve over time without
breaking changes to the public API surface of async libraries.

### `futures-io`

`futures-io` includes the updated `AsyncRead` and `AsyncWrite` traits.
These traits were previously included in the `tokio-io` crate.
However, `AsyncRead` and `AsyncWrite` don't have any dependencies on the `tokio`
crate or the rest of the `tokio` stack, so they will be moved into the `futures`
group of libraries. `futures-io` depends only on the `futures-core` crate
and the `bytes` crate (for the [`Buf`] and [`BufMut`] traits).

[`Buf`]: https://carllerche.github.io/bytes/bytes/trait.Buf.html
[`BufMut]: https://carllerche.github.io/bytes/bytes/trait.BufMut.html

### `futures-sink`

`futures-sink` includes the `Sink` trait, which allows values to be sent
asynchronously. The `Sink` trait is excluded from `futures-core` because its
design is much less stable than `Future` and `Stream`. There are several
potential breaking changes to the `Sink` trait under consideration.

This crate will depend only on `futures-core`.

### `futures-channel`

`futures-channel` includes the `mpsc` and `oneshot` messaging primitives.
This crate is a minimal addition on `futures-core` designed for low churn
so that channels may appear in the public APIs of libraries.

`futures-channel` will not include a dependency on the `futures-sink` crate
because it is more volatile. In order to have channels implement the `Sink`
trait, channels in `futures-channel` will expose `poll_send` methods which
will be used to implement `Sink` inside of the `futures-sink` crate.

This is the only crate that is not compatible with `no_std`, as it relies
on synchronization primitives from `std`.

### `futures-util`

`futures-util` contains a number of convenient combinator functions for
`Future`, `Stream`, and `Sink`, including the `FutureExt`, `StreamExt`,
and `SinkExt` extension traits.

This crate depends on `futures-core`, `futures-io`, and `future-sink`.

### `futures-executor`

`futures-executor` contains the `Executor` trait which allows spawning
asynchronously tasks onto a task executor. Like `Sink`, this is a less
mature API which hasn't seen as much practical use. There are several
possible breaking changes to this API coming in the near future.

This crate depends on `futures-core`.

### The plan for 1.0

After we've had a chance to iterate on these separate crates and to fully
integrate with [async/await notation], we will consolidate down to *three* final crates:

- `futures`, for `Future`, `Stream`, `Sink`, `AsyncRead`,
  `AsyncWrite`, and all related combinators.

- `futures-channel`, for channels and other synchronization primitives.

- `futures-executor` for the `Executor` trait which allows spawning futures
  to run asynchronously, and some built-in executors.

[async/await notation]: https://boats.gitlab.io/blog/post/2018-01-30-async-iii-moving-forward/

## Small, Breaking API Improvements and Clarifications

- `Task` will be renamed to `Waker`. This name more clearly reflects how
  the type is used. It is a handle used to awaken tasks-- it is not a `Task`
  itself.
- The `wait` method will be removed in favor of a free-function-based `blocking` API.
- The `Notify` trait will be simplified to not require explicit ID passing, while
  still supporting management of allocation.
- `Async` variant `NotReady` will be renamed to `Pending`.
- `AsyncRead` and `AsyncWrite` no longer have `io::Read` and `io::Write`
  bounds. Discussion on this can be found [in this Github issue.](https://github.com/tokio-rs/tokio-io/issues/83)
- `AsyncRead::prepare_uninitialized_buffer` has been replaced with an
  `initializer` function similar to [the unstable one on `io::Read`.](https://doc.rust-lang.org/std/io/trait.Read.html#method.initializer)
  This resolves the issue of the `unsafe` on `prepare_uninitialized_buffer`
  being `unsafe` to implement, *not* `unsafe` to use
  (the more common, and arguably the correct meaning of `unsafe fn`).
- `AsyncRead` and `AsyncWrite` now offer separate methods for non-vectorized and
  vectorized IO operations, with the vectorized version based on `IoVec`. This
  makes the API easier to understand, makes the traits object-safe, and removes
  the dependency on the `bytes` crate. The `Buf`-based methods can be recovered
  via extension traits, which also enables them to work on trait objects.
- `AsyncRead/Write` `read_buf`, `write_buf`, and `shutdown` have been renamed
  to `poll_read`, `poll_write`, and `poll_close` for increased standardization.
  The name `shutdown` was replaced [to avoid confusion with TCP shutdown.](https://github.com/tokio-rs/tokio-io/issues/80)
- The flexibility of [`unfold`](https://github.com/alexcrichton/futures-rs/issues/260)
  will be increased.
- [`JoinAll`'s lifetime requirements will be loosened.](https://github.com/alexcrichton/futures-rs/issues/285)
- [`filter` and `skip_while` will be made more consistent and flexible.](https://github.com/alexcrichton/futures-rs/issues/311)
- [`filter_map` will be more flexible.](https://github.com/alexcrichton/futures-rs/issues/338)
- [The default implementation of `Sink::close` will be removed.](https://github.com/alexcrichton/futures-rs/issues/413)
- [The implementation of `Future<Item = Option<T>>` for `Option<impl Future<Item = T>>`
  will be removed and replaced with an `IntoFuture` implementation.](https://github.com/alexcrichton/futures-rs/issues/445)
- [The order of type parameters in combinators and types will be standardized.](https://github.com/alexcrichton/futures-rs/issues/517)
- [`Flush` accessors will return `Option` instead of `panic`ing.](https://github.com/alexcrichton/futures-rs/issues/706)
- [`future::FutureResult` will be renamed to `future::Result`.](https://github.com/alexcrichton/futures-rs/issues/706)

## Documentation

In parallel with the 0.2 release, the futures team will continue working on a
book documenting futures, as well as a thorough revamp of the API documentation.

# Drawbacks
[drawbacks]: #drawbacks

- Async-ecosystem-wide breakage.
- Separate crates may make the `futures` API surface harder to understand.
  This should be alleviated by reexporting all of the functionality through
  the functionality through the master `futures` crate.

# Rationale and alternatives
[alternatives]: #alternatives

- Keep a subset of the methods on `Future` and `Stream` in the core trait,
  rather than the extension trait. The problem with this idea is that it's
  difficult to draw the line as to exactly which methods are prone to
  breakage. Potentially all of them could be subject to breakage in order
  to provide more convenient error-handling functionality. Furthermore,
  keeping some of the methods in `Future` and some in `FutureExt` could
  cause confusion among users and make it hard to discover the extension
  methods.
- Require `Async::Pending` to include a `WillWake` returned from a method on
  `Waker` . This would ensure that a `Waker` has been acquired.
  `WillWake` could be zero-sized in release mode, and in debug
  mode it could indicate what task it came from. This could help tracking
  down bugs tied to lost wakeups. However, it's not clear how this would
  be used with `select`-style futures which only return `Pending` when
  all of their child futures return `Pending`. Which `WillWake` should
  be passed on in this case? Really, you need a way to indicate that all
  of the child future `WillWake` the task, not just one. This scenario
  is one of the situations in which wakeups are most commonly lost, so
  a solution here would provide a significant advantage to this proposal.
  Suggestions and ideas are welcome!

# Unresolved questions
[unresolved]: #unresolved-questions

- Going forwards, it'd be interesting to explore ways for `cargo` or
  `crates.io` to natively support multi-crate packages or bundles.
  Facade crates such as the `futures` crate proposed here are a good
  intermediate solution.
