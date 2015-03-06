---
layout: post
title: Reading a file in Rust
---

This information will probably be wrong by the time you read it.

Rust is a programming language that aims to be a safe replacement for C. Previous safe-c languages (Cyclone, CCured, others?) were more academic: They did all the background research before developing the language but Rust is being designed and developed at the same time - I think it draws upon some of the previous research though. Not sure. I still write software in C and have the problems with it you'd expect. So I wanted to learn this a bit, hopefully I'll like it and find it useful.

The [Rust Book](https://doc.rust-lang.org/book/) is quite good but it doesn't tell you how to read a file. So here's how..

# Result (error handling)

First of all an important thing involved in all this is the error handling type:

```rust
enum Result<T, E> {
   Ok(T),
   Err(E)
}
```

and a [shorthand version](https://doc.rust-lang.org/std/io/type.Result.html):

```rust
type Result<T> = Result<T, Error>;
```

now Rust has pattern matching, so if you are given a `Result<T>` you can use it by matching on it like this:

```rust
    match foo {
        Ok(r) => println!("ok {}", r),
        Err(e) => println!("no")
    }
```

# Libraries and Imports

Here is how to import all the stuff we need to read a file:

```rust
use std::fs::File;
use std::io::{BufReader, BufReadExt};
```

This is all stuff from the standard library. Now the stdlib will be changing rapidly (whch is why I started by saying this code would probably not work soon) but let's look at how to open a File.

* [std/fs/struct.File](https://doc.rust-lang.org/std/fs/struct.File.html)
* [std/path/trait.AsPath](https://doc.rust-lang.org/std/path/trait.AsPath.html)
* [std/path/struct.Path](https://doc.rust-lang.org/std/path/struct.Path.html)

The type signature is complicated `fn open<P: AsPath + ?Sized>(path: &P) -> Result<File>` but basically it wants an input it calls path which has the AsPath trait, clicking through the docs you'll find that Path has the AsPath trait so we can use that:

```rust
    let fname = "main.rs";
    let path = Path::new(fname);
    let mut file = match File::open(&path) {
        Ok(file) => file,
        Err(e) => {
            println!("Could not open file {}: {}", fname, e);
            return;
        }
    };
```

You have to explicitly match on the resut of File::open rather than using `try!` like the example code shows (the example code doesn't work), I don't know why.

Now that we have a File opened we can use BufRead (In particular one of the extensions to BufRead) to get the lines of the file:

* [std/io/trait.BufRead](https://doc.rust-lang.org/std/io/trait.BufRead.html)
* [std/io/trait.BufReadExt](https://doc.rust-lang.org/std/io/trait.BufReadExt.html)
* [std/io/struct.Lines](https://doc.rust-lang.org/std/io/struct.Lines.html)

So BufReadExt has a function `fn lines(self) -> Lines<Self>`, Lines is a struct whose implementation is hidden:

```rust
pub struct Lines<B> {
    // some fields omitted
}
```

so that takes us to the next language feature we need to learn about...

# Iteration

The useful part about Lines is that it implements the iterator trait:

`impl<B: BufRead> Iterator for Lines<B>`

* [Rust Book, iter](https://doc.rust-lang.org/std/iter/)

This lets use a `for` loop to go through each line. Here is the code in full:

```rust
use std::io::{BufReader, BufReadExt};
use std::fs::File;

fn main() {
    let fname = "main.rs";
    let path = Path::new(fname);
    let mut file = match File::open(&path) {
        Ok(file) => file,
        Err(e) => {
            println!("Could not open file {}: {}", fname, e);
            return;
        }
    };
    
    let mut file = BufReader::new(file);
    for line_iter in file.lines() {
        match line_iter {
            Ok(line) => println!("ok {}", line),
            Err(e) => {
                println!("no");
                return;
            }
        }
    }
}
```

save it as `main.rs` then it can read its code in and print it with 'ok' prefixed before each line.

I'm compiling it like this: `LD_LIBRARY_PATH=/usr/local/lib:~/rust/dist/lib/ ~/Code/rust/dist/bin/rustc -C prefer-dynamic main.rs` and the result is a 112kB binary, it's 966 kB if you don't prefer-dynamic (since it statically links in all the rust runtime stuff - I'm not sure what that is yet).

