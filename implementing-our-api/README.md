# Implementing our API

The first thing we do is to describe and implement the platform-indipendant functionality and the common surface of our library. 

Navigate to `lib.rs`and open the file.

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

Let's just start by naming the structs and methods we need to implement the API we designed in the last chapter just to get a starting point. The 

===== TOKEN====

```rust
pub type Events = Vec< ... >;

pub struct Poll { ... }

pub struct Registry { ... }

impl Poll {
    pub fn new() -> io::Result<Poll> { ... }

    pub fn registrator(&self) -> Registrator { ... }

    pub fn poll(&mut self, events: &mut Events, timeout_ms: Option<i32>) -> io::Result<usize> { ... }
}

const WRITABLE: u8 = 0b0000_0001;
const READABLE: u8 = 0b0000_0010;

pub struct Interests(u8);
impl Interests {
    pub const READABLE: Interests = Interests(READABLE);
    pub const WRITABLE: Interests = Interests(WRITABLE);
}
impl Interests {
    pub fn is_readable(&self) -> bool { ... }

    pub fn is_writable(&self) -> bool { ... }
}

```

