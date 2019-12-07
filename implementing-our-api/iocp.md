# IOCP



```rust
  // We require a `&mut TcpStram` here, but we only need it for Windows. Now
    // there are ways to deal with this, but either way we need to in reality
    // mutate our `TcpSocket` on Windows the way we have designed this. If we
    // implement this as part of a Runtime which we control there are ways for
    // us to guarantee that the buffer is not touched or moved, so we could
    // mutate it without the user knowing safely, but this is not a Runtime so
    // we can't know that.
    pub fn register_with_id(
        &self,
        stream: &mut TcpStream,
        interests: Interests,
        token: usize,
    ) -> io::Result<Token> {
        self.registry
            .selector
            .registrator(self.is_poll_dead.clone())
            .register(stream, token, interests)?;
        Ok(Token::new(token))
    }
```

