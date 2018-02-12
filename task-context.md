- Feature Name: task-context
- Start Date: 2018-01-22
- RFC PR:
- Rust Issue:

# Summary
[summary]: #summary

Learnability, reasoning, debuggability and `no_std`-compatibility improvements achieved by
making task wakeup handles into explicit arguments.

# Motivation
[motivation]: #motivation

## Learnability

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
It's essential to remember to do this, and to ensure that the right schedulings
occur on every code path.

[explicit task issue]: https://github.com/alexcrichton/futures-rs/issues/129

## Reasoning about code

The implicit `Task` argument makes it difficult to tell which functions
aside from `poll` will use the `Task` handle to schedule a wakeup. That has a
few implications.

First, it's easy to accidentally call a function that will attempt to access
the task outside of a task context. Doing so will result in a panic, but it 
would be better to detect this mistake statically.

Second, and relatedly, it is hard to audit a piece of code for which calls
might involve scheduling a wakeup--something that is critical to get right
in order to avoid "lost wakeups", which are far harder to debug than an
explicit panic.

## Debuggability

Today's implicit TLS task system provides us with fewer "hooks" for adding debugging
tools to the task system. Deadlocks and lost wakeups are some of the trickiest things
to track down with futures code today, and it's possible that by using a "two stage"
system like the one proposed in this RFC, we will have more room for adding debugging 
hooks to track the precise points at which queing for wakeup occurred, and so on.

More generally, the explicit `Context` argument and distinct `Waker` type provide greater
scope for extensibility of the task system.

## no_std

While it's possible to use futures in `no_std` environments today, the approach
is somewhat of a hack. Threading an explicit `&mut Context` argument makes no_std
support entirely straightforward, which may also have implications for the futures
library ultimately landing in libcore.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `Future`, `Stream`, `AsyncRead`, and `AsyncWrite` traits have been modified
to take explicit `task::Context` arguments. `task::Context` is a type which
provides access to
[task-locals](https://docs.rs/futures/*/futures/macro.task_local.html) and a `Waker`.

In terms of the old `futures` task model, the `Waker` is analogous to the `Task`
returned by [`task::current`][current], and the `wake` method is analogous to
the old [`Task::notify`][notify].

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The `task::Context` interface looks like this:

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
    unsafe fn initializer(&self) -> Initializer { ... }

    fn poll_read(&mut self, buf: &mut [u8], ctx: &mut Context)
        -> Poll<usize, io::Error>;

    fn poll_vectored_read(&mut self, vec: &mut [&mut IoVec], ctx: &mut Context)
        -> Poll<usize, io::Error> { ... }
}

pub trait AsyncWrite {
    fn poll_write(&mut self, buf: &[u8], ctx: &mut Context)
        -> Poll<usize, io::Error>;

    fn poll_vectored_write(&mut self, vec: &[&IoVec], ctx: &mut Context)
        -> Poll<usize, io::Error> { ... }

    fn poll_close(&mut self, ctx: &mut Context) -> Poll<(), io::Error>;
}
```

# Drawbacks
[drawbacks]: #drawbacks

The primary drawback here is an ergonomic hit, due to the need to explicitly thread through 
a `Context` argument.

For combinator implementations, this hit is quite minor.

For larger custom future implementations, it remains possible to use TLS internally to recover the existing ergonomics, if desired.

Finally, and perhaps most importantly, the explicit argument is never seen by users who only use
futures via combinators or async-await syntax, so the ergonomic impact to such users in minimal.


# Rationale and alternatives
[alternatives]: #alternatives

The two-stage design (splitting `Context` and `Waker`) is useful both for efficial task-local storage, and to provide space for debugging "hooks" in later iterations of the library.

# Unresolved questions
[unresolved]: #unresolved-questions

- Can we come up with a better name than `task::Context`? `Task` is
  confusing because it suggests that we're being passed the task,
  when really we're _in_ the task. `TaskContext` is roughly equivalent
  to `task::Context` but doesn't allow users to `use futures::task::Context;`

- What is the best way to integrate the explicit `task::Context` with
  async/await generators? If generators are changed to accept arguments to
  `resume`, a pointer could be passed through that interface. Otherwise, it
  would be necessary to pass the pointer through TLS when calling into
  generators.

- The `task::Context::with_local` function can now likely be replaced by
  `HashMap`-like entry and accessor APIs. What should the precise API
  surface for accessing task-locals look like?
