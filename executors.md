# Summary
[summary]: #summary

This RFC proposes a design for `futures-executors`, including both executor
traits and built-in executors. In addition, it sets up a core expectation for all
tasks that they are able to spawn additional tasks, while giving fine-grained
control over what executor that spawning is routed to.

NOTE: this RFC assumes that [RFC #2] is accepted.

[RFC #2]: https://github.com/rust-lang-nursery/futures-rfcs/pull/2

# Motivation
[motivation]: #motivation

This is a follow-up to the [Tokio Reform], [Futures 0.2], and [Task Context]
RFCs, harmonizing their designs.

[Tokio Reform]: https://github.com/tokio-rs/tokio-rfcs/pull/3
[Futures 0.2]: https://github.com/rust-lang-nursery/futures-rfcs/pull/1
[Task Context]: https://github.com/rust-lang-nursery/futures-rfcs/pull/2

The design in this RFC has the following goals:

- Add a core assumption that tasks are *always* able to spawn additional tasks,
  avoiding the need to reflect this at the API level. (Note: this assumption is
  only made in contexts that also assume an allocator.)

- Provide fine-grained control over which executor is used to fulfill that assumption.

- Provide reasonable built-in executors for both single-threaded and
  multithreaded execution.

It brings the futures library into line with the design principles of the [Tokio
Reform], but also makes some improvements to the `current_thread` design.

# Proposed design

## The core executor abstraction: `Spawn`

First, in `futures-core` we define a general purpose executor interface:

```rust
pub trait Executor {
    fn spawn(&self, f: Box<Future<Item = (), Error = ()> + Send>) -> Result<(), SpawnError>;

    /// Provides a best effort **hint** to whether or not `spawn` will succeed.
    ///
    /// This allows a caller to avoid creating the task if the call to `spawn` will fail. This is
    /// similar to `Sink::poll_ready`, but does not provide any notification when the state changes
    /// nor does it provide a **guarantee** of what `spawn` will do.
    fn status(&self) -> Result<(), SpawnError> {
        Ok(())
    }

    // Note: also include hooks to support downcasting
}

// opaque struct
pub struct SpawnError { .. }

impl SpawnError {
    pub fn at_capacity() -> SpawnError { .. }
    pub fn is_at_capacity(&self) -> bool { .. }

    pub fn shutdown() -> SpawnError { .. }
    pub fn is_shutdown(&self) -> bool { .. }

    // ...
}
```

The `Executor` trait is pretty straightforward (some alternatives and tradeoffs
are discussed in the next section), and crucially is object-safe. Executors can
refuse to spawn, though the default surface-level API glosses over that fact.

We then build in an executor to the task context, stored internally as a trait
object, which allows us to provide the following methods:

```rust
impl task::Context {
    // A convenience for spawning onto the current default executor,
    // **panicking** if the executor fails to spawn
    fn spawn<F>(&self, F) -> Result<(), SpawnError>
        where F: Future<Item = (), Error = ()> + Send + 'static;

    // Get direct access to the default executor, which can be used
    // to deal with spawning failures
    fn executor(&self) -> &mut Executor;
}
```

With those APIs in place, we've achieved two of our goals: it's possible for any
future to spawn a new task, and to exert fine-grained control over generic task
spawning within a sub-future.

## Built-in executors

In the `futures-executor` crate, we then add two executors: `ThreadPool` and
`LocalPool`, providing multi-threaded and single-threaded execution,
respectively.

### Multi-threaded execution: `ThreadPool`

The `ThreadPool` executor works basically like `CpuPool` today:

```rust
struct ThreadPool { ... }
impl Spawn for ThreadPool { ... }

impl ThreadPool {
    // sets up a pool with the default number of threads
    fn new() -> ThreadPool;
}
```

Tasks spawn onto a `ThreadPool` will, by default, spawn any subtasks onto the
same executor.

### Single-threaded execution: `LocalPool`

The `LocalPool` API, on the other hand, is a bit more subtle. Here, we're
replacing `current_thread` from [Tokio Reform] with a slightly different, more
flexible design that integrates with the default executor system.

First, we have the basic type definition and executor definition, which is much
like `ThreadPool`:

```rust
// Note: not `Send` or `Sync`
struct LocalPool { ... }
impl Spawn for LocalPool { .. }

impl LocalPool {
    // create a new single-threaded executor
    fn new() -> LocalPool;
}
```

However, the rest of the API is more interesting:

```rust
impl LocalPool {
    // runs the executor until `f` is resolved, spawning subtasks onto `spawn`
    fn run_until<F, S>(&self, f: F, spawn: S) -> Result<F::Item, F::Error>
        where F: Future, S: Spawn;

    // a future that resolves when *all* spawned futures have resolved
    fn all_done(&self) -> impl Future<Item = (), Error = ()>;

    // spawns a possibly non-Send future, possible due to single-threaded execution.
    fn spawn_local<F>(&self, F)
        where F: Future<Item = (), Error = ()>;
}
```

The `LocalPool` is always run until a particular future completes execution, and
lets you *choose* where to spawn any subtasks. If you want something like
`current_thread` from [Tokio Reform], you:

- Use `all_done()` as the future to resolve, and
- Use the `LocalPool` *itself* as the executor to spawn subtasks onto by default.

On the other hand, if you are trying to run some futures-based code in a
synchronous setting (where you'd use `wait` today), you might prefer to direct
any spawned subtasks onto a `ThreadPool` instead.


# Rationale, drawbacks and alternatives
[alternatives]: #alternatives

The core rationale here is that, much like the global event loop in [Tokio
Reform], we'd like to provide a good "default executor" to all tasks. Using
`task::Context`, we can provide this assumption ergonomically without using TLS,
by essentially treating it as *task*-local data.

The design here is unopinionated: it gives you all the tools you need to control
the executor assumptions, but doesn't set up any particular preferred way to do
so. This seems like the right stance for the core futures library; external
frameworks (perhaps Tokio) can provide more opinionated defaults.

This stance shows up particularly in the design of `LocalPool`, which differs
from `current_thread` in providing flexibility about executor routing and
completion. It's possible that this will re-open some of the footguns that
`current_thread` was trying to avoid, but the explicit `spawn` parameter and
`all_done` future hopefully make things more clear.

Unlike `current_thread`, the `LocalPool` design does not rely on TLS, instead
requiring access to `LocalPool` in order to spawn. This reflects a belief that,
especially with borrowng + async/await, most spawning should go through the
default executor, which will usually be a thread pool.

Finally, the `Spawn` trait is more restrictive than the futures 0.1 executor
design, since it is tied to boxed, sendable futures. That's necessary when
trying to provide a *universal* executor assumption, which needs to use dynamic
dispatch throughout. This assumption is avoided in `no_std` contexts, and in
general one can of course use a custom executor when desired.

# Unresolved questions
[unresolved]: #unresolved-questions

TBD
