---
title: "Tower Web — Expanding the middleware stack"
slug: "tower-web-middleware"
date: 2018-09-07
author: Carl Lerche
draft: false
---

[`tower-web`](http://github.com/carllerche/tower-web) [version
0.2.2](https://crates.io/crates/tower-web/0.2.2) has been released. It comes
with a number of new features, which I will talk about in this post. Primarily,
the middleware story is starting to come together. I will be expanding some on
how middleware fits into Tower and web in general.

First, a quick recap
====================

[Tower](https://github.com/tower-rs/tower) is a library for writing modular and
reusable networking services (previously announced
[here](https://medium.com/@carllerche/announcing-tower-a-library-for-writing-robust-network-services-with-rust-67273f052c40)).
It does this by defining a simple trait representing asynchronous request /
response based services. It then provides a number of components that add
functionality such as retries, load balancing, logging, etc.. all based around
the `Service` trait.

[Tower Web](https://github.com/carllerche/tower-web) is an HTTP web framework
built using the tower stack (previously announced
[here](https://github.com/carllerche/tower-web)). While it aims to be full
featured, functional, and production ready, it also is an experiment driving
improvements to the Tower stack. Soon (in software time estimation terms), it
will be merged with [Warp](http://github.com/seanmonstar/warp) (more on that
later).

Web services
============

Defining a web service with Tower Web is pretty straightforward. Here is a
simple one:

```rust
impl_web! {
    impl HelloWorld {
        #[get("/")]
        fn hello_world(&self) -> Result<&'static str, ()> {
            Ok("hello world")
        }
    }
}

let service = ServiceBuilder::new()
    .resource(HelloWorld)
    .build();
```

The returned `service` is a value implementing the Tower
[`Service`](https://docs.rs/tower-service/0.1.0/tower_service/trait.Service.html)
trait. This trait takes the application logic provided by `HelloWorld` and
exposes it as an asynchronous function of an HTTP request to an HTTP response.
As a user, we can use the `service` value directly:

```rust
let request = http::Request::builder()
    .method("GET")
    .uri("http://example.com/")
    .body(())
    .unwrap();

let response_future = service.call(request);
let response = response_future.wait().unwrap();
```

More importantly, it provides a simple, common interface for Rust HTTP server
libraries to use. Any server library is able to take a value that implements
`Service` and run it. Applications are able to implement their logic decoupled
from the specific HTTP server implementation.

Middleware
==========

One great feature that comes from having a standardized `Service` trait is the
ability to define middleware. Middleware is used to decorate the application,
providing additional functionality.

A middleware is a value that contains a `Service` and implements `Service`
itself. The middleware’s `Service` implementation receives a request and passes
the request to the inner service. This allows the implementation to inject logic
either before sending the request to the inner service or after the inner
service responds. The middleware can mutate the request or the response at
either point.

Let’s update our service to add some middleware.

```rust
let service = ServiceBuilder::new()
    .resource(HelloWorld)
    .middleware(LogMiddleware::new("hello_world::web"))
    .middleware(DeflateMiddleware::new(Compression::best()))
    .build();
```

The returned `service` value still implements Tower `Service` and we can still
use it the same way as before. Now, when a request is sent, the request will be
logged and the response body will be compressed using zlib.

A simple implementation of `LogService` might look like this:

```rust
struct LogService<S> {
    inner: S,
}

impl<S, RequestBody> Service for LogService<S>
where S: Service<Request = http::Request<RequestBody>>,
{
    type Request = S::Request;
    type Response = S::Response;
    type Error = S::Error;
    type Future = S::Future;

    fn poll_ready(&mut self) -> Poll<(), Self::Error> {
        self.inner.poll_ready()
    }

    fn call(&mut self, request: Self::Request) -> Self::Future {
        info!("{} - {}", request.method(), request.uri());
        self.inner.call(request)
    }
}
```

Here, the request is logged before passing it to the inner service. The actual
implementation is [more
involved](https://github.com/carllerche/tower-web/blob/master/src/middleware/log/service.rs)
as we want to log the request once it has been processed so that we know things
like if the response was successful and how long it took.

To implement `DeflateMiddleware`, we want to do two things:

*   Inject a Content-Encoding header in the response.
*   Compress the response body, ideally without buffering the entire body up front.

Injecting the header is [easy
enough](https://github.com/carllerche/tower-web/blob/f018fe28ac0edbd71069845ba60bf88bf1cbe78b/src/middleware/deflate/service.rs#L68-L72),
but compressing the body is trickier. The Tower `Service` trait does not set any
restrictions on the request or response types and the `http::Response` type is
generic over the body. We need to set some sort of bound on the body in order to
work with it.

`The BufStream trait`
=====================

To solve the problem of “what should the request and response bodies be bound
by?”, Tower Web introduces a new trait: BufStream. The trait is an asynchronous
stream of values that implement
[`Buf`](https://docs.rs/bytes/0.4.10/bytes/trait.Buf.html) (such a creative
name).

A `Buf` type already abstracts values that contain raw bytes, whether those
bytes are in sequential memory or stored in fancier data structures like ropes.
A `BufStream` describes a value that asynchronously produces any number of `Buf`
instances, where each `Buf` instance represents a chunk of the stream.

Byte containers such as `&static [u8]`, `Vec<u8>`, and `Bytes`, can all
implement `BufStream`, themselves. They will just yield a single `Buf`
containing all of their data.

We will also be able to implement `BufStream` for existing data types such as
the Hyper [`Body`](https://docs.rs/hyper/0.12.9/hyper/struct.Body.html) or
[`futures::sync::mpsc::Receiver<T>`](https://docs.rs/futures/0.1/futures/sync/mpsc/struct.Receiver.html)
`where T: Buf`.

To implement the `DeflateMiddleware`, we can now use `BufStream` to implement
[`CompressStream`](https://github.com/carllerche/tower-web/blob/master/src/util/buf_stream/deflate.rs),
a type that takes a `T: BufStream` and compresses it. Now, the deflate
middleware just needs to modify the response to [wrap the body stream
with](https://github.com/carllerche/tower-web/blob/f018fe28ac0edbd71069845ba60bf88bf1cbe78b/src/middleware/deflate/service.rs#L66)
[`CompressStream`](https://github.com/carllerche/tower-web/blob/f018fe28ac0edbd71069845ba60bf88bf1cbe78b/src/middleware/deflate/service.rs#L66).

The Middleware trait
====================

As described above, a middleware is “just” a type that implements `Service` and
dispatches the request to the inner service. We could build up our middleware
stack like this:

```rust
let service = LogService::new(
    DeflateService::new(
        MyBusinessLogic::new()
    )
);
```

But that is no fun. We want a pretty builder API:

```rust
let service = ServiceBuilder::new()
    .resource(HelloWorld)
    .middleware(LogMiddleware::new("hello_world::web"))
    .middleware(DeflateMiddleware::new(Compression::best()))
    .build();
```

To achieve this, there is one final missing piece: the `Middleware` trait.

```rust
pub trait Middleware<S> {
    /// The wrapped service request type
    type Request;

    /// The wrapped service response type
    type Response;

    /// The wrapped service's error type
    type Error;

    /// The wrapped service
    type Service: Service<Request = Self::Request,
                         Response = Self::Response,
                            Error = Self::Error>;

    /// Wrap the given service with the middleware, returning a new service
    /// that has been decorated with the middleware.
    fn wrap(&self, inner: S) -> Self::Service;
}
```

The job of a `Middleware` implementation is to handle all of that annoying
wrapping so that the end user doesn’t have to. For example, the
[`LogMiddleware`](https://github.com/carllerche/tower-web/blob/master/src/middleware/log/middleware.rs#L30-L32)
[implementation](https://github.com/carllerche/tower-web/blob/master/src/middleware/log/middleware.rs#L30-L32)
takes the inner service `S` and returns `LogService<S>`.

Putting it all together, to implement a piece of logging middleware, the
following is needed:

*   `LogService` — Handles the logging logging.
*   `LogMiddleware` — Adds logging to the middleware stack.
*   `ResponseFuture` — If the middleware needs to see the response or take an
    action when the request has been handled.

Looking at the middleware implementations to date, it takes a bit of boilerplate
to get everything setup. Step one is to get everything working. Step two is to
iterate and improve things. Already, [Nikolay Kim](https://github.com/fafhrd91)
(of [Actix](https://github.com/actix)) has submitted a
[PR](https://github.com/tower-rs/tower/pull/104) that should cut down on
boilerplate. It adds combinators for combining service implementations.
Hopefully, once all this shakes out, the only type that will be needed to
implement by hand will be `LogMiddleware`.

The wrap function could look like:

```rust
fn wrap(&self, inner: S) -> Self::Service {
    service_fn(|request| {
        info!("request = {:?}", request);
        request
    }).and_then(inner)
}
```

Ok, it might take a few more Rust language features before we get to that, but
we are heading in the right direction!

Other new features
==================

Middleware isn’t the only new thing. There have been two other changes of note.

First [@lnicola](http://github.com/lnicola) added support for the
`#[web(either)]` annotation on derived response types. It is common for a
handler to require returning one of multiple response types picked at run time.
The way to handle this is by defining an enum. However, up until today, that
then required manual boilerplate to define `Response` for the enum. Now, that
boilerplate can be avoided. For example:

```rust
#[derive(Response)]
#[web(either)]
enum FileOrMsg {
    File(tokio::fs::File),
    Msg(String),
}

impl_web! {
    impl MyResource {
        #[get("/:foo")]
        fn index(&self, foo: String) -> impl Future<Item = FileOrMsg> {
            if foo == "file" {
                Ok(FileOrMsg::File(load_file()))
            } else {
                Ok(FileOrMsg::Msg("foo".into()))
            }
        }
    }
}
```

The enum is not limited to just two variants. So, go wild.

The second new feature of note, contributed by
[@manifest](https://github.com/manifest), is the ability to store arbitrary
configuration data when defining a service. The configuration data can be
retrieved in `Extract` implementations. This especially useful for
authentication where the user must be validated using secrets set at
configuration time. For example:

```rust
struct State {
    secret: String,
}

struct Resource;
struct User;

impl<B: BufStream> Extract<B> for User {
    type Future = Immediate<User>;

    fn extract(context: &Context) -> Self::Future {
        let state = context.config::<State>().unwrap();
        ...
    }
}

impl_web! {
    impl Resource {
        #[get("/")]
        fn action(&self, user: User) {
            ...
        }
    }
}

fn main() {
    ...
    ServiceBuilder::new()
        .config(State { secret: "secret".to_owned() })
        .resource(Resource)
        .run(&addr)
        .unwrap();
}
```

The [issue](https://github.com/carllerche/tower-web/issues/96) had some good
discussion.

Roadmap
=======

There has been lots of good work to date and the work will continue. I wanted to
close out this post with some thoughts on next steps.

Immediate work will focus on extracting `BufStream` into `tokio-buf`. There
already is a [PR](https://github.com/tokio-rs/tokio/pull/611) open for that,
though it requires more work. All the middleware infrastructure described in
this post currently lives directly in the tower-web repository. It also needs to
be extracted out.

The `Middleware` trait will most likely go in the `tower-middleware` crate. All
HTTP specific concerns will move to `tower-http`.

Once these components are extracted, then we can start focusing on the plan for
merging with Warp. We are already hitting [some
cases](https://github.com/carllerche/tower-web/issues/82) where we’d really like
to integrate with the Warp filters. Once `Middleware` and `BufStream` are
extracted, I would like to focus on writing up details on the merge plan.

Conclusion
==========

I believe that the abstractions and implementations evolving out of Tower Web
are promising and should help take the story for writing asynchronous services
in Rust one step further. However, more validation is needed. We (the Rust
community) must start pushing with real world applications.

I hope to see more people getting involved and join the
[Gitter](http://gitter.im/tower-rs/tower) channel. 😃