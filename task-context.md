- Feature Name: task-context
- Start Date: 2018-01-22
- RFC PR:
- Rust Issue:

# Summary
[summary]: #summary

Learnability, readability, and `no_std`-compatibility improvements achieved by
making task wakeup handles into explicit arguments.

# Motivation
[motivation]: #motivation

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
is the source of bugs or confusion. Storing the `Task` in TLS is also slightly
more ergonomic when writing manual `Future` implementations since you don't
have to manually plumb through a wakeup handle.

However, as described earlier, this ergonomic benefit comes at the cost of
discoverability: it's necessary to read and understand written documentation
in order to use or implement the `poll` function directly. Additionally, the
implicit argument makes it more difficult to tell which functions aside from
`poll` will use the `Task` handle to schedule a wakeup. Finally, and perhaps
most importantly, the explicit argument is never seen by users who only use
futures via combinators or async-await syntax, so the ergonomic impact to
such users in minimal.

This RFC proposes to resolve these issues by making the `Task` wakeup handle
into an explicit argument.

[explicit task issue]: https://github.com/alexcrichton/futures-rs/issues/129

## Other Nonlearnability-based Benefits

Since this structure no longer uses TLS, it also makes it easier to support
the use of `futures` in `no_std` environments.

The `&mut Context` makes it possible to provide guaranteed exclusive access
to task-local values, so it will no longer be necessary tu use `RefCell` or
similar in mutable task-locals.

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

# Drawbacks
[drawbacks]: #drawbacks

- Ergonomic loss due to the need to pass an explicit parameter.

# Rationale and alternatives
[alternatives]: #alternatives

- Don't require an explicit parameter and use TLS instead (status quo).

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
