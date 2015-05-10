# Decomposing Hello World

> Our first Iron server, and learning to understand it.

## Hello World!

Without further ado, here is our "Hello World" server:

```rust
// src/main.rs
extern crate iron;

use iron::prelude::*;
use iron::status;

fn main() {
    Iron::new(|req: &mut Request| {
        Ok(Response::with((status::Ok, "Hello World")))
    }).http("localhost:3000").unwrap();
}
```

Don't forget to include `iron` as a dependency in your `Cargo.toml`.

## Unpacking Our Code

There is a surprising amount to unpack in even such a short snippet. Iron
is made of several modular components, and has a very extensible design.
The resulting Iron code is often dense and quite flexible, providing a lot of
semantic meaning for little syntax.

Let's break it down a bit:

```rust
extern crate iron;

use iron::prelude::*;
use iron::status;
```

Here we're just bringing in iron and `use`ing the symbols we'll need,
including all of the symbols in the `prelude` module of `iron`. The `prelude`
module contains many of the most common things you'll need when using `iron`,
including the `Request`, `Response`, and `Iron` types. It harbors some other
stuff too, but we will come back to it later.

```rust
fn main() {
    Iron::new(/*..*/).http("localhost:3000").unwrap();
}
```

This part of the code is responsible for initializing then actually
starting the server.

We are listening using the HTTP protocol on localhost, port 3000. The `http`
method on `Iron` will accept anything which can be parsed into a `std::net::SocketAddr`,
and start listening, returning any errors.

The more interesting part here is the call to `Iron::new`. Let's take a look at
the signature of `Iron::new`:

```rust
impl<H> Iron<H> where H: Handler {
    pub fn new(handler: H) -> Iron<H>;
}
```

## Handlers

What is this `Handler` trait?

```rust
pub trait Handler: Send + Sync + Any {
    fn handle(&self, &mut Request) -> IronResult<Response>;
}
```

`Handler`s are the central component of Iron - they are responsible for receiving a
`Request` and producing a `Response`. You can think of a `Handler` as sort of
analogous to a "controller" from an MVC framework, but with less baggage and no
real limitations on what you might use it for.

The instance of a type implementing `Handler` which we pass to `Iron::new` will
be the main `Handler` used for our server. All incoming requests will be passed
to that `Handler` and the responses it returns will be sent back to clients.

In this case, our `Handler` is a closure; closures (and fns) with the
appropriate signature can be used as handlers, since there is an implementation
of `Handler` for all `F` which implement `Fn(&mut Request) -> IronResult<Response>`.

Now let's dig into our `Handler`, the closure we pass to `Iron::new`:

```rust
|req: &mut Request| -> IronResult<Response> {
    Ok(Response::with((status::Ok, "Hello World")))
}
```

I've annotated the return type for clarity, but it's not necessary to do so.

`IronResult` is simply an alias for `Result` with a specific error type, so we
wrap our `Response` value in `Ok` to make it an `IronResult<Response>` instead
of just a plain `Response`.

The meat of the code here is the call to `Response::with`, which actually
constructs the `Response` we will be writing back to the requesting client.
Let's take a look at the source of `Response::with`:

```rust
pub fn with<M: Modifier<Response>>(modifier: M) -> Response {
    Response::new().set(modifier)
}
```

There are a few things going on here, so again, we'll tackle them piece by
piece. Let's focus on the signature first, specifically the `M` type parameter,
which must implement `Modifier<Response>`.

## Modifiers

`Modifier`s are one of the core extension kinds in Iron. They allow you to
define new ways to make changes to other types. In Iron, the `Request` and
`Response` types are the target of most `Modifier`s.

To understand how `Modifier`s work, which we'll need to do before we can
understand `Response::with`, let's check out the `Modifier` trait:

```rust
pub trait Modifier<T> {
    fn modify(self, &mut T);
}
```

Pretty simple, right? All a `Modifier` can do is perform some action on
another type. The [modifier crate](https://github.com/reem/rust-modifier),
where `Modifier` comes from, also defines another trait, `Set`, which can be used
to apply modifiers.

`Set` looks like this:

```rust
pub trait Set {
    fn set<M: Modifier<Self>>(mut self, modifier: M) -> Self where Self: Sized {
        modifier.modify(&mut self);
        self
    }

    fn set_mut<M: Modifier<Self>>(&mut self, modifier: M) -> &mut Self {
        modifier.modify(self);
        self
    }
}
```

In Iron, both `Request` and `Response` implement `Set`, which allows you to
chain modifiers very easily, both through owned `Request`s and `Response`s and
through mutable references to the same.

If we look back the implementation of `Response::with`, we see that all it does
is create a new blank `Response` using `Response::new`, and then apply the
passed-in modifier to it.

The last piece of the puzzle is figuring out how our tuple that we pass to
`Response::with` actually implements `Modifier`. There are actually a few
different impls that we need to be aware of to figure this out.

There are several impls for concrete types, including `iron::status::Status`
and `&str`, the types we used, in the [`iron::modifiers`](https://github.com/iron/iron/blob/master/src/modifiers.rs)
module. Unfortunately, rustdoc is terrible at advertising these impls, so the
best way to get familiar with them is to look at the source.

Additionally, there is an impl of `Modifier` for tuples of `Modifier`s in the
`modifier` crate, so we can use `(Status, &str)` as a `Modifier`.

## Putting it all back together

Now that we've decomposed this small example sufficiently, let's put it all
back together - here is our server again for clarity:

```rust
extern crate iron;

use iron::prelude::*;
use iron::status;

fn main() {
    Iron::new(|req: &mut Request| {
        Ok(Response::with((status::Ok, "Hello World")))
    }).http("localhost:3000").unwrap();
}
```

We pull in iron and the things we'll need from it, create a new server using
`Iron::new`, pass our `Handler` (a closure) to it, and start listening.

Our `Handler` will return a new `Response` which has had its status set to
`status::Ok` (traditionally known as 200) and its body set to `Hello World`
by applying a `Modifier`.

We now have a running Iron server, and an understanding of all the fundamental
parts. That said, there is much more to explore in the upcoming chapters!

