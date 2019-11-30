# Reactor - Executor Pattern

This pattern is most often referred to as just [Reactor Pattern](https://dzone.com/articles/understanding-reactor-pattern-thread-based-and-eve) is a commonly used technique for "demultiplexing" asynchronous events in the order they arrive. Don't worry I'll explain this in english below.

In Rust we often refer to both a `Reactor`and an `Executor`when we talk about it's asynchronous model, the reason for this is that Rusts `Futures`fits nicely in between as the glue that allows these two pieces to work together.

One huge advantage of this is that this allows us in theory to pick both a `Reactor`and an `Executor`which best suits our problem at hand.

Before we talk more about how this relates to `Futures`lets first have a look at the minimum components this we need.

1. **Reactor**
   1. An event queue
   2. A way to notify the executor of an event
2. **Executor**
   1. Scheduler
   2. Handlers
3. **Task**
   1. User code
   2. Needs to register interest in an event with the Reactor
   3. Needs to be interruptable so it can be suspended and yield control to the Executor instead of waiting for I/O.

### The Reactor

Our cross platform Epoll/Kqueue/IOCP library can be viewed as the main building block of the reactor - the event queue part. The only missing piece is a way for us to communicate with the `Executor`that an event is ready and we actually need to recieve the events which are ready and wake up the task which will finish. The simplest way to do this is to use a `Channel`which is exactly what we'll do.

A very simple `Reactor`can look like this:

```rust
struct Reactor {
    handle: std::thread::JoinHandle<()>,
    registrator: Registrator,
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
                    Err(ref e) if e.kind() == io::ErrorKind::Interrupted => {
                        println!("INTERRUPTED: {}", e);
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

        Reactor { handle, registrator }
    }

    fn register_stream_read_interest(&self, stream: &mut TcpStream, token: usize) {
        self.registrator.register(stream, token, Interests::readable()).expect("registration err.");
    }

    fn stop_loop(&self) {
        self.registrator.close_loop().expect("close loop err.");
    }
}
```

{% hint style="info" %}
### How does this relate async in Rust?

In Rust a library like [mio](https://github.com/tokio-rs/mio) will be what drives the `Reactor` part. In Rust we have `Futures`which pass on a `Waker`to the `Reactor`. Instead of communicating directly with the executor through a channel, the Reactor wil call `Waker.wake()`on the relevant `Waker`once an event is ready.
{% endhint %}

### Executor

The executor needs to schedule the execution of events which are ready and provide a way for us to set handlers for events. In our example we'll specifically set a handler function which will resume our task when the corresponding event is ready.

We run the handlers for each task as they arrive, and we run them synchronously. This is what we mean by "demultiplexing" asynchronous events.

A simple executor can look like this:

```rust
struct Excutor {
    events: Vec<(usize, Box<dyn FnMut()>)>,
    evt_reciever: Receiver<usize>,
}

impl Excutor {
    fn new(evt_reciever: Receiver<usize>) -> Self {
        Excutor { events: vec![], evt_reciever }
    }
    fn suspend(&mut self, id: usize, f: impl FnMut() + 'static) {
        self.events.push((id, Box::new(f)));
    }
    fn resume(&mut self, event: usize) {
        println!("RESUMING EVENT: {}", event);
        let (_, f) = self.events
            .iter_mut()
            .find(|(e, _)| *e == event)
            .expect("Couldn't find event.");
        f();
    }
    fn block_on_all(&mut self) {
        while let Ok(recieved_token) = self.evt_reciever.recv() {
            assert_eq!(TEST_TOKEN, recieved_token, "Non matching tokens.");
            println!("EVENT: {} is ready", recieved_token);
            self.resume(recieved_token);
        }
    }
}
```

It's not very sophisticated but will do the work for us. As you see a handler, which will actually represent a suspended task in our example, doesn't need to be anyting more fancy than a closure which we'll invoke when the time is right. The means of communication between our `Reactor`and `Executor`is just a regular `std::sync::mspc::Channel`and all we do in `block_on_all`is to wait for events which we can respond to.

There is of course many ways which we could choose to handle this, feel free to play around and try for yourself.

{% hint style="info" %}
### How does this relate to async in Rust?

In rust libraries like `Tokio`or `async_std`takes on the role as Executors. Generally a `Reactor`and an `Executor`will be provided together in a runtime so you won't have to actually call different methods on the `Executor`and `Reactor`like we do here and instead leave that to the runtime you choose. 

However, there is nothing in Rusts standard library or language which prevents you to choose an `Reactor`and an `Executor`based on what suits your needs best.
{% endhint %}

### Task

To actually use our `Reactor`and `Executor`we need to provide some code which glues everything together. For our simple example, it will look like this:

```rust
fn main() {
    let (evt_sender, evt_reciever) = channel();
    let reactor = Reactor::new(evt_sender);
    let mut executor = Excutor::new(evt_reciever);
    
    // ===== TASK =====
    let mut stream = TcpStream::connect("slowwly.robertomurray.co.uk:80").unwrap();
    let request = b"GET /delay/1000/url/http://www.google.com HTTP/1.1\r\nHost: slowwly.robertomurray.co.uk\r\nConnection: close\r\n\r\n";

    stream.write_all(request).expect("Stream write err.");
    reactor.register_stream_read_interest(&mut stream, TEST_TOKEN);
    
    // ===== SUSPEND TASK =====
    executor.suspend(TEST_TOKEN, move || {
        let mut buffer = String::new();
        stream.read_to_string(&mut buffer).unwrap();
        assert!(!buffer.is_empty(), "Got an empty buffer");
        reactor.stop_loop();
    });
    
    // ===== TASK END =====

    executor.block_on_all();
    // NB! Best practice is to make sure to join our child thread. We skip it here for brevity.
    println!("EXITING");
}
```

There are a few things to note here. First of all the `TcpStream`is provided to use by our `Reactor`and is not the one in the standard library. Sencondly you'll see that we have only implemented methods for `stream_read`and not for `Write`. If we had to write all our async code like this it wouldn't be very ergonomic. Thats why we use runtimes which does a lot of this for us, like closing down our `Poll`loop, releasing resources and joining threads.

**Lets go through the steps we take here:**

1. We provide a means of communication between our `Reactor`and `Executor`
2. We instantiate a `Reactor`and an `Executor`
3. We open a socket and write a request \(to a slow endpoint wich will wait 1000ms before responding\)
4. We register a `Read`intrerest on that socket with our `Reactor`
5. We register a handle function with our `Executor`to be run once data is ready
6. We stop the `Poll`loop manually in our handle \(we only have one task\)
7. We run our `Executor`and block until all tasks are finished

{% hint style="info" %}
### How does this releate to async in Rust?

We have a task here which we manually stop in the middle and then resume once data is ready for us. The task is "Get data from the socket and assert that we actually recieved some data".

To accomplish this we use a callback based approach. In Rust, we'd normally use `Futures`to do this. We'll cover this in more detail in a later book, but it's important to remember that a `Future`is just a different way of creating an interruptable task. 

In addition, Rusts `Futures`provide a `Waker`which should be used to let the `Executor`know that a task can be woken up and resumed. In our example we have a tight coupling between the executor and reactor through the use of a regular `Channel`as a way to communicate. With `Futures`the executor and reactor can be totally decoupled.

As a final note. Here we run part of our task on the main thread. Normally, a runtime will let you just `spawn`your task an take care of suspending and resuming it for you.
{% endhint %}

