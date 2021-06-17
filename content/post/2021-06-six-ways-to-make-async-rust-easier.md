---
title: "Exploring ways to make async Rust easier"
slug: "six-ways-to-make-async-rust-easier"
date: 2021-06-17
author: Carl Lerche
draft: false
---

Asynchronous Rust is powerful but has a reputation for being hard to learn.
There have been various ideas on how to fix the trickiest aspects, though with
my focus being on [Tokio 1.0](https://tokio.rs/blog/2020-12-tokio-1-0), I had
not been able to dedicate much focus to those topics. However, Niko's [async
vision](https://rust-lang.github.io/wg-async-foundations/vision.html) effort has
recently started the discussion again, so I thought I would take some time to
participate.

In this article, I collect some previously proposed ideas and offer some new
ones, tying them together to explore what could be. This exploration isn't a
proposal but a thought experiment: what could we do if we didn’t have to worry
about the status quo. Making a significant change to Rust would be disruptive.
We would need a rigorous way to determine the pros and cons to determine that
the churn is worth it. I also urge you to approach the article with an open
mind. I expect some aspects will generate an immediate adverse reaction. Try to
suppress it and approach it with an open mind.

Before exploring different paths asynchronous Rust could take, we should first
understand when to use asynchronous programming styles. After all, asynchronous
programming is more challenging than just using threads. So what is the benefit?
Some might say the reason is performance and that asynchronous code is faster
because threads are too heavyweight. Reality is more nuanced. Using threads for
I/O based application can be faster depending on details. For example, a
threaded [echo server](https://github.com/carllerche/echo-benchmarks) is
[faster](https://github.com/carllerche/echo-benchmarks#results) than an
asynchronous version when there less than about 100 concurrent connections.
After that, the threaded version starts dropping off, but not drastically.

I believe that the better reason for asynchronous is it enables modeling complex
flow control efficiently. For example, patterns like pausing or canceling an
in-flight operation are challenging without asynchronous programming.
Furthermore, with threads, coordinating between connections requires
synchronization primitives and starts adding contention. With asynchronous
programming, we avoid adding synchronization by operating on multiple
connections on the same thread.

Rust's asynchronous model does a phenomenal job of enabling us to model complex
flow control. For example, [mini-redis' subscribe
command](https://github.com/tokio-rs/mini-redis/blob/master/src/cmd/subscribe.rs#L94-L156)
implementation is very concise and elegant. Yet, it isn't all sunshine and
rainbows. When getting started, users of asynchronous Rust report a confusing
learning curve. While getting started feels straightforward, it is easy to
stumble on unexpected sharp edges. Niko and others have been doing a fantastic
job cataloging the sharp edges as part of the [async vision
effort](https://github.com/tokio-rs/valuable/pull/49). While there are
opportunities to improve on several fronts, I believe that the biggest issue
with asynchronous Rust is that it can violate the principle of least surprise.

Let's start with an example.
[Alan](https://rust-lang.github.io/wg-async-foundations/vision/characters/alan.html)
is learning Rust, has read the Rust book and the Tokio guide, and wants to write
a toy chat server. He opts for a simple line-based protocol and encodes each
line with a length prefix. His line parsing function looks like this:

```rust
async fn parse_line(socket: &TcpStream) -> Result<String, Error> {
    let len = socket.read_u32().await?;
    let mut line = vec![0; len];
    socket.read_exact(&mut line).await?;
    let line = str::from_utf8(line)?;
    Ok(line)
}
```

This code looks very much like blocking Rust code except for the `async` and
`await` keywords. Even though Alan has never written Rust before, he can read
this function and understand how it behaves, or so he thinks. When testing
locally, his chat server appears to work, so he sends a link to Barbara.
Unfortunately, after chatting for a bit, the server crashes with an "invalid
UTF-8" error. Alan is baffled; he inspects his code and finds no apparent bugs.

So what is the problem? It turns out that the task uses a `select!` higher up in
the call stack:

```rust
loop {
    select! {
        line_in = parse_line(&socket) => {
            if let Some(line_in) = line_in {
                broadcast_line(line_in);
            } else {
                // connection closed, exit loop
                break;
            }
        }
        line_out = channel.recv() => {
            write_line(&socket, line_out).await;
        }
    }
}
```

Suppose a message is received on `channel` while `parse_line` is waiting for
more data, the `select!` statement aborts the `parse_line` operation, losing
in-progress parsing state as a result. In the following loop iteration,
`parse_line` is called again and starts from the middle of a frame, resulting in
reading gibberish.

Therein lies the problem: any async Rust function may stop running at any time
if the caller cancels it, and unlike with blocking Rust, cancellation is a
typical asynchronous operation. Worse, no affordance guides new users to
discover this behavior.


# A Shiny Future

What if we could fix this so that asynchronous Rust meets the learner's
expectations at each step? If behavior must deviate from expectations, then
there must be an affordance to point the learner in the right direction.
Additionally, we want to minimize surprises during the learning process,
especially early on.

Let's start by fixing the unexpected cancellation problem by saying that async
functions always run to completion (first proposed
[here](https://github.com/Matthias247/rfcs/pull/1)). With completion-guaranteed
futures, Alan learns that asynchronous Rust behaves like blocking Rust, but with
the addition of the `async` and `await` keywords. Spawning tasks adds
concurrency, and channels coordinate between tasks. Instead of `select!` taking
arbitrary async statements, it works with channels and channel-like types (e.g.,
`JoinHandle`).

With completion-guaranteed futures, Alan's chat server looks like this.

```rust
async fn handle_connection(socket: TcpStream, channel: Channel) {
    let reader = Arc::new(socket);
    let writer = reader.clone();
    
    let read_task = task::spawn(async move {
        while let Some(line_in) in parse_line(&reader).await {
            broadcast_line(line_in);
        }
    });
    
    loop {
        // `channel` and JoinHandle are both "channel-like" types.
        select! {
            res = read_task.join() => {
                // The connection closed, exit loop
                break;
            }
            line_out = channel.recv() => {
                write_line(&writer, line_out).await;
            }
        }
    }
}
```

The code is similar to the earlier example, but because all async statements
must complete and `select!` only accepts channel-like types, the call to
`parse_line()` moves to a spawned task. This change prevents the error Alan
encounters with today's asynchronous Rust. Should Alan try to call
`parse_line()` in the `select!` statement, he would get a compiler error with a
recommendation to spawn a task. Select's channel-like requirement ensures that
it is safe to abort losing branches. Channels can store values, and receiving
the value is atomic. Losing select branches does not lose data on cancelation.

# Cancelation

What happens if writing encounters an error? Today, `read_task` keeps running.
Instead, Alan wants an error to result in gracefully closing the connection and
all tasks. Unfortunately, this is where we start running into design challenges.
When we can drop any async statement at any time, cancellation is trivial: drop
the future. We need a way to cancel an in-flight operation as this ability is
one of the primary reasons to use asynchronous programming. To achieve this,
`JoinHandle` provides a `cancel()` method:

```rust
async fn handle_connection(socket: TcpStream, channel: Channel) {
    let reader = Arc::new(socket);
    let writer = reader.clone();
    
    let read_task = task::spawn(async move {
        while let Some(line_in) in parse_line(&reader).await? {
            broadcast_line(line_in)?;
        }
        
        Ok(())
    });
    
    loop {
        // `channel` and JoinHandle are both "channel-like" types.
        select! {
            _ = read_task.join() => {
                // The connection closed or we encountered an error,
                // exit the loop
                break;
            }
            line_out = channel.recv() => {
                if write_line(&writer, line_out).await.is_err() {
                    read_task.cancel();
                    read_task.join();
                }
            }
        }
    }
}
```

But what does `cancel()` do? It cannot immediately abort the task because async
statements are now guaranteed to complete. We do want the canceled task to stop
processing and return as soon as possible. Instead, all resource types within
the canceled task will stop operating and return an "interrupted" error. Further
attempts to use them will also result in errors. This strategy is similar to
[Kotlin](https://kotlinlang.org/docs/cancellation-and-timeouts.html), except
that Kotlin raises an exception. If `read_task` is awaiting `socket.read_u32()`
in `parse_line()` when canceled, then the `read_u32()` function returns
immediately with `Err(io::ErrorKind::Interrupted)`. The `?` operator bubbles up
the task and results in the task terminating.

At first glance, this behavior may also seem like the task arbitrarily stops
running, but this is not the case. To Alan, the current async Rust abort
behavior appears as the task hangs indefinitely. By forcing resources, such as
sockets, to return an error on cancellation, Alan can follow the cancelation
flow. Alan may choose to add `println!` statements or use other debugging
strategies to understand better how the canceled task terminates.

## AsyncDrop

Unbeknownst to Alan, his chat server is avoiding most syscalls by using
io-uring. Asynchronous Rust can transparently leverage the io-uring API thanks
to completion-guaranteed futures and `AsyncDrop`. When Alan drops the
`TcpStream` value at the end of `handle_connection()`, the socket is
asynchronously closed. To do this, the `TcpStream` includes the following
`AsyncDrop` implementation.


```rust
impl AsyncDrop for TcpStream {
    async fn drop(&mut self) {
        self.uring.close(self.fd).await;
    }
}
```

Niko already has a [plausible
proposal](https://hackmd.io/bKfiVPRpTvyX8JK_Ng2EWA?view) for using `async fn` in
traits. The remaining open question is how to handle the implicit `.await`
point. Currently, asynchronously awaiting a future requires an `.await` call. An
`AsyncDrop` trait would add a hidden yield point added by the compiler when a
value goes out of scope within an async context. This behavior would violate the
principle of least surprise. Why would there be implicit await points if others
are explicit?

One proposal to solve the "sometimes implicit drop" problem requires an explicit
function call to perform the drop asynchronously.

```rust
my_tcp_stream.read(&mut buf).await?;
async_drop(my_tcp_stream).await;
```

Of course, what happens if the user forgets the async drop call? After all, the
compiler handles dropping with blocking Rust, and this is a powerful feature.
Also, note how the snippet above has a subtle bug: the `?` operator skips the
`async_drop` call on a read error. The Rust compiler could provide warnings that
catch the problem, but what is the fix? Is there a way to make `?` compatible
with an explicit `async_drop`? 

# Get rid of .await

What if, instead of requiring explicit calls to async drop, we remove the
`await` keyword? Alan would no longer have to use `.await` after calling an
async function (e.g., `socket.read_u32().await`). When calling an async function
within an async context, the `.await` becomes implicit.

This may seem like a big departure from today’s Rust, and it is. But it is good
for us to question our assumptions. It’s interesting to look at .await in light
of the criteria laid out in Aaron’s “[Reasoning
footprint](https://blog.rust-lang.org/2017/03/02/lang-ergonomics.html)” blog
post. Implicit `.await` has limited applicability and context-dependence by only
occurring within async statements. Alan only has to look at the function
definition to know he is within an async context. Additionally, it would be easy
for an IDE to highlight yield points.

Removing `.await` has another benefit: it brings async Rust in line with
blocking Rust. The concept of blocking is already implicit. With blocking Rust,
we don't write `my_socket.read(buffer).block?`. When he writes his asynchronous
chat server, the only noticeable difference to Alan becomes the need to annotate
his functions with the `async` keyword. Alan can intuit the execution of the
async code. The issue of "lazy futures" goes away, and Alan cannot accidentally
do the following and wonder why "two" prints first.

```rust
async fn my_fn_one() {
    println!("one");
}

async fn my_fn_two() {
    println!("two");
}

async fn mixup() {
    let one = my_fn_one();
    let two = my_fn_two();
    
    join!(two, one);
}
```

The “.await” RFC did include some discussion of implicit await. At that time,
the most compelling argument against implicit awaits was that the await calls
annotated points at which the async statement could be aborted. This argument
would apply less with completion-guaranteed futures. Of course, with abort-safe
async statements, should the await keyword remain? This question would need to
be answered. Regardless, removing “.await” would be a big change and would need
to be approached cautiously. Usability studies would need to demonstrate that
the pros outweigh the cons.

# Scoped tasks

So far, Alan can build his chat server with asynchronous Rust without learning
many new concepts or hitting unexpected behavior. He learned about `select!` but
the compiler enforces selecting over channel-like types. Besides that, Alan
added `async` to his functions, and it just worked. He shows his code to
[Barbara](https://rust-lang.github.io/wg-async-foundations/vision/characters/barbara.html)
and asks about needing to wrap the socket with an `Arc`. Barbara suggests he
looks into scoped tasks as a way to avoid the allocation.

A scoped task is the asynchronous equivalent of crossbeam's [scoped
threads](https://docs.rs/crossbeam/0.8.1/crossbeam/thread/index.html): a task
that can borrow data owned by the spawner. Alan can use a scoped task to avoid
the Arc in his connection handler.

```rust
async fn handle_connection(socket: TcpStream, channel: Channel) {
    task::scope(async |scope| {
        let read_task = scope.spawn(async || {
            while let Some(line_in) in parse_line(&socket)? {
                broadcast_line(line_in)?;
            }

            Ok(())
        });
        
        loop {
            // `channel` and JoinHandle are both "channel-like" types.
            select! {
                _ = read_task.join() => {
                    // The connection closed or we encountered an error,
                    // exit the loop
                    break;
                }
                line_out = channel.recv() => {
                    if write_line(&writer, line_out).is_err() {
                        break;
                    }
                }
            }
        }
    });
}
```

The key to ensuring safety is guaranteeing that the scope outlives all tasks
spawned within the scope, or, in other words, ensuring that async statements
complete. There is a downside. Enabling scoped tasks requires making
"Future::poll" unsafe as polling the future to completion becomes required for
memory safety (covered in Matthias’
[RFC](https://github.com/Matthias247/rfcs/pull/1)). The addition of unsafe would
make implementing `Future` by hand much more difficult. As a mitigation, we need
to remove the need for virtually all users to implement `Future` by hand,
including when implementing traits like `AsyncRead` and `AsyncIterator`. I
believe this is an achievable goal.

Besides scoped tasks, ensuring async statement completion enables passing
pointers from the task value to the kernel when using io-uring or integrating
with C++ futures. In some cases, it may also be possible to avoid the allocation
when spawning sub-tasks, which could be helpful in embedded contexts, though it
would require a slightly different API than above.

## Add concurrency by spawning.

With today's asynchronous Rust, applications can add concurrency by spawning a
new task, using `select!` or `FuturesUnordered`. So far, we have discussed
spawning and `select!`. I propose removing `FuturesUnordered` as it is a common
source of bugs. When using `FuturesUnordered`, it is easy to spawn tasks into
it, expecting they will run in the background, then be surprised when they don't
make any progress (see [status quo
story](https://rust-lang.github.io/wg-async-foundations/vision/status_quo/aws_engineer/solving_a_deadlock.html)).

Instead, we can provide a similar utility using scoped tasks.

```
let greeting = "Hello".to_string();

task::scope(async |scope| {
    let mut task_set = scope.task_set();
    
    for i in 0..10 {
        task_set.spawn(async {
            println!("{} from task {}", greeting, i);
            
            i
        });
    }
    
    async for res in task_set {
        println!("task completed {:?}", res);
    }
});
```

Each spawned task runs concurrently, borrows data from the spawning task, and
the `TaskSet` utility provides a similar API as `FuturesUnordered` without the
hazard. Other primitives, such as buffered streams, can also be implemented on
top of scoped tasks.

There is an opportunity to explore new concurrency primitives on top of these
primitives. For example, structured concurrency, similar to Kotlin, would now be
possible. Matthias has [previously
explored](https://github.com/tokio-rs/tokio/issues/1879) this space, but
asynchronous Rust's current model makes it impossible to achieve. By switching
asynchronous Rust to guarantee completion, we unlock this whole area.

# What about `select!`?

At the beginning of the article, I claimed that using asynchronous programming
enables us to model complex flow control efficiently. The most effective
primitive we have today is `select!`. I also proposed in this article to reduce
`select!` only to accept channel-like primitives, forcing Alan to spawn two
tasks per connection to read and write concurrently. While spawning a task does
help prevent bugs when canceling the read operation, it also would have been
possible to rewrite the read operation to tolerate unexpected cancellation. For
example, when parsing frames in
[mini-redis](https://github.com/tokio-rs/mini-redis/blob/master/src/connection.rs#L56),
we first store received data in a buffer. When canceling the read operation, no
data is lost because it is in the buffer. The next call to read will resume from
where we left off. The Mini-Redis read operation is abort-safe.

What if, instead of limiting `select!` to channel-like types, we restrict it to
abort-safe operations. Receiving from a channel is abort-safe, but so is reading
from a buffered I/O handle. The point here is, instead of assuming that all
asynchronous operations are abort-safe, we require the developer to opt-in by
adding `#[abort_safe]` (or `async(abort)`) to their function definition. This
opt-in strategy has a few advantages. First, when Alan is learning asynchronous
Rust, he doesn't need to know anything about abort safety. It is possible to
implement everything without the concept by spawning to get concurrency.

```rust
#[abort_safe]
async fn read_line(&mut self) -> io::Result<Option<String>> {
    loop {
        // Consume a full line from the buffer
        if let Some(line) = self.parse_line()? {
            return Ok(line);
        }

        // Not enough data has been buffered to parse a full line
        if 0 == self.socket.read_buf(&mut self.buffer)? {
            // The remote closed the connection.
            if self.buffer.is_empty() {
                return Ok(None);
            } else {
                return Err("connection reset by peer".into());
            }
        }
    }
}
```

Instead of having the default be abort-safe statements, it becomes opt-in. This
opt-in strategy follows the existing pattern around unwind safety. When a new
developer jumps into the code, the annotation informs them that the function
must uphold the abort safety guarantees. The rust compiler may even provide
additional checks and warnings for functions annotated with `#[abort_safe]`.

Now Alan can use his `read_line()` function from a loop with "select!".

```rust
loop {
    select! {
        line_in = connection.read_line()? => {
            if let Some(line_in) = line_in {
                broadcast_line(line_in);
            } else {
                // connection closed, exit loop
                break;
            }
        }
        line_out = channel.recv() => {
            connection.write_line(line_out)?;
        }
    }
}
```

# Mixing abort-safe and non-abort-safe

The `#[abort_safe]` annotation introduces two variants of asynchronous
statements. Mixing abort-safe with non-abort-safe statements requires special
consideration. Calling an abort-safe function is always possible, whether from
an abort-safe or non-abort-safe context. However, the Rust compiler will prevent
calling non-abort-safe functions from an abort-safe context and provide a
helpful error message.

```rust
async fn must_complete() { ... }

#[abort_safe]
async fn can_abort() {
    // Invalid call => compiler error
    must_complete();
}
```

```rust
async fn must_complete() { ... }

#[abort_safe]
async fn can_abort() {
    // Valid call
    spawn(async { must_complete() }).join();
}
```

The developer can always spawn a new task to bridge a non-abort-safe function
with an abort-safe context.

Including two flavors of asynchronous statements adds complexity to the
language, but this complexity appears late in the learning curve. When learning
asynchronous Rust, the default asynchronous statement is non-abort-safe. From
this context, the learner can call asynchronous functions regardless of
abort-safety. Abort-safety will appear in late chapters of asynchronous Rust
tutorials as an optional advanced topic.

# Getting there

Shifting from the current, abort-safe by default, asynchronous model to
completion-guaranteed will require a new Rust edition. For discussion's sake,
let's say that Rust edition 2026 introduces this change. The `Future` trait in
this edition would be changed to model completion-guaranteed futures and would
not be compatible with the trait from older versions. Instead, the old trait
exists in the 2026 edition as `AbortSafeFuture` (naming TBD).

With the 2026 edition, adding `#[abort_safe]` to an asynchronous statement would
generate an `AbortSafeFuture` implementation instead of `Future`. Any async
function written with pre-2026 editions implements the `AbortSafeFuture` trait,
making all existing async code compatible with the new edition (recall that an
abort-safe statement is callable from any context).

Upgrading an old codebase to completion-guaranteed async Rust requires adding
`#[abort_safe]` to all async statements. This transition is a mechanical
process, and a tool can automate the process. Runtimes like Tokio will probably
need to issue a breaking change to support the new completion-guaranteed
asynchronous Rust.

# Parting thoughts

I discussed several potential changes for Rust. To briefly recap, they were:

*   Async functions guarantee completion.
*   Remove the `await` keyword.
*   Introduce an `#[abort_safe]` annotation to mark async functions that are safe to abort.
*   Limit `select!` to accept only abort-safe branches.
*   Cancel spawned tasks by disabling resources from completing.
*   Support scoped tasks.

I believe these changes could significantly simplify asynchronous Rust, though
making the change comes at the cost of disrupting the status quo. Before making
any decisions, we need more data. What percentage of today’s asynchronous code
is abort-safe? Can we run usability studies to evaluate the benefit of these
changes? How much cognitive load would having two flavors of async statements
add to Rust?

I’m also hoping that this exploration will spark discussion, and perhaps others
will propose alternative directions. Now is the time to come up with wild ideas
for where Rust could go. Either write up articles or submit [shiny
future](https://rust-lang.github.io/wg-async-foundations/vision/shiny_future.html)
PRs to the [wg-async-foundations
repository](https://github.com/rust-lang/wg-async-foundations).

And finally…

![What if we remove await](/img/remove-await.jpg)

