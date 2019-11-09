# Creating a Library in Rust

The first thing we need to do is to create a library project to start coding. We'll name our project `minimio.`

Run the following commands to create our project:

```text
mkdir minimio
cd minimio
cargo init --lib
```

### Designing our API

The first think I like to do when I know roughly what API  I want to use is to start a new project by defining roughly the API I want to use. 

For now we only want to cover one use case and that is to provide a event queue we can use in our [examples-node-eventloop](https://github.com/cfsamson/examples-node-eventloop). Now, whether you've read my prevoius book or not shouldn't matter much, since we'll create an integration test that defines what we want to accomplish anyway.

As a big inspiration in implementing the API I've looked into how [mio ](https://github.com/tokio-rs/mio)does it but for convenience I've made some compromizes to simplify our code a little bit.

{% hint style="info" %}
Reacently, `mio` changed from using IOCP as the backing API on Windows and started using WePoll instead simplifying their implementation greatly. I'll cover WePoll shortly in the end of this book, but for now, just know that I've used [mio v0.6.x](https://github.com/tokio-rs/mio/tree/v0.6.x) version as the inspiration for the code we use in this book for this exact reason so if you go to dive into the source code, make sure to switch to the `v0.6.x`branch.
{% endhint %}

### Creating an integration test

In Rust, it's common to have unit tests in the same file as code. Integration tests usually goes in a sperate `tests`folder, so let's make one. Our folder structure should look like this once done:

```text
minimio -
        |
        |--> src
        |     |
        |     |--> lib.rs
        |
        |--> tests
        |     |
        |     |--> api.rs
```

Now open `api.rs`and let's start desiging how we want our API to work:

```rust
#[test]
fn proposed_api() {
    let mut poll = Poll::new().unwrap();
    let registrator = poll.registrator();
```

The first thing we do is to create a test function and use the `#[test]`attribute so Cargo will notice this as a test we want to run.

Inpsired by `mio` we call our main eventqueue instance `Poll`. We know that setting up a `Poll`can fail since we're going to have to ask the OS to actually create a queue for us.

The next important part is the `registrator`. Now this is what actually allows us to register events to the event queue.

{% hint style="info" %}
We're assuming that the normal use case here is that we keep the registrator on one thread and send `Poll`of to a different thread where it will wait for events to occur. Of curse this need not be the case but it's important for us that we consider this use case in pur design from the start.let \(evt\_sender, evt\_reciever\) = channel\(\);
{% endhint %}

The next thing we do is to create a channel to send events from our `eventqueue`thread to our main thread which will act on the event that occurred. We also need a runtime to store a list of events we are waiting for and the logic we want to run when the event is ready.

```rust
let (evt_sender, evt_reciever) = channel();
let mut rt = Runtime { events: vec![] };
```

Don't worry, we'll have a look at what `Runtime`is in the end.



