---
title: "Rust module system"
date: 2020-01-27
---

I've been writing small programs in rust as a way to learn the language.
While trying to organize one of the application, I realized I didn't fully understand how the module system worked.

The official docs say that if you have your code organized in a single rs file, you can just split up the module into different files and the same rules apply.
Without understanding how the module system actually works, you might be mislead into making
false assumptions and hence run into errors that you might not understand how to solve.
I think I must have read the [documentation](https://doc.rust-lang.org/book/ch07-01-packages-and-crates.html) 3-4 times before I realize that just skimming over the the documentation wasn't going to work.
 
So I went back to basics and broke down the problem.
That, and asking a ton of questions on the Rust Discord channel, which was really helpful.
I figured I would write this up in case this helps out anyone else.

I assume you already have read the basics of using modules to control scope and privacy.
The [documentation](https://doc.rust-lang.org/book/ch07-02-defining-modules-to-control-scope-and-privacy.html) explains that very well.

Let's say you have an amazing application, and you have all the code broken down into modules, but everything is in one file.

### main.rs 

{{< highlight rust >}}
mod foo {
    pub fn test_foo() {
        println!("test_foo");
        // call a function from bar
        bar::test_bar();
    }

    mod bar {
        pub fn test_bar() {
          // call a constant from the constants module
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
    // call a function from module one
    one::test1();
    // call a function from module foo
    foo::test_foo();
}
{{< / highlight >}}

In general, a very useless little application.
But it will demonstrate some of the things I learned along the way.

[`super`](https://doc.rust-lang.org/book/ch07-03-paths-for-referring-to-an-item-in-the-module-tree.html) is how a module
can reference an item in its parent module.
So `super::super::` is a way of specifying a module that is the grand-parent of the module.

If we were to look at how the modules are organized, they would look
like this

```
            main
        /    |      \
     foo constants  one
      /
   bar
```

You have 3 modules (`foo`, `constants` and `one`) with a single parent main.
We will call `main` our **crate root** module.
We will discuss this more later on.

Let's start splitting up the modules. First the `constants` module.
I'll move the code in the constants module into its own file `constants.rs`

### constants.rs

{{< highlight rust >}}
// Move the contents to a new file, without the mod block
pub const CONSTANT_A: u8 = 5;
{{< / highlight >}}

### main.rs

{{< highlight rust >}}
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
{{< / highlight >}}

I added comments to where I made the changes.

Notice however, we have maintained our module hierarchy.
Even though the `constants` mod is in its own file, it still has
`main.rs` as its root.

Let's move `foo` into its own file next.

### foo.rs
{{< highlight rust >}}
// bring the constants module into scope
use crate::constants;

pub fn test_foo() {
    println!("test_foo");
    bar::test_bar();
}

mod bar {

    pub fn test_bar() {
        // since constants is in the scope of the module foo
        // we can refer to constants module by referring to it
        // via the parent of the bar module.
        println!("bar {}", super::constants::CONSTANT_A);
        test_bar_internal();
    }

    fn test_bar_internal() {
        println!("test_bar_internal");
    }
}
{{< / highlight >}}

### main.rs

{{< highlight rust >}}
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
{{< / highlight >}}

Notice we added `mod foo` to `main.rs`.
We did the same when moving `constants` to its own file.
By adding it to `main.rs`, I'm declaring that my crate's module hierarchy
is still the same as before. I.E., the module `foo` is a child module of `main`
```
            main
        /    |      \
     foo constants  one
      /
   bar
```

In the case of my binary crate, my [crate root](https://doc.rust-lang.org/book/ch07-01-packages-and-crates.html) module is `main`.
If you have a library crate, the crate root would be `lib.rs`.
These are files that must be included in any rust project.

For example, if you were to rename `main.rs` to `blah.rs`, `cargo` will complain when
I try to build the project:

{{< highlight bash >}}
$ cargo build
error: failed to parse manifest at `~/module-test/Cargo.toml`

Caused by:
  no targets specified in the manifest
  either src/lib.rs, src/main.rs, a [lib] section, or [[bin]] section must be present
{{< / highlight >}}

There are ways to change the defaults, but that is beyond the scope of this
discussion.

Why did we add `mod foo` to `main.rs`?
We did that to maintain the module hierarchy that was set up when we started refactoring.

Why did I add `use crate::constants`?
[`use crate`](https://doc.rust-lang.org/book/ch07-03-paths-for-referring-to-an-item-in-the-module-tree.html) is an absolute path.
If you want to use a function from a module, we need to know its path in the module hierarchy.
By saying `use crate::constants`, we are bring the module `constants`, which happens to be a direct child of the crate root, in to the scope of module `foo`.

Instead of using `use crate::constants`, we could have also used `use super::constants` because `constants` is the direct child of the crate root, like `foo`.
Sibling modules in different files cannot refer to each other directly.
They must reference each other via a common root, which in our case is `main`, which also happens
to be the crate root.

We have one more step left.
The module `bar` is still in the same file as `foo`.
Let's move `bar` to be its own file.

Note: [Rust 2018 removed the requirement of always needing a `mod.rs` file](https://doc.rust-lang.org/edition-guide/rust-2018/module-system/path-clarity.html#no-more-modrs).
So I wont use `mod.rs` for now.

`bar` is a submodule of `foo`. So let's set that up by creating a folder called `foo` and moving `bar` into that folder.
### foo/bar.rs

{{< highlight rust >}}
use crate::constants;

pub fn test_bar() {
    println!("bar {}", constants::CONSTANT_A);
    test_bar_internal();
}

fn test_bar_internal() {
    println!("test_bar_internal");
}
{{< / highlight >}}

### foo.rs

{{<  highlight rust >}}
// declare bar to be a submodule of foo
mod bar;

pub fn test_foo() {
    println!("test_foo");
    bar::test_bar();
}

{{<  / highlight >}}

No changes were required to `main.rs` since it never referenced `bar`.

What happened here?
Since `bar` is a submodule of `foo`, we want to maintain that relationship.
We do that by [creating a folder with the same name as `foo`](https://doc.rust-lang.org/reference/items/modules.html#module-source-filenames) and moving `bar` there.

We started with a single file `main.rs`.
Our application source folder now looks like this:

```
│   constants.rs
│   main.rs
│   foo.rs
└───foo
        bar.rs
```

Another way we could have split up `foo` and `bar` was by using `mod.rs`.
Instead of having `foo.rs`, create `mod.rs` under the `foo` folder, and move the
contents of `foo` into `mod.rs`.

### foo/mod.rs
{{< highlight rust >}}
// this is exactly the same as what used to be foo.rs
mod bar;

pub fn test_foo() {
    println!("test_foo");
    bar::test_bar();
}

{{< / highlight >}}

No other changes were required. This is the new folder structure.

```
│   constants.rs
│   main.rs
└───foo
        bar.rs
        mod.rs
```

Notice the lack of `foo.rs`. 
This configuration and the one prior are identical.
This approach is how the module would have been organized prior to Rust 2018.

We are done organizing each module into its own file.
Now we can focus on each module on its own, add unit and integration tests
in each module without having to worry about breaking other pieces.

Some key pieces to remember

- Every module needs a root. This is why we needed to add `mod foo` and `mod constants` to `main.rs`.
- There can only be one root. You can't add `mod foo` to multiple files.
- Understand what your module hierarchy looks like and use `self`, `super` and `crate` accordinly.
- When in doubt, just type out the examples in the docs.
- Really read the [docs](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html). Then read them again.
