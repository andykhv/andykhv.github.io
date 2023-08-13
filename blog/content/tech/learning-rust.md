---
title: "On Learning Rust"
date: 2023-08-11T14:13:21-07:00
draft: false
---
# On Learning Rust 
After completing [Rustling's exercises](https://github.com/rust-lang/rustlings), I wanted to write a bit about the interesting concepts I learned and what Rust has to offer!

Rust is a unique programming language. It doesn't have a runtime or garbage collector, but the compiler doesn't require us to manually manage memory either!

__i.e. allocating dynamic memory in C:__
```C
char *string = (char *) malloc(sizeof(char) * 32); //allocate 32 bytes of memory to the heap
free(str) //deallocate memory from the heap
```

In C, memory is managed manually through the stack and heap. Local variables with known sizes at compile time are allocated to the stack. Values with dynamic sizes not known at compile time are allocated to the heap. There is extra bookkeeping for managing values in the heap compared to the stack. Accessing values through the stack involves moving the stack pointer -- this is fast!

How is memory managed in Rust if we don't need to manually allocate memory to the heap? Well, memory is managed by its compiler. One of Rust's features is compile-time memory safety. It knows when to re-use registers and manage the stack and heap. This is done by enforcing the concepts of __ownership__ and __borrowing__ during compilation of a rust program.

## Ownership and Borrowing
There are a few rules for [Ownership](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html):
1. Each value has an owner
2. There can only be one owner at a time
3. When the owner goes out of scope, the value will be dropped.

Let's take a look at an example rust program:

```rust
fn main() {
    let mut foo = String::from("hello"); //foo is initialized and owned
    foo.push('!');

    appendExclamation(foo); //foo's ownership is moved into appendExclamation's scope

    println!("{}", foo); //the compiler complains about this line 
}

fn appendExclamation(bar: String) { //appendExclamation has ownership of bar
    println!("{}", bar);
}
```

An error here is calling `println!("{}", foo)` after passing `foo` into `appendExclamation()`. This is an instance where the compiler enforces the rules of Ownership. `foo`'s value is dropped in main's scope due to passing its value into `appendExclamation()`, because it is now in the scope of `appendExclamation()`. Therefore, `appendExclamation()` is now the owner of `foo`. `foo` in `main` is now dropped, and the value is freed.


### Borrowing
What about passing `foo` as a reference into `appendExclamation()` as `&foo`?

```rust
fn main() {
    let mut foo = String::from("hello"); //foo is initialized and owned
    foo.push('!');

    appendExclamation(&foo); //foo's ownership is moved into appendExclamation's scope

    println!("{}", foo); //the compiler complains about this line 
}

fn appendExclamation(bar: &String) { //appendExclamation has ownership of bar
    println!("{}", bar);
}
```

This program compiles and outputs:
```rust
hello!
hello!
```

In this program, `appendExclamation` is borrowing the value of `foo` from `main`. `main` still maintains ownership of `foo`. In Rust context, passing values by reference is considered __borrowing__. Another caveat is that a borrowed value cannot be modified. If we append another character to `foo` inside the scope of `appendExclamation`, the rust compiler will output an error.

The idea of __mutable borrows__ come into play with the `mut` keyword when modifying values by reference. `mut` is a keyword that enables a value to be modified, because values without `mut` are inherently immutable in Rust. Without the `mut` modifier, we would not be able to append `'!'` to `foo`. We can do the same thing for references, but the compiler only allows exactly one `&mut Type` value at a time. Therefore there cannot be more than one borrower that mutates the given value. Obviously, there can always be more than one `&Type` value, because the referenced values are essentially __read-only__. 

Here is an example of passing a mutable reference to `appendExclamation`:
```rust
fn main() {
    let mut foo = String::from("hello"); //foo is initialized and owned
    foo.push('!');

    appendExclamation(&mut foo); //foo's ownership is moved into appendExclamation's scope

    println!("{}", foo); //the compiler complains about this line 
}

fn appendExclamation(bar: &mut String) { //appendExclamation has ownership of bar
    bar.push('!')
    println!("{}", bar);
}
```

This program outputs:
```rust
hello!!
hello!!

```

Rust's standard library does contain types that follow the idea of _Resource Acquistion is Initialization_, in which ownership is encapsulated in an object. Such types are: [Box](https://doc.rust-lang.org/std/boxed/struct.Box.html), [Vec](https://doc.rust-lang.org/std/vec/struct.Vec.html), [Rc](https://doc.rust-lang.org/std/rc/struct.Rc.html), [Arc](https://doc.rust-lang.org/std/sync/struct.Arc.html). 


## Thoughts
This article is definitely not an in-depth look into Rust and its internals, but I wanted to convey a unique characteristic that Rust embodies.
Ownership and Borrowing are a set of rules that enable a lower footprint in CPU and memory -- while also being memory-safe like Java. There is definitely a steeper learning curve for Rust compared to other languages like Python. I don't recommend Rust as an initial programming language to learn. In the era of cloud computing, distributed systems and generative AI -- I think Rust will be a rewarding language. For me, I see Rust as a medium to learn about networks, systems programming and web servers.


Thanks for reading!
