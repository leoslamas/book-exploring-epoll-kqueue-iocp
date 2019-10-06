# Exploring Epoll, Kqueue and IOCP with Rust

This is the repository for the gitbook where we explore the basics of how
epoll (on Linux), kequeue (on Macos) and IOCP (on Windows) work.

We do this by implementing a very simple and extremely limited cross platform event loop.

## Motivation

This is a companion book to my previous [Exploring Async Basics by Implementing a Node Event Loop in Rust]
and since both `libuv` and `mio` uses a cross platform eventloop using epoll, kqueue and IOCP
under the hood I needed to cover this subject in some detail as well.

## Status

This book is still under development, but will be released in the near future.