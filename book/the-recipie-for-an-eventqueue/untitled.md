# Design

The first thing we need to do is to create a library project where we'll organize our code. We'll name our project `minimio.`

Now, whether you've read my previous book or not shouldn't matter much, since we'll create an integration test that defines what we want to accomplish anyway. Run the following commands to create our project:

```text
mkdir minimio
cd minimio
cargo init --lib
```

## Using integration tests to drive the API design

The first think I like to do when I know roughly what API I want to use is to start a new project by writing down the API and how I want to use it in an integration test.

For now we only want to cover one use case and that is to provide a event queue we can use in our [examples-node-eventloop](https://github.com/cfsamson/examples-node-eventloop)

I've looked into how [mio ](https://github.com/tokio-rs/mio)API as an inspiration but for convenience I've made some compromises to simplify our code to fit an example.

{% hint style="info" %}
Reacently, `mio` changed from using IOCP as the backing API on Windows and started using WePoll instead simplifying their implementation greatly. For now, just know that I've used [mio v0.6.x](https://github.com/tokio-rs/mio/tree/v0.6.x) version as the inspiration for the code we use in this book for this exact reason so if you go to dive into the source code, make sure to switch to the `v0.6.x`branch.
{% endhint %}

## Creating an integration test

In Rust, it's common to have unit tests in the same file as code. Integration tests usually goes in a separate `tests`folder, so let's make one. Our folder structure should look like this once done:

```text
minimio
   |
   +--> src
   |     |
   |     +--> lib.rs
   +--> tests
   |      |
   |      +--> api.rs
```

Now open `api.rs`and let's start designing how we want our API to work.

## **Our Requirements**

1. We want to be able to block the current thread while we wait for events
2. We want the same API across operating systems
3. We want to be able to register interest from a different thread than we run the main loop on

_Ok, so before we dive into the code, let's consider what we can immediately derive from these requirements._

## 1. Blocking the current thread

We need an interface we can use which provides a way for us to block the thread while waiting for events to be ready. We also need something that identifies an event, so we know which one finished in what order. We need to be able to send a signal to stop waiting and exit the blocking call or else our program would never exit \(if the event queue is run on our main thread or end in a "bad" way if we run it in a child thread\).

{% hint style="info" %}
* Inpsired by `mio` we call our main event queue instance `Poll`, and the blocking method `poll()`
* We'll call the thing which identifies an event a `Token` \(which will simply be a `usize`in this case\)
* We'll need a structure representing an `Event`
{% endhint %}

## 2. One API for all platforms

This is a difficult one. And it's very difficult to decide on in advance before actually digging into the details of both Epoll, Kqueue and IOCP.

We know that IOCP requires us to provide a buffer which it will fill with data but Epoll and Kqueue let's us know when data is ready to read. At least we can derive from these facts that we need to hook into whatever method the user calls for retrieving data. It seems impossible for us to provide any abstraction lower than that.

Once we realize that we also realize that using `std::net::TcpStream`is out of the question, but we can implement our own `TcpStream`where we "hijack" the `Read`methods \(and `Write`methods if we implemented those as well\). This way we can use `std::net::TcpStream`under the hood but implement our own `Read`implementation.

{% hint style="info" %}
* We need to provide a `TcpStream`
* We need something to represent what kind of event we're interested in. We'll call that `Interests`
{% endhint %}

## 3. Register interest from a different thread

So we want the user to be able to register interest from one thread and block while waiting for events on another thread. That means we need some way of having a basic synchronization to know that our `Poll`instance is actually waiting for events and that we have created an event queue.

It's apparent that this structure will be closely tied to our event queue. There are many ways to solve this but let's just start with a name and an idea of how it should work.

{% hint style="info" %}
* We need a `Registrator`which knows if our event queue is alive and we must be able to send the `Registrator` instance to another thread.
{% endhint %}

## The integration test

So, we have a rough idea about our API now so let's start sketching out a test. We'll divide this into some smaller unit tests later and use this integration test to know when we've reached our goal for now.

Let's start with the imports we know we need from our library:

{% hint style="info" %}
An integration test in Rust uses our library as it would be used by any other user, therefore we can only access the public API of our library.
{% endhint %}

```rust
use minimio::{Events, Interests, Poll, Registrator, TcpStream};
```

The first thing we want to do is to consider how we want to use our API. More specifically we're going to use a `Reactor`and an `Executor`to create a task, suspend it, wait for a `READABLE`event and then resume and finish our task.

{% hint style="info" %}
See the chapter named [The Reactor-Executor Pattern](../appendix-1/reactor-executor-pattern.md) in the appendix for an explanation of this pattern. Most questions that will be answered there.
{% endhint %}

Our library will be used in the `Reactor`so let's focus on that specific part of the code first.

```rust
struct Reactor {
    handle: Option<JoinHandle<()>>,
    registrator: Option<Registrator>,
}

impl Reactor {
    fn new(evt_sender: Sender<usize>) -> Reactor {
        let mut poll = Poll::new().unwrap();
        let registrator = poll.registrator();

        // Set up the epoll/IOCP event loop in a seperate thread
        let handle = thread::spawn(move || {
            let mut events = Events::with_capacity(1024);
            loop {
                println!("Waiting! {:?}", poll);
                match poll.poll(&mut events, Some(200)) {
                    Ok(..) => (),
                    Err(ref e) if e.kind() == io::ErrorKind::Interrupted => break,
                    Err(e) => panic!("Poll error: {:?}, {}", e.kind(), e),
                };

                for event in &events {
                    let event_token = event.id();
                    evt_sender.send(event_token).expect("Send event_token err.");
                }
            }
        });

        Reactor { handle: Some(handle), registrator: Some(registrator) }
    }

    fn registrator(&mut self) -> Registrator {
        self.registrator.take().unwrap()
    }
}
```

**The really interesting parts of this code is these lines:**

```rust
let mut poll = Poll::new().unwrap();
let registrator = poll.registrator();
```

Here we create our event queue by creating a new `Poll`and we get a `Registrator`by calling `poll.registrator()`which we'll use to register interest in events. The `Registrator` is designed to be held in a separate thread from the `Poll`instance.

We want to block and wait for events in a separate thread, so we spawn a new thread and write the following code to be executed on that thread.

**First we create a collection of `Events`:**

```rust
let mut events = Events::with_capacity(1024);
```

The reason for creating the collection here is mainly because it aligns well with how all three operating systems returns events and it's again heavily inspired by how `mio`does this as well.

Next up is the actual blocking call where we wait for events.

```rust
match poll.poll(&mut events, Some(200)) {
    Ok(..) => (),

    Err(ref e) if e.kind() == io::ErrorKind::Interrupted => break,

    Err(e) => panic!("Poll error: {:?}, {}", e.kind(), e),
    };
```

Here we pass in a reference to our event collection and a timeout of 200 ms. The call to `poll`returns a `io::Result`. If the `Error`is of `io::ErrorKind::Interrupted`we close down the loop. Any other error causes a `Panic`.

{% hint style="info" %}
`poll`returns a result back to us. We `match`on the `Result`of `poll` since we want to catch any errors there. The most important part here is that we use the `ErrorKind::Interrupted`as a way to signal to our eventloop that we have sent a `close`signal.

If we don't do this we have no way of shutting the eventloop down \(my first implementation blatantly disregarded this which is a problem if you want to shut your threads down properly\).

I used `io::ErrorKind::Interrupted` to indicate that we have received a signal to close down the loop instead of implementing a type to represent this state. This is to keep our code as short as possible and `Interrupted`is after all somewhat descriptive.
{% endhint %}

The last part of our `Poll`thread is to actually go through the events we got \(if any\) and communicate to the `Executor`that a certain task is ready to make progress.

```rust
for event in &events {
    let event_token = event.id().value();
    evt_sender.send(event_token).expect("send event_token err.");
}
```

The last important part is how we register interest in events to our event queue. To do that we need to leave our `Reactor`and write the main body of our test:

```rust
#[test]
fn proposed_api() {
    let (evt_sender, evt_receiver) = channel();
    let mut reactor = Reactor::new(evt_sender);
    let mut executor = Excutor::new(evt_receiver);

    let mut stream = TcpStream::connect("slowwly.robertomurray.co.uk:80").unwrap();
    let request = b"GET /delay/1000/url/http://www.google.com HTTP/1.1\r\nHost: slowwly.robertomurray.co.uk\r\nConnection: close\r\n\r\n";

    stream.write_all(request).expect("Stream write err.");

    let registrator = reactor.registrator();
    registrator.register(&mut stream, TEST_TOKEN, Interests::READABLE).expect("registration err.");

    executor.suspend(TEST_TOKEN, move || {
        let mut buffer = String::new();
        stream.read_to_string(&mut buffer).unwrap();
        assert!(!buffer.is_empty(), "Got an empty buffer");
        registrator.close_loop().expect("close loop err.");
    });

    executor.block_on_all();
}
```

**The interesting lines here are:**

```rust
let mut stream = TcpStream::connect("slowwly.robertomurray.co.uk:80").unwrap();
let registrator = reactor.registrator();
registrator.register(&mut stream, TEST_TOKEN, Interests::READABLE).expect("...");
registrator.close_loop().expect("close loop err.");
```

First we use our own implementation of a `TcpStream` to open a socket. Next we get a `Registrator`by calling `Reactor::registrator()`. This `registrator` should be able to live in a different thread from our `Poll`instance.

To register interest in an event on the `TcpStream`we call `Registrator::register()`and pass in an exclusive reference to our `TcpStream`, a token which identifies this exact event and a flag indicating what kind of event we're interested in on that socket.

Lastly we close our loop. Our example is a bit contrived in the way that we actually call our `Registrator`inside our first task and close the loop immediately. However, for our test this an easy way to test what we want to accomplish.

{% hint style="info" %}
### **Some tips if you're implementing your own event queue**

First I thought I could use the standard `std::net::TcpStream`here. And that worked well enough when implementing the readiness based models `Epoll and Kqueue`, but once you start implementing `IOCP`you realize that you have a problem!

How can you handle the fact that `Kqueue/Epoll`alerts you when data **is ready to be read** while `IOCP`alerts you when data **has been read**? At some point you either need to abstract over this implementation detail, or you have to let the user handle the fact that they're dealing with two platform dependant implementations...

It was a little bit annoying to figure this out when I thought I had a design that worked for two platforms and already had a working API for them. It required a substantial rewrite to work with `IOCP`, but hey, if you're reading this at least you can learn from my mistake.

My recommendation is to actually start the other way around from what I did, figure out a design that works for `IOCP`and fit the readiness based solutions into working with that API. It's easier than to start with the readiness based ones an try to fit IOCP into that model.
{% endhint %}

## The full code

The code in this test is explained in the Appendix: The Reactor-Executor Pattern. The test has some minor changes in that we take care to join all threads before we exit, and we remove all print statements. In addition, we actually separate the `Reactor`and the `Registrator`and actually send them to different threads in the test, so we take care to cover all our requirements.

```rust
use minimio::{Events, Interests, Poll, Registrator, TcpStream};
use std::{io, io::Read, io::Write, thread, thread::JoinHandle};
use std::sync::mpsc::{channel, Receiver, Sender};

const TEST_TOKEN: usize = 10; // Hard coded for this test only

#[test]
fn proposed_api() {
    let (evt_sender, evt_receiver) = channel();
    let mut reactor = Reactor::new(evt_sender);
    let mut executor = Excutor::new(evt_receiver);

    let mut stream = TcpStream::connect("slowwly.robertomurray.co.uk:80").unwrap();
    let request = b"GET /delay/1000/url/http://www.google.com HTTP/1.1\r\nHost: slowwly.robertomurray.co.uk\r\nConnection: close\r\n\r\n";

    stream.write_all(request).expect("Stream write err.");

    let registrator = reactor.registrator();
    registrator.register(&mut stream, TEST_TOKEN, Interests::READABLE).expect("registration err.");

    executor.suspend(TEST_TOKEN, move || {
        let mut buffer = String::new();
        stream.read_to_string(&mut buffer).unwrap();
        assert!(!buffer.is_empty(), "Got an empty buffer");
        registrator.close_loop().expect("close loop err.");
    });

    executor.block_on_all();
}

struct Reactor {
    handle: Option<JoinHandle<()>>,
    registrator: Option<Registrator>,
}

impl Reactor {
    fn new(evt_sender: Sender<usize>) -> Reactor {
        let mut poll = Poll::new().unwrap();
        let registrator = poll.registrator();

        // Set up the epoll/IOCP event loop in a seperate thread
        let handle = thread::spawn(move || {
            let mut events = Events::with_capacity(1024);
            loop {
                match poll.poll(&mut events, Some(200)) {
                    Ok(..) => (),
                    Err(ref e) if e.kind() == io::ErrorKind::Interrupted => break,
                    Err(e) => panic!("Poll error: {:?}, {}", e.kind(), e),
                };
                for event in &events {
                    let event_token = event.id();
                    evt_sender.send(event_token).expect("send event_token err.");
                }
            }
        });

        Reactor { handle: Some(handle), registrator: Some(registrator) }
    }

    fn registrator(&mut self) -> Registrator {
        self.registrator.take().unwrap()
    }
}

impl Drop for Reactor {
    fn drop(&mut self) {
        let handle = self.handle.take().unwrap();
        handle.join().unwrap();
    }
}

struct Excutor {
    events: Vec<(usize, Box<dyn FnMut()>)>,
    evt_receiver: Receiver<usize>,
}

impl Excutor {
    fn new(evt_receiver: Receiver<usize>) -> Self {
        Excutor { events: vec![], evt_receiver }
    }
    fn suspend(&mut self, id: usize, f: impl FnMut() + 'static) {
        self.events.push((id, Box::new(f)));
    }
    fn resume(&mut self, event: usize) {
        let (_, f) = self.events
            .iter_mut()
            .find(|(e, _)| *e == event)
            .expect("Couldn't find event.");
        f();
    }
    fn block_on_all(&mut self) {
        while let Ok(received_token) = self.evt_receiver.recv() {
            assert_eq!(TEST_TOKEN, received_token, "Non matching tokens.");
            self.resume(received_token);
        }
    }
}
```

