# future-by-example

This document is intended to let readers start working with Rust's `Future` quickly. Some other
useful reading includes:

- [The official Tokio documentation][tokio]
- [Zero-cost futures in Rust][zero]
- [Tokio Internals: Understanding Rust's asynchronous I/O framework from the bottom up][internals]

[tokio]: https://tokio.rs/
[zero]: https://aturon.github.io/blog/2016/08/11/futures/
[internals]: https://cafbit.com/post/tokio_internals/

## `Future`

The [`Future` trait][future] from [`futures`][futures] represents an asynchronous operation that
can fail or succeed, producing a value either way. It is like an async version of
[`Result`][result]. This document assumes that the reader is familiar with `Result`, which is
[covered][result in trpl] in the second edition of *The Rust Programming Language*.

One of the most common questions about `Future` seems to be, "how do I get the value out of it?"
The easiest way to do this is to call the `wait` method. This runs the `Future` in the current
thread, blocking all other work until it is finished.

This is not frequently the best way to run a `Future`, because no other work can happen
until the `Future` completes, which completely defeats the point of using asynchronous
programming in the first place. However, it can be useful in unit tests, when debugging, or at
the top level of a simple application.

See the section on reactors for better ways to run a `Future`.

[future]: https://docs.rs/futures/*/futures/future/trait.Future.html
[futures]: https://docs.rs/futures
[result]: https://doc.rust-lang.org/std/result/
[result in trpl]: https://doc.rust-lang.org/book/second-edition/ch09-02-recoverable-errors-with-result.html

```rust
extern crate futures;
extern crate future_by_example;

fn main() {
    use futures::Future;
    use future_by_example::new_example_future;

    let future = new_example_future();

    let expected = Ok(2);
    assert_eq!(future.wait(), expected);
}
```

A `Future` can be modified using many functions analogous to those of `Result`, such as `map`,
`map_err`, and `then`. Here's `map`:

```rust
extern crate futures;
extern crate future_by_example;

fn main() {
    use futures::Future;
    use future_by_example::new_example_future;

    let future = new_example_future();
    let mapped = future.map(|i| i * 3);

    let expected = Ok(6);
    assert_eq!(mapped.wait(), expected);
}
```

Like a `Result`, two `Future`s can be combined using `and_then` and `or_else`:

```rust
extern crate futures;
extern crate future_by_example;

fn main() {
    use futures::Future;
    use future_by_example::{new_example_future, new_example_future_err, ExampleFutureError};

    let good = new_example_future();
    let bad = new_example_future_err();
    let both = good.and_then(|good| bad);

    let expected = Err(ExampleFutureError::Oops);
    assert_eq!(both.wait(), expected);
}
```

`Future` also has a lot of functions that have no analog in `Result`. Because we're talking
about aynchronous programming, now we have to choose whether we want to run two independent
operations one after the other (in sequence), or at the same time (in parallel).

For example, to get the results of two independent `Future`s, we *could* use `and_then` to run
them in sequence. However, that strategy is silly, because we are only making progress on one
`Future` at a time. Why not run both at the same time?

`Future::join` creates a new `Future` that contains the results of two other `Future`s.
Importantly, both of the input `Future`s can make progress at the same time. The new `Future`
completes only when both input `Future`s complete. There's also `join3`, `join4` and `join5` for
joining larger numbers of `Future`s.

```rust
extern crate futures;
extern crate future_by_example;

fn main() {
    use futures::Future;
    use futures::future::ok;
    use future_by_example::new_example_future;

    let future1 = new_example_future();
    let future2 = new_example_future();

    let joined = future1.join(future2);
    let (value1, value2) = joined.wait().unwrap();
    assert_eq!(value1, value2);
}
```

Whereas `join` completes when *both* `Future`s are complete, `select` returns whichever
of two `Future`s completes first. This is useful for implementing timeouts, among other things.
`select2` is like `select` except that the two `Future`s can have different value types.

## Creating a `Future`

Many libraries return `Future`s for asynchronous operations such as network calls. Sometimes you
may want to create your own `Future`. Implementing a `Future` from scratch is difficult, but
there are other ways to create futures.

You can easily create a `Future` from a value that is already available using the `ok` function.
There are similiar `err` and `result` methods.

```rust
extern crate futures;

fn main() {
    use futures::Future;
    use futures::future::ok;

    // Here I specify the type of the error as (); otherwise the compiler can't infer it
    let future = ok::<_, ()>(String::from("hello"));
    assert_eq!(Ok(String::from("hello")), future.wait());
}
```

## Futures and types
Working with `Future`s tends to produce complex types. For example, the full type of the
expression below is actually:

```
futures::Map<
  futures::Map<
    futures::Join<
      futures::FutureResult<u64, ()>,
      futures::FutureResult<u64, ()>
    >,
    [closure@src/lib.rs:...]>,
  [closure@src/lib.rs:...]
>
```

That is, for every transformation, we add another layer to the type of our `Future`! This can
sometimes be confusing. In particular, it can be challenging to identify ways to write out the
types that aren't brittle or verbose.

In order to help the Rust compiler do type inference, below we have specify the type of
`expected`. It's much terser than writing the full type out, and adding another operation won't
break compilation.

```rust
extern crate futures;

fn main() {
    use futures::future::ok;
    use futures::Future;

    let expected: Result<u64, ()> = Ok(6);
    assert_eq!(
        ok(5).join(ok(7)).map(|(x, y)| x + y).map(|z| z / 2).wait(),
        expected
    )
}
```

Alternatively, we can make use of `_` to let the Rust compiler infer types for us.

```rust
extern crate futures;

fn main() {
    use futures::future::ok;
    use futures::Future;
    use futures::Map;

    let expected: Result<_, ()> = Ok(6);
    let twelve: Map<_, _> = ok(5).join(ok(7)).map(|(x, y)| x + y);
    assert_eq!(twelve.map(|z| z / 2).wait(), expected)
}
```

Rust requires that all types in function signatures are specified.

One way to achieve this for functions that return `Future`s is to specify the full return
type in the function signature. However, specifying the exact type can be verbose, brittle, and
difficult.

It would be nice to be able to define a function like this:

```
fn make_twelve() -> Future<Item=u64, Error=()> {
    unimplemented!()
}
```

However, the compiler doesn't like that:

```
error[E0277]: the trait bound `futures::Future<Item=u64, Error=()>: std::marker::Sized` is not satisfied
   --> src/lib.rs:119:13
    |
119 |         let twelve = make_twelve();
    |             ^^^^^^ `futures::Future<Item=u64, Error=()>` does not have a constant size known at compile-time
    |
    = help: the trait `std::marker::Sized` is not implemented for `futures::Future<Item=u64, Error=()>`
    = note: all local variables must have a statically known size
```

This can be solved by wrapping the return type in a `Box`. One day, this will be solved in a
more elegant way with the currently unstable [impl Trait][impl trait] functionality.

[impl trait]: https://internals.rust-lang.org/t/help-test-impl-trait/6516

```rust
extern crate futures;

fn main() {
    use futures::Future;
    use futures::future::ok;

    fn make_twelve() -> Box<Future<Item=u64, Error=()>> {

        ok(5).join(ok(7)).map(|(x, y)| x + y).boxed()
    }

    let twelve = make_twelve();
    assert_eq!(twelve.map(|z| z / 2).wait(), Ok(6))
}
```

Unlike functions, closures do not require all types in their signatures to be explicitly
defined, so they don't need to be wrapped in a `Box`.

```rust
extern crate futures;

fn main() {
    use futures::Future;

    let make_twelve = || {
        use futures::future::ok;

        // We don't need to put our `Future` inside of a `Box` here.
        ok(5).join(ok(7)).map(|(x, y)| x + y)
    };

    let expected: Result<u64, ()> = Ok(6);
    let twelve = make_twelve();
    assert_eq!(twelve.map(|z| z / 2).wait(), expected)
}
```

## A more powerful way to run Futures
Composing a bunch of `Futures` into a single `Future` and calling `wait` on it is a simple and
easy method as long as you only need to run a single `Future` at a time. However, if you only
need to run a single `Future` at a time, perhaps you don't need the `futures` crate in the first
place! The `futures` crate promises to efficiently juggle many concurrent tasks, so let's
see how that might work.

The [`tokio-core`][tokio-core] crate has a struct called [`Core`][core] which can run multiple
`Future`s concurrently. `Core::run` runs a `Future`, returning its value. Unlike `Future::wait`,
though, it allows the `Core` to make progress on executing other `Future` objects while `run`
running. The `Future` in `Core::run` is the main event loop, and it may request that new
`Future`s be run by calling `Handle::spawn`. Note that the `Future`s run by `spawn` don't get to
return a value; they exist only to perform side effects.

[tokio-core]: https://docs.rs/tokio-core
[core]: https://docs.rs/tokio-core/*/tokio_core/reactor/struct.Core.html

```rust
extern crate futures;
extern crate tokio_core;

fn main() {
    use tokio_core::reactor::Core;
    use futures::future::lazy;

    let mut core = Core::new().unwrap();
    let handle = core.handle();
    let future = lazy(|| {
        handle.spawn(lazy(|| {
            Ok(()) // Ok(()) implements FromResult
        }));
        Ok(2)
    });
    let expected: Result<_, ()> = Ok(2usize);
    assert_eq!(core.run(future), expected);
}
```


License: MIT/Apache-2.0
