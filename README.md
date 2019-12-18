---
description: Cross Platform Event Queues Explained With Rust
---

# The Boring Book About Epoll, Kqueue and IOCP

Yep. After spending a couple of hundred hours on this I've concluded that this is actually pretty boring stuff. It's fun to know, but definately boring to learn. 

However, it's very usefull knowledge which can be pretty hard to come by, and it's a big deal if you know this well. I do really hope this is a good introduction which can either serve as a reference or as an inspiration to dig deeper.

Even though this might be a boring book, I do promise one thing. If you get through it all the way to the end, writing your own code along with it and making sure you understand everything, you'll find it extremely fun afterwards. 

Of course, my goal is to make this trip we're about to venture on as fun as possible. 

{% hint style="info" %}
**This book is developed in the open and there are some resources I'll point you at right away:**

\*\*\*\*[**The repository for this Gitbook:**](https://github.com/cfsamson/book-exploring-epoll-kqueue-iocp)  
****The repository contains the text of this book. Use the [Issue Tracker ](https://github.com/cfsamson/book-exploring-epoll-kqueue-iocp/issues)if you have questions or feedback of any kind. If you spot any mistaks, want to suggest changes or otherwise contribute to this book please make a [Pull Request](https://github.com/cfsamson/book-exploring-epoll-kqueue-iocp/pulls). Feedback and/or contributions are greatly appreciated!

[**The example code for this book - the minimio library:**](https://github.com/cfsamson/examples-minimio)  
I suggest that you clone or fork the repository and play with the example yourslf. Of course, all the code will be presented in this book as well.

If you spot any mistakes in the code or changes you want to suggest, use the issue tracker and create pull requests as you would in any other project.   
  
_The name `minimio` was chosen because of the similarity with the full fledged library in rust called_ [_mio_](https://github.com/tokio-rs/mio)_._
{% endhint %}

### Who is this book for?

This book should be interesting if you want to learn more about:

* How FFI works in Rust and what the crates `libc` and `mio`provides for you
* How to create a cross platform eventloop using the same methods as `Node` and `Tokio`uses 
* How to create a library in Rust that conditionally compiles code for the three major platforms
* How to make raw syscalls on Linux, Macos and Windows
* How `mio` works

_We'll avoid any external dependencies so we make sure we remove as much "magic" as possible to make sure we really understand the tools we use._

### What we'll do

We'll create an extremely simple cross platform library which leverages Epoll on Linux, Kqueue on BSD/Macos and IOCP for Windows to read incoming data from a socket. 

We'll introduce the different syscalls and the infrastructure we need to create a working event queue and  look at the considerations we need to make to create a cross platform event queue by providing a common API for all three implementations.

{% hint style="info" %}
If you've read about asynchonous programming and how webservers and other high performance asynchronous programs work, you most likely have read that many of them rely on Epoll, Kqueue and IOCP to handle I/O. This book aims to be a short and concise introduction to all three of them and a practical way to learn how they work and how you can use them.
{% endhint %}

### Disclaimer

This will be an introduction to this subject. The example we write will have a strong focus on create a working but minimal example. There are many edge cases we don't cover, but I try to mention the most important ones in the appendix chapter:[ It's not that easy.](appendix-1/its-not-that-easy.md) Please read that chapter too. 

If there are important omissions I have made which are not mentioned there, consider making creating an [issue ](https://github.com/cfsamson/book-exploring-epoll-kqueue-iocp/issues)or a [pull request](https://github.com/cfsamson/book-exploring-epoll-kqueue-iocp/pulls) to add it.

### Prerequisites

This subject is admittedly pretty difficult \(at leas I think so\). If both Rust and concurrent programming is something you're new to I do suggest that you check out my previous books first. After that you should be more than prepared for this book:

[Green Threads Explained in 200 Lines of Rust](https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/)

[The Node Experiment - Exploring Async Basics with Rust](https://cfsamson.github.io/book-exploring-async-basics/)

### Motivation

This is a companion book to my previous book: [The Node Experiment - Explorign Async Basics with Rust](https://cfsamson.github.io/book-exploring-async-basics/). Since both `libuv` and `mio` uses a cross platform eventloop using epoll, kqueue and IOCP under the hood I needed to cover this subject in some detail as well. 

It turned out that getting a deeper understanding of this subject was pretty hard with information scattered around and mostly covering one aspect of either Epoll, Kqueue or IOCP. Therefore, I chose to spend some extra time and effort to collect this information and present it here to make it easier for the next person that ventures on the same quest as I did.

## Status

This book is still under development, but will be released in the near future.

