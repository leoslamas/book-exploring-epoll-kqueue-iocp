# Designing our API

The first thing we need to do is to create a library project to start coding. We'll name our project `minimio.`

Run the following commands to create our project:

```text
mkdir minimio
cd minimio
cargo init --lib
```

### Using integration tests to drive the API design

The first think I like to do when I know roughly what API  I want to use is to start a new project by defining roughly the API I want to use. 

For now we only want to cover one use case and that is to provide a event queue we can use in our [examples-node-eventloop](https://github.com/cfsamson/examples-node-eventloop). Now, whether you've read my prevoius book or not shouldn't matter much, since we'll create an integration test that defines what we want to accomplish anyway.

I've looked into how [mio ](https://github.com/tokio-rs/mio)API as an inspiration but for convenience I've made some compromizes to simplify our code to fit an example.

{% hint style="info" %}
Reacently, `mio` changed from using IOCP as the backing API on Windows and started using WePoll instead simplifying their implementation greatly. I'll cover WePoll shortly in the end of this book, but for now, just know that I've used [mio v0.6.x](https://github.com/tokio-rs/mio/tree/v0.6.x) version as the inspiration for the code we use in this book for this exact reason so if you go to dive into the source code, make sure to switch to the `v0.6.x`branch.
{% endhint %}

### Creating an integration test

In Rust, it's common to have unit tests in the same file as code. Integration tests usually goes in a sperate `tests`folder, so let's make one. Our folder structure should look like this once done:

```text
minimio
   |
   |--> src
   |     |--> lib.rs
   |
   |--> tests
   |      |--> api.rs
```

Now open `api.rs`and let's start desiging how we want our API to work.

**We have some requirements:**

1. We want to be able to block the current thread while we wait for events
2. We want the same API across operating systems
3. We want to be able to register interest from a different thread than we run the main loop on

Ok, so before we dive into the code, let's consider what we can immideately derive from these requirements.

### Blocking the current thread

We need an interface we can use which provides a way for us to block the current thread while waiting for events to be ready. We also need something that identifies an event so we know which one finished in what order. We need to be able to send a signal to stop waiting in \(1\) and exit from the blocking call or else our program would never exit \(if the event queue is run on our main thread\) or end i a "bad" way if we run it in a child thread.

* We'll call the structure providing a blocking method `Poll`
* We'll call the thing which identifies an event a `Token` \(which will simply be a `usize`in this case\)
* We'll need a structure representing an `Event`

### Same API for all platforms

This is a difficult one. And it's very difficult to decide on in advance before actually digging into the details of both Epoll, Kqueue and IOCP. We know that IOCP requires us to provide a buffer which is filled with data but Epoll and Kqueue let's us know when data is ready to read. At least we can derive from that tha we need to hook into whatever method the user calls for retrieving data. It seems impossible for us to provide any abstration lower than that.

Once we realize that we also realize that using `std::net::TcpStream`is out of the question, but we can implement our own `TcpStream`where we "hijack" the `Read`methods \(and `Write`methods if we implemented those as well\). This way we can use `std::net::TcpStream`under the hood but implement our own `Read`implementation.

* We need to provide a `TcpStream`
* We need something to represent what kind of event we're interested in. We'll call that `Interests`

### Register interest from a different thread

So we want the user to be able to register interest from one thread and block while waiting for events on another thread. That means we need some way of having a basic synchonization to know that our `Poll`instance is actually waiting for events and that we have created an event queue.

It's apparent that this structure will be closely tied to our event queue. There are many ways to solve this but let's just start with a name and an idea of how it should work.

* We need a `Registrator`which knows if our event queue is alive and can be sent to another thread.

### The integration test

So we have a rough idea about our API now so let's start scetching out a test. We'll divide this into smaller unit tests later but test against this to know when we've reached our goal for now.

Lets start with the imports we know we need from our library:

```rust
use minimio::{Events, Interests, Poll, Registrator, TcpStream};
```

```rust
#[test]
fn proposed_api() {
    let mut poll = Poll::new().unwrap();
    let registrator = poll.registrator();
```

The first thing we do is to create a test function and use the `#[test]`attribute so Cargo will notice this as a test we want to run.

Inpsired by `mio` we call our main eventqueue instance `Poll`. We know that setting up a `Poll`can fail since we're going to have to ask the OS to actually create a queue for us.

The next important part is the `registrator`. Now this is what actually allows us to register events to the event queue.

The `Poll` instance and the `registrator`is what we really want to test. These two together provides all we really need to handle our event queue. The `Poll`instance handles waiting for events and returning information on an event we're waiting for. The `registrator`let's us register interest in new events. The rest is really "plumbing" we need to test this in a realistic scenario.

{% hint style="info" %}
We're assuming that the normal use case here is that we keep the registrator on one thread and send `Poll`of to a different thread where it will wait for events to occur. Of curse this need not be the case but it's important for us that we consider this use case in pur design from the start.
{% endhint %}

The next thing we do is to create a channel to send events from our `eventqueue`thread to our main thread which will act on the event that occurred. We also need a runtime to store a list of events we are waiting for and the logic we want to run when the event is ready.

```rust
let (evt_sender, evt_reciever) = channel();
let mut rt = Runtime { events: vec![] };
let test_token = 10;
```

Don't worry, we'll have a look at what `Runtime`is in the end. The `test_token`variable is a token we'll use to identify our event. We'll use a usize as a `Token`throughout this book.

The next step is to set up a simple event queue, and we do this in a different thread so we can block there while we wait for our main thread to register an event to the queue.

```rust
let handle = thread::spawn(move || {
    let mut events = Events::with_capacity(1024);
    loop {
        match poll.poll(&mut events, Some(200)) {
            Ok(..) => (),
            Err(ref e) if e.kind() == io::ErrorKind::Interrupted => {
                break;
            }
            Err(e) => panic!("Poll error: {:?}, {}", e.kind(), e),
        };
        for event in &events {
            let event_token = event.id().value();

            evt_sender.send(event_token).expect("send event_token err.");
        }
    }
});
```

After we've spawned a new thread we want that thread to first create a list of "empty" event structs we'll send off to the OS and get back populated with information about any events that have happened.

Next we enter into a `loop`which will block on `poll.poll(&mut events, Some(200))` until an event has happened. We pass in the zeroed list of events and set a timeout of `200`milliseconds.

We get an result back. It's important to note that the events are now placed in the `events`list and not returned. We `match`on the `Result`of `poll` since we want to catch any errors there. The most important part here is that we use the `ErrorKind::Interrupted`as a way to signal to our eventloop that we have sent a `close`signal. 

If we don't do this we have no way of shutting the eventloop down \(my first implementation blatantly disregarded this which is a problem if you want to shut your threds down properly\).

Next, we go through the list of events that has happened and get the `token`. We use the channel we set up earlier to signal to our main thread that event with id x is ready.

Now we need to do something we can wait on. We create a web `GET`request to a very slow server and register interest in getting a notice when a "read" event is ready for us.

```rust
let mut stream = TcpStream::connect("slowwly.robertomurray.co.uk:80").unwrap();
let request = "GET /delay/1000/url/http://www.google.com HTTP/1.1\r\n\
               Host: slowwly.robertomurray.co.uk\r\n\
               Connection: close\r\n\
               \r\n";
stream
    .write_all(request.as_bytes())
    .expect("Error writing to stream");

registrator
    .register(&mut stream, test_token, Interests::readable())
    .expect("registration err.");
```

There is one important thing to note here. First I thought I could use the standard `std::net::TcpStream`here. And that worked well enough when implementing the readiness based models `Epoll and Kqueue`, but once you start implementing `IOCP`you realize that you have a problem!

How can you handle the fact that `Kqueue/Epoll`alerts you when data **is ready to be read** while `IOCP`alerts you when data **has been read**? At some point you either need to abstract over this implementation detail, or you have to let the user handle the fact that they're dealing with two platform dependant implementations...

The answer is that we can't have the user care about this. We need to abstract over something so they don't have to worry about this. 

{% hint style="info" %}
Yes, it was a little bit annoying to figure this out when I thought I had a design that worked for two platforms. It required a substantial rewrite to work with `IOCP`, but hey, if you're reading this at least you can learn from my mistake.

My recommendation is to actually start the other way around, figure out a design that works for `IOCP`and fit the readiness based solutions into working with that API. It's easier than to start with the readiness based ones an try to fit IOCP into that model.
{% endhint %}

We choose to abstract over the difference between the readiness based models and the completion based model by providing our own `TcpStream`struct which you need to use with this library. This is the same as for example `mio`does so at least we're in good company.

We use our `registrator`to register an interest in `read`events on this `TcpStream`. In addition we choos an API where we explicitly pass in an **unique** token to identify this event. Note that this places a burden on the user of this API to ensure the uniqueness of this token. We could do that for them, but we chose not to do that in our library.

The last step is to actually wait in our main thread for the event to happen.

```rust
rt.spawn(test_token, move || {
    let mut buffer = String::new();
    stream.read_to_string(&mut buffer).unwrap();
    assert!(!buffer.is_empty(), "Got an empty buffer");
});

while let Ok(recieved_token) = evt_reciever.recv() {
    assert_eq!(test_token, recieved_token, "Non matching tokens.");
    rt.run(recieved_token); 
    
    // let's close the event loop since we know we only have 1 event
    registrator.close_loop().expect("close loop err.");
}

handle.join().expect("error joining thread");
```

We do this by first registering a callback which we will run once we've gotten a notice that our event is ready. We simply move the `TcpStream`over to this closure and read from it and asserts that we actually got some data. We also give the callback the same unique id as the `token`we registered with the event queue.

The next step is to listen on our channel for incoming messages. Our `eventqueue`thread will send a token to us to identify the event that occurred to us when it's ready.

We assert that the token we recieved is the same as our `test_token`which had a value of `10`. Next we run the callback we registered with that id.

We then register a `close`event using `registrator.close_loop().expect("close loop err.");`. This will cause our `eventqueue`thread to close and we'll get an `Err`value on our channel, which will cause us to break out of the loop.

Finally we join the thread as a good practice making sure that all destructors are run before we exit our process.

Now, this is actually the blueprint of our design. The next chapters will cover how to actually getting this to work.

### Bonus section

While not required, I'll also quickly explain how we set up our very simple runtime:

```rust
struct Runtime {
    events: Vec<(usize, Box<dyn FnMut()>)>,
}

impl Runtime {
    fn spawn(&mut self, id: usize, f: impl FnMut() + 'static) {
        self.events.push((id, Box::new(f)));
    }

    fn run(&mut self, event: usize) {
        let (_, f) = self
            .events
            .iter_mut()
            .find(|(e, _)| *e == event)
            .expect("Couldn't find event.");
        f();
    }
}
```

As you see, this is a very simple runtime. The backing structure is a `Vec<usize, Box<dyn FnMut()>)>`which is an array with a tuple of `usize`which is the Id of the callback and a Boxed closure which is the actual callback we register.

The `Runtime`has two methods. `spawn`takes an unique id and a closure. We simply store both the id and the closure so we can find the exact closure and run it later.

The last function is `run`which takes an id as argument which identifies which callback to run. We use `iter_mut`and `find`to mutably iterate over the callbacks and find the one with an id equal to the `id`we passed in. 

Since `find`returns an `Option`we unwrap it here so we either `panic`or get the value. Since we are iterating over tuples of `(usize, Box<dyn FnMut()>)`we dicard the `usize`value and assign the closure we found to `f`using `let (_, f)`and finally we invoke the callback we had stored by calling `f();`.

