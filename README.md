# Introduction

If you've read about asynchonous programming and how webservers and other high performance asynchronous programs work, you most likely have read that many of them rely on Epoll, Kqueue and IOCP to handle I/O. This book aims to be a short and conciese introduction to all three of them and a practical way to learn how they work and how you can use them.

{% hint style="info" %}
This book is developed in the open and has two repositories:

1. The repository for this Gitbook
2. The repository for the code example
{% endhint %}

### Who is this book for?

This book should be interesting if you want to learn more about:

* How FFI works in Rust and what the crates `libc` and `mio`provides for you
* How to create a cross platform eventloop using the same methods as `Node` and `Tokio`uses 
* How to create a library in Rust that conditionally compiles code for the three major platforms
* How to make raw syscalls on Linux, Macos and Windows
* How `mio` works

The code for this book lives in this repository: [examples-minimio](https://github.com/cfsamson/examples-minimio)

The name `minimio` was chosen because of the similarity with the full fledged library in rust called [mio](https://github.com/tokio-rs/mio).

### What we'll do

We'll create an extremely simple cross platform library which leverages Epoll on Linux, Kqueue on BSD/Macos and IOCP for Windows to read incoming data from a socket. Our main focus is to introduce the different syscalls and the infrastructure we need to create a workigin event queue.

{% hint style="info" %}
We'll avoid any external dependencies so we make sure we remove as much "magic" as possible to make sure we really understand the tools we use.
{% endhint %}

**The book will be divided into three parts:**

1. Creating a library and designing an API
2. Getting our hands dirty with Epoll, Kqueue and IOCP 
3. Creating cross platform abstractions

After we're done you should have a pretty good understanding of how Epoll. Kqueue and IOCP works,

### Motivation

This is a companion book to my previous book: [The Node Experiment - Explorign Async Basics with Rust](https://cfsamson.github.io/book-exploring-async-basics/). Since both `libuv` and `mio` uses a cross platform eventloop using epoll, kqueue and IOCP under the hood I needed to cover this subject in some detail as well.

## Status

This book is still under development, but will be released in the near future.

