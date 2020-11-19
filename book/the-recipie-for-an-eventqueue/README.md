# Part 2: A cross platform event queue

So you're either among the 1 % of potential readers actually interested in implementing a toy example of a cross platform event loop, or you couldn't resist checking out part 2 out despite my warnings. Well, don't say I didn't warn you. It's a lot of ground to cover so let's get going!

In this part of the book we'll implement the `minimio`library, an extremely simplified version of a cross platform event queue. I actually encourage you to clone or fork the repository and play around with the code as we go through.

{% hint style="info" %}
[The respository can be found here.](https://github.com/cfsamson/examples-minimio)
{% endhint %}

In this chapter we want to introduce and prepare all the pieces we need to let a user can set up an event queue using our library. Our end goal is something which looks something like this:

```rust
let poll = Poll::new();
let registrator = poll.registrator();

// We can, but don't have to, handle events in a separate thread
let event_queue_thread = thread::spawn(move || {
    // create something to hold our incoming events
    let mut incoming = Events::new();
    loop {
        // blocks(!!!) until an event is ready
        poll.poll(&mut incoming, ...);
        // we returned so we know the OS has placed information about the
        // event(s) that are ready in our queue.
        for event in incoming {
            handle_event(event);
        }
    }
};
// lets us register interest in events in our main thread so we don't have
// to block and can just move on. We don't even need to spend time to check if
// they're ready or not since we'll get notified on our event-queue-thread
registrator.register(...);
...
```

## The parts we need to create

### `Poll`

The `Poll`structure is the heart of our event queue. This structure implements one important method: `poll()`. When called this method tells the OS _"I want you to park the thread I'm in, work on something else, and wake me up when one of the events I told you I'm interested in is ready"._

### `Registrator`

The `Registrator`has one job. It tells our operating system that we're interested in an event. An example of this is when data is ready for us to read on a network socket. The `Registrator` is tied to `Poll`in the sense that we want the OS to wake us up from the call to `poll()`when the event is ready. Every event queue has an ID that is unique. Typically, `Registrator`and `Poll`is on two different threads.

### `Event`

`Poll`can wait for any type of event it supports. Some examples of this is `Read`or `Write`events on a socket or a pipe. These types of events depend on what kind of resource we're working with. Timeouts are another type of event which most systems support.

### `TcpStream` \(or any other resource\)

TcpStream is an I/O resource we want to register an interest in. This needs to be abstracted over if we want to support both `Epoll`,`Kqueue`and `IOCP`. We'll only cover the case of being interested in `Read`events on the `TcpStream`but this is only limited to what kind of resources the OS allows us to register interests on.

### `Token`

A token will be a unique identifier for the event. We need this to actually be able to tell the different events from each other.

This is really all there is to it. The concept is not difficult, but making these components requires a bit of effort since we're dealing with three different implementations. Let's dive and start designing our API.

