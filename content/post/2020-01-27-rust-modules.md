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
            // fixed the reference to the constant
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
[`super`](https://doc.rust-lang.org/book/ch07-03-paths-for-referring-to-an-item-in-the-module-tree.html) is how a module
can reference an item in it's parent module.

It's a bit ugly with all the `super`s, but we're not done yet.
Let's move `foo` into it's own file next.
