- Feature Name: futures-2
- Start Date: 2018-01-22
- RFC PR:
- Rust Issue:

# Summary
[summary]: #summary

Release a breaking change to the `futures` crate.
The main goals of this breakage are:
- Learnability and readability improvements acheived by making task wakeup
  handles into explicit arguments.
- Separation of `futures` into multiple, smaller crates, which are reexported
  in a `futures` facade crate. This will make it possible to gradually evolve
  `futures` libraries without major ecosystem breakage.
- Removal of deprecated functionality.
- Minor API cleanup (renamings, generalizations, etc.).

This breakage may (depending on time to accept and implement) overlap with
library changes resulting from the release of the `tokio` crate
(formerly `tokio-core`) and `mio` 0.7.
The `tokio` changes will result in changes to many popular libraries
in the Rust async ecosystem. By releasing `futures` 0.2 at or around the same
time, we hope to minimize the number of breaking changes to the ecosystem.

# Motivation
[motivation]: #motivation

## Explicit Task Wakeup Handles

The `futures` task wakeup mechanism is [a regular source of confusion for new
users][explicit task issue]. Looking at the signature of `Future::poll`, it's
hard to tell how `Future`s work:

```rust
fn poll(&mut self) -> Poll<Self::Item, Self::Error>;
```

`poll` takes no arguments, and returns either `Ready`, `NotReady`, or an error.
From the signature, it looks as though the only way to complete a `Future` is
to continually call `poll` in a loop until it returns `Ready`. This is
incorrect.

Furthermore, when implementing `Future`, it's necessary to schedule a wakeup
when you return `NotReady`, or else you might never be `poll`ed again. The
trick to the API is that there is an implicit `Task` argument to `Future::poll`
which is passed using thread-local storage. This `Task` argument must be
`notify`d by the `Future` in order for the task to awaken and `poll` again.

Once users become aware of the TLS `Task`, it's rare that this implicit argument
is the source of bugs. Storing the `Task` in TLS is also slightly more
ergonomic when writing manual `Future` implementations since you don't have to
manually plumb through a wakeup handle.

However, as described earlier, this ergonomic benefit comes at a significant
learnability cost. Additionally, the implicit argument makes it more difficult
to tell which functions aside from `poll` will use the `Task` handle to
schedule a wakeup. Furthermore, the wakeup handle never shows up in async/await
or combinator code.

This RFC proposes to resolve these issues by making the `Task` wakeup handle
into an explicit argument. Since this avoids TLS, it also makes it easier to
support `no_std` environments.

[explicit task issue]: https://github.com/alexcrichton/futures-rs/issues/129

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

## Small, Breaking API Improvements
[motivation-small-breakage]: #motivation-small-breakage

0.2 breakage would make it possible to resolve a [number of minor issues]
that require breaking changes to fix.

[number of minor issues]: https://github.com/alexcrichton/futures-rs/issues?utf8=%E2%9C%93&q=is%3Aissue+is%3Aopen+0.2

## Removing Deprecated Types, Functions, and Aliases
[motivation-rm-deprecated]: #motivation-rm-deprecated

Over time, the `futures` crate has had a number of deprecations and renamings.
Releasing 0.2 would provide an opportunity to clean up old, deprecated code.

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

The `futures` crate has a new release! This release splits the `futures`
crate into a number of smaller libraries to allow for easier API evolution
and validation.

Library authors should update to use the appropriate crates:
- `futures-core` for the `Future`, `Stream`, `IntoFuture`, `IntoStream` traits
  and their dependencies.
- `futures-io` which includes the updated `AsyncRead` and `AsyncWrite` traits.
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
`futures` crate, which provides convenience reexports of all the `futures-xxx`
crates. Expect that this crate will publish more frequent semver-breaking
changes, but that updating will not normally result in significant or
widespread breakage.

