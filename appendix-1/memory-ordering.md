# Memory ordering

In our library we have the following code to avoid registering interest in new events to an event queue which has recieved a close signal:

```rust
pub fn register(
    &self,
    soc: &mut TcpStream,
    token: usize,
    interests: Interests,
) -> io::Result<()> {
    if self.is_poll_dead.load(Ordering::SeqCst) {
        return Err(io::Error::new(
            io::ErrorKind::Interrupted,
            "Poll instance is dead.",
        ));
    }
    
    // WHAT HAPPENS IF ANOTHER THREAD CLOSES THE LOOP AT THIS EXACT MOMENT?
    
    ffi::create_io_completion_port(soc.as_raw_socket(), self.completion_port, 0)?;

    let op = ffi::Operation::new(token);
    soc.operations.push_back(op);

    if interests.is_readable() {
        ffi::wsa_recv(
            soc.as_raw_socket(),
            &mut soc.wsabuf,
            soc.operations.back_mut().unwrap(),
        )?;
    } else {
        unimplemented!();
    }

    Ok(())
}
```

When writing code for multiple threads there are several subtle things we need to consider. In the case of this exact code, what happens if we were to have several registrators on different threads and one is closing the loop at the same time another is registering an interest?

Well, while we do check that the `Poll`instance is not dead before we register an event there is nothing in our code which prevents 

There are two reasons we use Atomics and these exact memory orderings.

1. Compiler reordering
2. CPU caching and reordering

Let's start with a very naive and unsafe way of doing this and have a look at the different options we have and why we use them:

```rust
static mut LOCKED: bool = false;

pub fn test(mut b: i32) -> Option<i32> {
    if unsafe { !LOCKED } {
        unsafe { LOCKED = true };
        let a = b + 1;
        unsafe { LOCKED = false };
        Some(a)
    } else {
        None
    }
}

fn main() {
    let a = 1 + test(2).unwrap();
}
```

When we compile with `cargo run --debug`we get this assembly output \(abbreviated\)

```text
.LBB262_4:
	movb	playground::LOCKED(%rip), %al
	xorb	$-1, %al
	testb	$1, %al
	jne	.LBB262_6
	movq	32(%rsp), %rax
	addq	playground::COUNTER(%rip), %rax
	setb	%cl
	testb	$1, %cl
	movq	%rax, 16(%rsp)
	jne	.LBB262_7
	jmp	.LBB262_8

.LBB262_6:
	movb	$1, playground::LOCKED(%rip)
	jmp	.LBB262_4
```

Now this looks fine. We check the flag and do exactly what we would expect given our code. Now, let's look at the assembly output with `opt-level=3`.

```text
example::test:
        lea     edx, [rdi + 1] # first parameter + 1
        mov     eax, 1         # set the
        ret
```

Oh, ok, so that was short. 

