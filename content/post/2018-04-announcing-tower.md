---
title: "Announcing Tower"
description: "A library for writing robust network services with Rust."
slug: "announcing-tower"
date: 2018-04-06
author: Carl Lerche
draft: false
---

Tower ([Github](https://github.com/tower-rs/)) has been a bit of an open secret
for a while now, but it has been getting some mentions recently after the Rust
all hands, so I thought I would write a more formal post to talk about it, the
goals, and the short term roadmap.

tl;dr
=====

Tower is a library for writing robust network services with Rust. It is being
built in service of the [Conduit](http://github.com/runconduit/conduit) proxy,
which is using the Tokio ecosystem to build the world’s smallest, fastest, most
secure network proxy. Roughly, the main components are:

*   A [service
    abstraction](https://github.com/tower-rs/tower/blob/11b591b6e037ae671bc8da755cfb3095a3f83568/src/lib.rs#L156)
    for request / response based protocols.
*   A collection of [protocol agnostic
    middleware](https://github.com/tower-rs/tower/tree/11b591b6e037ae671bc8da755cfb3095a3f83568).

Tower will  also provide a batteries included experience for implementing HTTP and gRPC services.

*   `tower-web` will be a server web framework, focusing on ergonomics and ease
    of use. Allowing users to rapidly build out production ready HTTP services
    based on the tower stack.
*   `[tower-grpc](https://github.com/tower-rs/tower-grpc)` is a
    [gRPC](http://grpc.io/) client and server based on the tower stack. It takes
    a proto file and generates the stubs for you. It uses
    [Prost](https://github.com/danburkert/prost) (a really great library) for
    the protobuf part.

And, of course, Tower is built on top of the [Tokio](http://tokio.rs/) stack.

Extracting what is common across protocols
==========================================

The fundamental premise upon which Tower is built is that all client and server
implementations for request / response based protocols require the same set of
functionality in order to be robust and production ready. The goal of Tower is
to extract all of this commonality into a set of abstractions and components.

This is not a new idea. There are similar projects for other languages that have
proven out that this idea works out. The most prominent is
[Finagle](https://twitter.github.io/finagle/) for Scala.

The first step is to define a core abstraction: the service.

```rust
pub trait Service {
    /// Requests handled by the service.
    type Request;

    /// Responses given by the service.
    type Response;

    /// Errors produced by the service.
    type Error;

    /// The future response value.
    type Future: Future<Item = Self::Response, Error = Self::Error>;

    /// Returns `Ready` when the service is able to process requests.
    ///
    /// If the service is at capacity, then `NotReady` is returned and the task
    /// is notified when the service becomes ready again. This function is
    /// expected to be called while on a task.
    ///
    /// This is a **best effort** implementation. False positives are permitted.
    /// It is permitted for the service to return `Ready` from a `poll_ready`
    /// call and the next invocation of `call` results in an error.
    fn poll_ready(&mut self) -> Poll<(), Self::Error>;

    /// Process the request and return the response asynchronously.
    ///
    /// This function is expected to be callable off task. As such,
    /// implementations should take care to not call `poll_ready`. If the
    /// service is at capacity and the request is unable to be handled, the
    /// returned `Future` should resolve to an error.
    fn call(&mut self, req: Self::Request) -> Self::Future;
}
```

This service trait provides the abstraction representing a handling a single
request. Both the request and the response types are generic. One way to think
of a `Service` is as an asynchronous function of `Request`to `Response`.

If you are thinking that this is similar to `tokio-service` then you would be
right. That was a first attempt, but it was put on hold as Tokio itself was
developed.

An HTTP hello world service might look like this:

```rust
impl Service for HelloWorld {
    type Request = http::Request<String>;
    type Response = http::Response<String>;
    type Error = ();
    type Future = future::FutureResult<Self::Response, Self::Error>;
    
    fn poll_ready(&mut self) -> Poll<(), Self::Error> {
        Ok(Async::Ready(()))
    }
    
    fn call(&mut self, request: http::Request<String>) -> Self::Future {
        let response = http::Response::builder()
            .status(200)
            .header("content-length", "11")
            .body("hello world".into_string())
            .unwrap();
            
        Ok(response).into_future()
    }
}
```

Now, HTTP libraries like [Hyper](http://hyper.rs/) can run any service that
implements the tower trait for the HTTP request and response types.

If you are thinking that this is a lot of boilerplate, you would be right again.
This is where `tower-web` comes in, I will get to that soon.

Middleware
==========

Because services are now implemented using a standard trait, these
implementations can be decorated to add further behavior. This is where
middleware comes in.

A piece of middleware is any type that takes a `T: Service` and implements
`Service` itself adding some sort of additional behavior.

Tower [already comes](https://github.com/tower-rs/tower) with a set of
middleware that can be used with any protocol and mixed / matched to cover the
needs of the application. Some middleware that we already wrote includes:

*   `[tower-timeout](https://github.com/tower-rs/tower/blob/master/tower-timeout/src/lib.rs)`takes
    a service and requires the response to complete within the specified
    duration. If it takes too long, the request is timed out.
*   `[tower-balance](https://github.com/tower-rs/tower/blob/master/tower-balance/src/lib.rs)`
    distributes the incoming requests across a number of inner service
    instances.
*   `[tower-buffer](https://github.com/tower-rs/tower/blob/master/tower-buffer/src/lib.rs)`
    specifies a max number of concurrent requests that the inner service can
    handle. When the max is reached, it will stop accepting new requests until
    in-flight ones complete.

Tower Web
=========

Having to manually define service implementations and configure middleware
stacks requires a lot of boilerplate and boilerplate isn’t fun. The thing is, in
**most** cases, an application is going to be built on a pretty standard set of
middleware. To make it as easy as possible to just write applications, Tower
will provide a batteries included, ergonomic web framework.

When using Tower Web, it will be trivial to get your web service defined and out
of the box it will be production grade, building on top of the Tokio and Tower
stacks.

The code for Tower Web has not yet been opened up. There is still a bit of churn
there, but keep an eye out for more announcements really soon!

Tower gRPC
==========

Tower gRPC has already been opened up and can be found
[here](https://github.com/tower-rs/tower-grpc). We use it in Conduit. The
easiest way to get a feel for how it works is to look at the
[examples](https://github.com/tower-rs/tower-grpc/tree/master/tower-grpc-examples).
You start with a gRPC [proto
file](https://github.com/tower-rs/tower-grpc/blob/master/tower-grpc-examples/proto/helloworld/helloworld.proto),
which defines the service. Then, as part of `cargo build` you
[convert](https://github.com/tower-rs/tower-grpc/blob/master/tower-grpc-examples/build.rs)
the proto file into Rust stubs. From there, you can use the generated gRPC
[client](https://github.com/tower-rs/tower-grpc/blob/master/tower-grpc-examples/src/helloworld/client.rs#L53-L55)
or implement the
[server](https://github.com/tower-rs/tower-grpc/blob/master/tower-grpc-examples/src/helloworld/server.rs).
This is what the server half looks like:

```rust
impl server::Greeter for Greet {
    type SayHelloFuture = future::FutureResult<Response<HelloReply>, tower_grpc::Error>;

    fn say_hello(&mut self, request: Request<HelloRequest>) -> Self::SayHelloFuture {
        println!("REQUEST = {:?}", request);

        let response = Response::new(HelloReply {
            message: "Zomg, it works!".to_string(),
        });

        future::ok(response)
    }
}
```

Also, Tower gRPC is a pure Rust implementation. So there is no binding to
external C or C++ libraries. It uses [h2](http://github.com/carllerche/h2) under
the hood for the HTTP/2.0 protocol layer.

How to get involved
===================

If you look at the various READMEs for the linked crates, you will find some
discouraging language recommending that you don’t use it. The Tower stack is
still in its very early days, but we are using it today successfully in Conduit.
An initial, official, release of Tower will hopefully happen by summer. That
said, if you are willing to pull up your sleeves, and become a contributor, you
should feel free to start using it today.