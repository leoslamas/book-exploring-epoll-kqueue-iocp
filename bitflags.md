# Bitflags

Bitflags are commonly used as a conventient way of providing users of an api a set of options they can set. We call this setting a `flag`.

### Bitflags by example

Let's say you have 8 different choises a user can can set. For our example let's say it's a set of toppings on a pizza:

1. Mozzarella
2. Peccorino
3. Gorgonzola
4. Pepperoni
5. Tomato
6. Anchovies
7. Parma ham
8. Basil

We can set any combination of these flags, even none. A very efficient way of representing these choises is using an `u8`. There are 2^8 different combinations \(256 combinations possible using these 8 toppings\).

**We could represent this using a `u8`in two ways:**

1. Assiging each possible combination a unique value from 0-255.
2. Using the bits in the `u8`as a flag to indicate if a topping is choosing or not

While both are possible, a chef might want to first know what kind of cheese the customer wants. Now in the first case it would be 8 possible \(unique\) numbers that would signify that for example Mozzarella is chosen.

In the second alternative the chef needs only to check if the first bit is 1 or 0. 

The problem assigning a unique number for each possible combination is exponentially cumbersome when increasing the number of choises. Thats why using bitflags are popular.

Here are some examples using bitflags:

| Toppings | Bitflag |
| :--- | :--- |
| Mozzarella, Pepperoni, Tomato | 00011001 |
| Mozzarella, Tomato, Basil | 10010001 |
| All possible toppings | 11111111 |

### Bitflags in Rust

If we wanted to represent this in code we could do as follows:

```rust
const MOZZARELLA: u8 = 0x01;
const PECCORINO: u8 = 0x02;
const GORGONZOLA: u8 = 0x04;
const PEPPERONI: u8 = 0x08;
const TOMATO: u8 = 0x10;
const ANCHOVIES: u8 = 0x20;
const PARMA_HAM: u8 = 0x40;
const BASIL: u8 = 0x80;

fn main() {
    let pizza1: u8 = MOZZARELLA | PEPPERONI | TOMATO;
    let pizza2: u8 = MOZZARELLA | TOMATO | BASIL;
    let pizza3: u8 = MOZZARELLA | PECCORINO | GORGONZOLA | PEPPERONI | TOMATO | ANCHOVIES | PARMA_HAM | BASIL;
    
    println!("pizza1: {:08b}", pizza1);
    println!("pizza2: {:08b}", pizza2);
    println!("pizza3: {:08b}", pizza3);
    
    println!("Does pizza1 contain anchovies: {}", has_anchovies(pizza1));
    println!("Does pizza2 contain anchovies: {}", has_anchovies(pizza2));
    println!("Does pizza3 contain anchovies: {}", has_anchovies(pizza3));
    
    
}

fn has_anchovies(choises: u8) -> bool {
    choises & ANCHOVIES == ANCHOVIES
}
```

{% embed url="https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=e097d16e123cc5c47ada0e9f8831cb57" caption="You can try the code on the Rust playground" %}

There many efficient and convenient ways of working with such flags using OR, AND, XOR etc to check whether a flag, a combination of flags is set.

