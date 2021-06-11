---
title: "Tower Web — A new web framework for Rust"
slug: "announcing-tower-web"
date: 2018-08-09
author: Carl Lerche
draft: false
---

I previously announced [Tower]({{< ref "/post/2018-04-announcing-tower" >}}) and
mentioned that a web framework was in the works. It took longer than I had hoped
(as it sometimes does with software), but today, I am opening up [Tower
Web](https://github.com/carllerche/tower-web).

Tower Web is an asynchronous HTTP web framework that focuses on removing
boilerplate. It is built on top of [**Tokio**](https://tokio.rs/),
[**Hyper**](http://hyper.rs/), and of course
[**Tower**](https://github.com/tower-rs/tower). It works today on **stable**
Rust.

Here is “hello world”.

```rust
#[macro_use]
extern crate tower_web;

use tower_web::ServiceBuilder;

#[derive(Clone, Debug)]
struct HelloWorld;

impl_web! {
    impl HelloWorld {
        /// @get("/")
        fn hello_world(&self) -> Result<String, ()> {
            Ok("Hello world".to_string())
        }
    }
}

pub fn main() {
    let addr = "127.0.0.1:8080".parse().expect("Invalid address");
    println!("Listening on http://{}", addr);

    ServiceBuilder::new()
        .resource(HelloWorld)
        .run(&addr)
        .unwrap();
}
```

Tower Web uses a macro based approach. The `impl_web!` macro looks at the impl
blocks and generates the necessary glue code to serve `HelloWorld` as an HTTP
service.

The `@get("/")` doc comment is meaningful. This tells Tower Web that GET
requests to `/` should be mapped to the `hello_world` method. Tower Web opted to
use doc comments as annotations to enable supporting stable Rust today. Once
attribute macros land on the stable channel, Tower Web will switch to attribute
macros.

This hello world example is cool, but it doesn’t illustrate where Tower Web
really shines.

Decoupling HTTP and application logic
=====================================

Tower Web strives to enable applications to be written without including any
HTTP related concerns. That means, methods should be written using only “plain
old Rust types” ([PORT](https://en.wikipedia.org/wiki/Plain_old_Java_object)?).
To illustrate, lets look at an example that uses a path capture.

```rust
#[derive(Clone, Debug)]
struct HelloWorld;

impl_web! {
    impl HelloWorld {
        /// @get("/hello/:name")
        fn greet(&self, name: String) -> Result<String, ()> {
            Ok(format!("Hello, {}", name))
        }
    }
}
```

This example extracts a request path segment and uses that as part of the
response. What is interesting is that the `greet` method is implemented without
any special types. It just takes a `String` as the argument.

Tower Web uses convention over configuration. The convention is that the
argument identifier will match the capture identifier. In this case, both the
argument and capture are named `name`. Because of this, Tower Web knows that the
`name` argument should be populated with the value captured from the path.

Because the method is written without any special types, testing also becomes
very easy.

```rust
#[test]
fn test_greet() {
    let web = HelloWorld;

    assert_eq!(
        web.greet("carl".to_string()).unwrap(),
        "Hello, carl");
}
```

When it comes to more complicated types, such as JSON request bodies, Tower Web
also has you covered.

```rust
#[derive(Clone, Debug)]
struct HelloWorld;

#[derive(Debug, Extract)]
struct MyData {
    foo: usize,
    bar: Option<String>,
}

impl_web! {
    impl HelloWorld {
        /// @post("/data")
        fn greet(&self, body: MyData) -> Result<String, ()> {
            Ok(format!("Hello, {:?}", body))
        }
    }
}
```

The `#[derive(Extract)]` generates an extract implementation for the type. It
uses Serde under the hood, so all the various Serde annotations work here too.
When used as an argument named `body`, Tower Web will deserialize the request
body into the struct. Once again, the method has been implemented without any
HTTP specific types.

The following curl request will succeed:

`curl --vv --request POST -H 'content-type: application/json' --data '{"foo":1,"bar":"baz"}' http://localhost:8080/data`

Tower Web is able to do validation on the type as well. The `foo` field is
defined to be numeric and required. Omitting it results in an HTTP 400, bad
request, response.

Responding
==========

A resource method can return any type that implements `Response`. `String` is
one of these types. Some others include `serde_json::Value`, `http::Response`,
and `tokio::fs::File` (for serving a file from disk). Tower Web also provides a
`#derive(Response)` macro enabling custom return types.

```rust
#[derive(Debug, Response)]
#[web(status = "201")]
struct MyData {
    foo: usize,
    bar: Option<String>,
}

impl_web! {
    impl HelloWorld {
        /// @get("/data")
        /// @content_type("json")
        fn greet(&self) -> Result<MyData, ()> {
            Ok(MyData {
                foo: 123,
                bar: None,
            })
        }
    }
}
```

In this example, we respond with a custom type. The resource method is also
annotated with `@content_type("json")`. This indicates that `MyData` should be
serialized using the JSON format.

Also note that `MyData` has an attribute: `#[web(status = "201")]`. This tells
Tower Web to set the HTTP response status code to 201. There are a number of
other ways to customize the response with `#[derive(Response)]` and these are
documented in the API docs and examples.

Asynchronous
============

So far, all the requests have been processed synchronously. However, resource
methods may return futures as well. Here is an example using `impl Future`.

```rust
impl_web! {
    impl HelloWorld {
        /// @get("/async")
        fn hello_future(&self) -> impl Future<Item = String, Error = ()> + Send {
            // async implementation
        }
    }
}
```

Note that `impl Future` is bound with `Send`. Tower Web uses Hyper to serve the
application and Hyper requires the response future to be bound by `Send`.

Just a taste
============

This is just a very quick tour of what Tower Web has to offer (I didn’t even
touch on middleware). To get started, check out the examples and API docs found
at the [Tower Web repository](https://github.com/carllerche/tower-web). Also,
there are a lot more features planned, so try things out, provide feedback, and
get your hands dirty with some PRs.

Warp — the future of asynchronous web for Rust
==============================================

As you might have seen, my esteemed colleague, seanmonstar recently
[announced](https://seanmonstar.com/post/176530511587/warp)
[Warp,](https://github.com/seanmonstar/warp) a framework he has been working on.
We both have been working in parallel and have been experimenting with different
approaches to define web services in Rust.

The goal is to avoid fragmentation and rally behind a single crate. There is no
need to duplicate effort and re-implement the world each time.

In a month or so, Tower Web will become a part of Warp. Warp will then provide
both the filter based API and the macro API offered by Tower Web. This will let
us focus on a single library.

Merging with Warp will also enable mixing and matching the various API styles.
For example, you could build a single service using both filters and macros. Or,
you could respond from a filter with a `#[derive(Response)]` struct.

In the short term, while I gather feedback from the initial release, feature
development will happen in the Tower Web repository.

Acknowledgements
================

I want to call out Jake Goulding ([shepmaster](https://github.com/shepmaster)).
Jake provided a lot of valuable early feedback and has already contributed quite
a good number of PRs ❤️.

Conclusion
==========

Give Tower Web a shot, provide feedback and PRs. Also, I will be blogging again
soon to talk more about the Tower / Web stack!