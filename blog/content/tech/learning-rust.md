---
title: "On Learning Rust"
date: 2023-08-11T14:13:21-07:00
draft: true
---
# On Learning Rust 
After completing [Rustling's exercises](https://github.com/rust-lang/rustlings), I wanted to write a bit about the interesting concepts of writing Rust code and what it has to offer!

Rust is a unique programming language. It doesn't have a runtime or garbage collector, but the compiler doesn't require us to manually manage memory either!

__i.e. allocating dynamic memory in C:__
```C
char *string = (char *) malloc(sizeof(char) * 32); //allocate 32 bytes of memory to the heap
free(str) //deallocate memory from the heap
```

How is memory managed then? Well, memory is managed by its compiler. It knows when to re-use registers and manage the stack and heap. This is done by enforcing the concepts of __ownership__ and __borrowing__ during compilation of a rust program.

## Ownership and Borrowing
There are a few rules for [Ownership](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html):
- Each value has an owner
- There can only be one owner at a time
- When the owner goes out of scope, the value will be dropped.

Let's take a look at an example rust program:

```rust
fn main() {
    let foo = String::from("hello"); //foo is initialized and owned
    println!("{}", foo);

    appendExclamation(foo); //foo's ownership is moved into appendExclamation's scope

    println!("{}", foo); //because foo's ownership is moved, this is an invalid statement 
}

fn appendExclamation(mut bar: String) { //appendExclamation has ownership of bar
    bar.push('!');
    println!("{}", bar);
}
```

Well, if you try compiling this program with `rustc`, you will get some errors. 


## Iterators