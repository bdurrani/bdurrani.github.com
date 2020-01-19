---
title: "A bit of rust"
date: 2020-01-19
---

While killing some time online, I can into a [blog](https://chrisdown.name/2020/01/13/1195725856-and-friends-the-origins-of-mysterious-numbers.html) post that explained
the origins of some weird numbers by looking at the hex data and converting that
to strings.

This gave me something to try in rust. Specifically, taking the bit of python
code that converted strings into unsigned 32-bit and 16-bit integers and porting
that to rust.

I started by converting a string to bytes.

```
let raw_bytes_slice = command.as_bytes();
let raw_bytes_vector = raw_bytes_slice.to_ve
println!("raw_bytes {:?}", raw_bytes);
```

Rust strings are UTF-8. I had a set of well-defined strings
so there would be no encoding issues.

The next step was to convert the byte array into a `u32` or `u16`. Rust
already has functions for this.

The part that took some getting used to was the slices and the arrays.

[as_bytes()](https://doc.rust-lang.org/std/primitive.str.html#method.as_bytes) returns a
byte slice `&[u8]`.
[from_be_bytes](https://doc.rust-lang.org/std/primitive.u32.html#method.from_be_bytes)
expects a 4-byte array as an input `[u8;4]`.

This was my first approach, pre-allocate the correct type
of array and copy the slice into it.

```
let mut array: [u8; 4] = [0x20; 4];
let raw_bytes = trunc.as_bytes();
array.copy_from_slice(&raw_bytes[..4]);
```

A slightly more dangerous (more C-like) way was to disable the
soundness checks and just point to the slice.

```
let raw_bytes = string_data.as_bytes();
let ptr = raw_bytes.as_ptr() as *const [u8; 4];
let array: [u8; 4] = unsafe { *ptr };
```

I settled on the following approach I got from the Rust discord channel.

```
let num = u32::from_be_bytes(str_data.as_bytes()[..4].try_into().unwrap());
```

I guess I should have paid more attention to the documentation since this code snippet is there for the documentation for from_le_bytes().

From there it was pretty straight forward.
It's just a matter of iterating over the different sizes of integers and the different http verbs.

```
use std::mem::size_of;
use std::convert::TryInto;

fn main() {
    let column_width = 10;

    println!(
        "{0: <4} | {1: <4$} | {2: <4$} | {3: <4$}",
        "size", "verb", "big-endian", "little-endian", column_width
    );
    for size in vec![size_of::<u16>(), size_of::<u32>()] {
        for item in vec![
            "GET", "HEAD", "POST", "PUT", "DELETE", "OPTIONS", "TRACE", "PATCH",
        ] {
            let mut trunc = String::from(item);
            // pad strings with spaces
            trunc = format!("{1: <0$}", 4, trunc);
            trunc.truncate(size);
            let be_string: String;
            let le_string: String;

            if size == 4 {
                let array:[u8;4] = trunc.as_bytes()[..4].try_into().unwrap();
                be_string = u32::from_be_bytes(array).to_string();
                le_string = u32::from_le_bytes(array).to_string();
            } else if size == 2 {
                let array:[u8;2] = trunc.as_bytes()[..2].try_into().unwrap();
                be_string = u16::from_be_bytes(array).to_string();
                le_string = u16::from_le_bytes(array).to_string();
            } else {
                unimplemented!();
            }

            println!(
                "{0: <4} | {1: <4$} | {2: <4$} | {3: <4$}",
                size,
                item,
                be_string,
                le_string,
                column_width
            );
        }
    }
}
```

I could probably clean this up further by switching out the `if` with a `match`.

The only other problem I had was a weird compiler error about borrowing that
was actually about me missing an `else` case. The [unimplemented](https://doc.rust-lang.org/std/macro.unimplemented.html) handled that case.

I still have a lot to learn but I'm really enjoying everything about
the language so far.