This change comes alongside the release of the new `tokio` crate.
These crates offer greatly easier-to-use and easier-to-evolve APIs.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Below you will find a partial list of some of the breaking changes we hope to
include in the new `futures` 0.2 release. Note that this list is not exact,
and does not aim to be so: this list is merely intended to motivate and
explain some of the desired changes to the `futures` create. Please leave
comments about specific issues on [the `futures` repo](https://github.com/alexcrichton/futures-rs),
as detailed discussion could easily overwhelm the RFC thread.

## Explicit Task Context and Wakeup Handles

The `Future`, `Stream`, `AsyncRead`, and `AsyncWrite` traits have been modified
to take explicit `task::Context` arguments. `task::Context` is a type which
provides access to
[task-locals](https://docs.rs/futures/*/futures/macro.task_local.html) and a `Waker`:

```rust
mod task {

  /// The context for the currently running task.
  pub struct Context {

      /// Returns a `Waker` which can be used to wake up the current task.
      pub fn waker(&self) -> Waker { ... }

      /// Runs the provided closure with the task-local pointed to by the
      /// provided `key`.
      ///
      /// `LocalKey`s are acquired using the `task_local` macro.
      pub fn with_local<T, R, F>(&mut self, key: LocalKey<T>, f: F) -> R
          where F: FnOnce(&mut T) -> R
      {
          ...
      }
  }

  /// A handle used to awaken a task.
  pub struct Waker {

      /// Wakes up the associated task.
      ///
      /// This signals that the futures associated with the task are ready to
      /// be `poll`ed again.
      pub fn wake(&self) { ... }
  }

  ...
}
```

The modified traits will look as follows:

```rust
pub trait Future {
    type Item;
    type Error;
    fn poll(&mut self, ctx: &mut Context) -> Poll<Self::Item, Self::Error>;
}

pub trait Stream {
    type Item;
    type Error;
    fn poll_next(&mut self, ctx: &mut Context) -> Poll<Option<Self::Item>, Self::Error>;
}

pub trait AsyncRead {
    type Error;
    unsafe fn initializer(&self) -> Initializer { ... }

    fn poll_read(&mut self, buf: &mut [u8], ctx: &mut Context)
        -> Poll<usize, Self::Error>;

    fn poll_vectored_read(&mut self, vec: &mut [&mut IoVec], ctx: &mut Context)
        -> Poll<usize, Self::Error> { ... }
}

pub trait AsyncWrite {
    type Error;
    fn poll_write(&mut self, buf: &[u8], ctx: &mut Context)
        -> Poll<usize, Self::Error>;

    fn poll_vectored_write(&mut self, vec: &[&IoVec], ctx: &mut Context)
        -> Poll<usize, Self::Error> { ... }

    fn poll_close(&mut self, ctx: &mut Context) -> Poll<(), Self::Error>;
}
```

In terms of the old `futures` task model, the `Waker` is analogous to the `Task`
returned by [`task::current`][current], and the `wake` method is analogous to
the old [`Task::notify`][notify].

Astute readers will notice other changes in the traits shown above.
These will be addressed in later sections.

[current]: https://docs.rs/futures/0.1.16/futures/task/fn.current.html
[notify]: https://docs.rs/futures/0.1.16/futures/task/struct.Task.html#method.notify

## Crate Separation

The `futures` crate will be split into several different libraries.

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

## Small, Breaking API Improvements and Clarifications

- `Task` will be renamed to `Waker`. This name more clearly reflects how
  the type is used. It is a handle used to awaken tasks-- it is not a `Task`
  itself.
- `Async` variant `NotReady` will be renamed to `Pending`.
- `AsyncRead` and `AsyncWrite` no longer have `io::Read` and `io::Write`
  bounds. Discussion on this can be found [in this Github issue.](https://github.com/tokio-rs/tokio-io/issues/83)
- `AsyncRead::prepare_uninitialized_buffer` has been replaced with an
  `initializer` function similar to [the unstable one on `io::Read`.](https://doc.rust-lang.org/std/io/trait.Read.html#method.initializer)
  This resolves the issue of the `unsafe` on `prepare_uninitialized_buffer`
  being `unsafe` to implement, *not* `unsafe` to use
  (the more common, and arguably the correct meaning of `unsafe fn`).
- `AsyncRead` and `AsyncWrite` now offer separate methods for
  non-vectorized and vectorized IO operations. This makes the API easier to
  understand, makes the traits object-safe, and removes the dependency on
  the `bytes` crate.
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
