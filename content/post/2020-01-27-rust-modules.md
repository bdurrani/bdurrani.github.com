---
title: "Rust module system"
date: 2020-01-27
---

I've been writing small programs in rust as a way to learn the language. While trying to organize the application, I realized I didn't fully understand how the module system worked.

A lot of the code out there says that if you have your code organized in a single rs file, you can just split up the module into different files and the same rules apply. I think I must have read the following document (https://doc.rust-lang.org/book/ch07-01-packages-and-crates.html) 3-4 times before I fully started to absorb the material. That, and asking a tom of questions on the Rust Discord channel, which was really helpful.

Let's say you have an amazing application, and you have all the code organized into modules, but everything is in one file. I assume you already have read the basics of using modules to control scope and privacy (https://doc.rust-lang.org/book/ch07-02-defining-modules-to-control-scope-and-privacy.html)

```
mod foo {
    pub fn test_foo() {
        println!("test_foo");
        bar::test_bar();
    }

    mod bar {
        pub fn test_bar() {
            println!("bar {}", super::super::constants::CONSTANT_A);
		  test_bar_internal();
        }

        fn test_bar_internal() {
            println!("test_bar_internal");
        }
    }
}

mod constants {
    pub const CONSTANT_A: u8 = 5;
}

mod one {
    pub fn test1() {}
}

fn main() {
    println!("Hello, world! {}", constants::CONSTANT_A);
    one::test1();
    foo::test_foo();
}
```

In general, a very useless little application. But it will demonstrate some of the things I learned along the way.

[`super`](https://doc.rust-lang.org/book/ch07-03-paths-for-referring-to-an-item-in-the-module-tree.html) is how a module
can reference an item in it's parent module.

If we were to look at how the modules are organized, they would look
like this

```
            main
        /   |       \
     foo constants  one
      /
   bar
```

You have 3 modules (foo, constants and one) with a single parent main.
We call main our **crate root**.
We will discuss this more later on.

Let's start splitting up the modules. First the `constants` module.
I'll move the code in the constants module into a file in the same level as `main.rs`

Constants.rs

```
// Move the contents to a new file, without the mod block
pub const CONSTANT_A: u8 = 5;
```

```Main.rs

// declare the existance of the constants module
mod constants;

mod foo {
    pub fn test_foo() {
        println!("test_foo");
        bar::test_bar();
    }

    mod bar {
        pub fn test_bar() {
            println!("bar {}", super::super::constants::CONSTANT_A);
            test_bar_internal();
        }

        fn test_bar_internal() {
            println!("test_bar_internal");
        }
    }
}

mod one {
    pub fn test1() {}
}

fn main() {
    println!("Hello, world! {}", constants::CONSTANT_A);
    one::test1();
    foo::test_foo();
}
```

I added comments to where I made the changes.

Notice however, we have maintained our module tree.
Even though the `constants` mod is in it's own file, it still has
`main.rs` as its root.

Let's move `foo` into it's own file next.

```foo.rs
use crate::constants;

pub fn test_foo() {
    println!("test_foo");
    bar::test_bar();
}

mod bar {

    pub fn test_bar() {
        println!("bar {}", super::constants::CONSTANT_A);
        test_bar_internal();
    }

    fn test_bar_internal() {
        println!("test_bar_internal");
    }
}
```

Here is what `main.rs` looks like

```
mod constants;
mod foo;

mod one {
    pub fn test1() {}
}

fn main() {
    println!("Hello, world! {}", constants::CONSTANT_A);
    one::test1();
    foo::test_foo();
}
```

Notice we added `mod foo` to `main.rs`.
We did the same when moving `constants` to its own file.
By adding it to `main.rs`, I'm declaring that my crate still tree looks
like the following:

```
           root
        /   |       \
     foo constants  one
      /
   bar
```

I renamed `main` to `root`.
What is root?
In the case of my binary crate, my [crate root](https://doc.rust-lang.org/book/ch07-01-packages-and-crates.html) is `main.rs`.
If you have a library crate, the crate root would be `lib.rs`.
These are special files that must be included in any rust project.

For example, if you were to rename `main.rs` to anything else, `cargo` will complain

```
$ cargo build
error: failed to parse manifest at `~\module-test\Cargo.toml`

Caused by:
  no targets specified in the manifest
  either src/lib.rs, src/main.rs, a [lib] section, or [[bin]] section must be present
```

There must be a way to change the default target, but that is beyond the scope of this
discussion.

Why did we add `mod foo` to `main.rs`?
Because that is how our program was initially organized.

Why did I add `use crate::constants`?
[`use crate`](https://doc.rust-lang.org/book/ch07-03-paths-for-referring-to-an-item-in-the-module-tree.html) is an absolute path.
If you want to use a function, we need to know its path.
By saying `use crate::constants`, we are saying the `constants` module is the direct child of
the crate root.
Instead of using `use crate::constants`, we could have also used `use super::constants` because `constants` is the direct child of the crate root, like `foo`.
Sibling modules in different files cannot refer to each other directly.
They must reference each other via a common root, which in our case is `main`, which also happens
to be the crate root.
We didn't have to do this before because all the modules were in the same file.

We still have one more step to do.
The module `bar` is still in the same file as `foo`.
Let's move `bar` to be it's own file.

Note: [Rust 2018 removed the requirement of always needing a `mod.rs` file](https://doc.rust-lang.org/edition-guide/rust-2018/module-system/path-clarity.html#no-more-modrs).
So I wont use `mod.rs` for now.

`bar` is a submodule of `foo`. So let's set that up by creating a folder called `foo` and moving `bar` into that folder.
