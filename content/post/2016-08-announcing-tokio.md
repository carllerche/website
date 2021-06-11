---
title: "Announcing Tokio"
description: "A Finagle inspired network application framework for Rust."
slug: "announcing-tokio"
date: 2016-08-03
author: Carl Lerche
draft: false
---

I’m very excited to announce a project that has been a long time in the making.
[Tokio](https://github.com/tokio-rs/tokio) is a network application framework
for rapid development and highly scalable deployments of clients and servers in
the [Rust programming language](http://rust-lang.org).

It strives to make writing robust, scalable, and **production ready** network
clients and servers as easy as possible. It does this by focusing on small and
reusable components… and by being really, really, fast.

Go check it out
[https://github.com/tokio-rs/tokio](https://github.com/tokio-rs/tokio). I’ll
wait.

Welcome back. Let’s start our tour of Tokio.

Service Trait
=============

Writing composable and reusable components is made possible by standardizing the
interface that clients and servers use. In Tokio, this standardized interface is
the \`Service\` trait and it is the cornerstone of Tokio’s building blocks. This
trait is heavily inspired (Ok, stolen) from
[Finagle](https://twitter.github.io/finagle).

Here is what the Service trait looks like:

```rust
pub trait Service: Send + 'static {
    type Req: Send + 'static;

    type Resp: Send + 'static;

    type Error: Send + 'static;

    type Fut: Future<Item = Self::Resp, Error = Self::Error>;

    fn call(&self, req: Self::Req) -> Self::Fut;
}
```

A relatively simple trait, but it unlocks a world of possibility. This symmetric
and uniform API reduces your [server or client into a
function](https://monkey.org/~marius/funsrv.pdf), thus enabling middleware (or
filters, depending on your lingo).

A Service is an asynchronous function from Request to Response, where
asynchronicity is managed by the brilliant
[Futures](https://crates.io/crates/futures) crate.

A middleware is a component that acts both as a client and a server. It
intercepts a request, modifies it, and passes it on to an upstream Service
value.

Let us use timeouts as an example, something every network application requires.

This Timeout middleware, shown above, only has to be written once, then it can
be inserted as part of any network client for any protocol, as shown below.

Now, the really interesting thing is that the exact same middleware could be
used in a server as well, shown below.

Tokio’s Reactor
===============

The Service trait addresses how all the various network services fit together to
build one cohesive application, but it doesn’t care about the details of how
each service is built. This is where the Tokio reactor fits in.

The Tokio reactor is a non-blocking runtime built on top of
[Mio](http://github.com/carllerche/mio). It has the job of scheduling tasks to
run based on readiness notifications of the underlying sockets.

Let us start with a simple example: listening on a socket and accepting inbound
connections.

Look at all the things that I’m not doing. This is all non-blocking code. There
is no interest explicitly registered with the reactor, there are no callbacks
passed around, everything happens right there in the \`tick\` function.

This is where I think Tokio does something new and interesting (if this has been
done before, I am not aware of it).

Instead of requiring that you explicitly register interest in a socket in order
to receive readiness notifications, as Mio requires, Tokio observes which
sources (in this case, a TcpListener) the task depends on by noticing which
sources the task uses. In other words, by simply using your sockets, timers,
channels, etc… you are expressing readiness interest to the reactor.

So, in this example, the first time the reactor calls the Listener task,
\`self.socket.accept()\` will return \`Ok(None)\` which indicates that the
listener socket is not ready to complete the accept operation. However, the
listener socket will also register interest in itself with the reactor. When the
listener becomes ready, the task is invoked again by the reactor. This time the
accept will succeed. The reactor is able to make its observations with virtually
no performance overhead. You can [see for
yourself](https://github.com/tokio-rs/tokio/blob/master/src/reactor/reactor.rs)
if you don’t believe me.

This pattern works out nicely for more [complex
cases](https://github.com/tokio-rs/tokio/blob/master/src/proto/pipeline/server.rs#L39-L108)
as well. I’m looking forward to seeing what people come up with.

Community
=========

Tokio is now open to the world while still at a very early stage. This is
intentional. **Release early, release often**. I am hoping to get the Rust
community involved now to shape the direction of Tokio and to get the protocol &
middleware ecosystem jump started. It is up to you to build these reusable
components and share them.

Do you want a Cassandra driver? Write it. Do you want an HTTP 2 server? Do it!

A number of Rust community members have already jumped in to help shape Tokio to
what it is today.

[Sean McArthur](http://seanmonstar.com) of [Hyper](http://hyper.rs/) has already
started working to integrate the Tokio reactor in Hyper for the 0.10 release.
His feedback has been invaluable. We already have a [proof of
concept](http://github.com/tokio-rs/tokio-hyper) HTTP server providing a Service
trait based API.

[Tikue](http://github.com/tikue) is working on getting his RPC framework
[Tarpc](https://github.com/google/tarpc) working on Tokio using the provided
building blocks. We’ve already discovered further components Tokio could provide
to reduce the amount of time he has to spend writing networking code.

This is the beginning, not the end. There is still so much to do and I hope that
we can all do it together. If you want to build something with Tokio, get
involved. Talk to me and the other members of the Tokio community. Jump in
[Gitter](https://gitter.im/tokio-rs/tokio) and start a conversation. Let’s build
this **together**.