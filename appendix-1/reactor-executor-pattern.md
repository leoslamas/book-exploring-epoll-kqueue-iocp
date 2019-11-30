# Reactor - Executor Pattern

This pattern is most often referred to as just [Reactor Pattern](https://dzone.com/articles/understanding-reactor-pattern-thread-based-and-eve) is a commonly used technique for "demultiplexing" asynchronous events in the order they arrive. Don't worry I'll explain this in english below.

In Rust we often refer to both a `Reactor`and an `Executor`when we talk about it's asynchronous model, the reason for this is that Rusts `Futures`fits nicely in between as the glue that allows these two pieces to work together.

One huge advantage of this is that this allows us in theory to pick both a `Reactor`and an `Executor`which best suits our problem at hand.

Before we talk more about how this relates to `Futures`lets first have a look at the minimum components this pattern needs:

1. Reactor
   1. An event queue
   2. A way to notify the executor of an event
2. Executor
   1. 

