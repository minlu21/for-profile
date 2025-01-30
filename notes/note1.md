---
title: 'Rust Programming Language'
date: 2024-06-27T08:49:17+08:00
draft: true
type: note
tags: ['rust', 'programming language', 'intro']
---

## Introduction

* println! calls a Rust macro. If it called a function instead it would be entered as println.
* Cargo = Rust's build system and package manager
    * Builds code (`cargo build`)
    * Download libraries code depends on (called dependencies)
    * Builds libraries
    * Creating project: `cargo new [proj-name] --bin`
    * `--bin` argument makes executable application (i.e., a binary) rather than a library.
    * Generates a `Cargo.toml` file and `src` directory + git repo.
    * Run code (`cargo run`)
    * `Cargo.lock` is automatically generated to keep track of the versions of dependences in the project. This is never modified. 
    * `cargo check` checks code to see if it compiles but doesn't produce any executable. (much faster than `cargo build`).
    * Compile with optimziations using `cargo build --release`.

## Chap 2/3

* To obtain user input, bring in `std::io` (`println!` is in the prelude)
* `use` statement brings in library into scope.
* `let` is used to create a variable
* Variables are immutable by default, so add keyword `mut` to create a mutable variable. 
* `String::new()`: `::` indicates that the `new` is an associated function of the `String` type. 
    * Associated function is implemented on a type rather than on a particular instance of a `String` (i.e., a static method).
* `std::io::stdin()` returns an instance of `std::io::Stdin()` a type that represents a handle to the standard input for the terminal.
* `&` indicates that a function argument is a reference, which allows you to let multiple parts of you code access one piece of data without needing to copy that data into memory multiple times.
    * Make sure to also make references mutable using `&mut var_name` rather than `&var_name`. 
* Handle potential failure using `Result` type. 
    * `.read_line()` puts what the user types into the string passed in and also returns an `io::Result`. 
    * Rust has a lot of types named `Result` in its std. lib., generic `Result` + specific versions for submoudles like `io::Result`.
    * `Result` types are enums (enumerations).
    * Handle exceptions by using pattern matching with the `match` keyword. `Ok(x) => x` and `Err(_) => handle error`.
* Enums: Types that can have a fixed set of values, called the enum's variants. 
    * For `Result`, the variants are `Ok` or `Err`.
    * For now, use `.expect("[msg]")` to recover when `io::Result` returns `Err` (crashes the program). 
    * For `Ordering`, the variants are `Less`, `Greater`, `Equal` (the three outcomes possible when comparing 2 values). 
* Print values with `println!` placeholders.
    * E.g., `println!("Hello, {}", "Min Lu");`
* External Crates:
    * Install external crates by adding the dependency to `Cargo.toml`
    * Use external dependencies in the Rust program by using `extern crate crate_name;` at the top
    * Import external crate types with `use`
* Match expression:
    * Made up arms = a pattern and the code that should be run if the value given to the beginning of the match expression matches the arm's pattern.
* Shadowing values:
    * Often used in situations in which we want to convert a value from one type to another type.
    * Lets us reuse variable names. 
    * Can change the type of the value stored in the same variable name. 
* Loop:
    * `loop` keyword by itself creates an infinite loop.
    * `while` keyword creates conditional loop.
    * `for` keyword loops through a collection of items. 
* Constants:
    * Declare with `const` instead of `let`
    * Type must be annotated
    * Can be declared in any scope
    * Can only be set to a constant expression, not to the result of a function call or any other value computed at runtime. 
* Statements vs Expressions:
    * Statements: instructions that perform some action and don't return a value.
        * `let` is always a statement, so you can't do something like (`let x = (let y = 6)`), or equivalently in another language is (`x = y = 6`).  
    * Expressions: evaluate to a resulting value
        * Block used to create new scopes `{}` is an expression
        * Expressions do not have ending semicolons
        * Adding semicolons to end of expression turns it into a statement
* Functions:
    * Indicate the return type of a function using an `-> type` after function parameter parentheses.
    * Return value of a function = value of final expression in the body of a function
* Rust does not automatically convert non-boolean types (e.g., integer 0 or 1) to a boolean. 

## Chap 4

* Ownership: Makes memory safety guarantees without needing a garbage collector.
* Rules:
    * Each value in Rust has a variable that's called its owner.
    * There can be only one owner at a time.
    * When the owner goes out of scope, the value will be dropped.
* Move:
    * When reassining heap-allocated variables, instead of coping the entire data on the heap (v. inefficient), Russt copies the stack data without the heap data.
    * Different from shallow copy since Rust also invalidates the old variable's memory. 
* Clone: Create deep copies of heap data.
* References: For example,
    ```rust
    let s1 = String::from("hello");
    let len = calculate_length(&s1);

    fn calculate_length(s: &String) -> usize [
        s.len()
    ]
    ```
    Allow you to refer to some value without taking ownership of it. 
    * Opposite of referencing is dereferencing, using `*`. 
    * References are immutable by default; not allowed to modify something we have a reference to. 
    * To create mutable reference, use `&mut String` instead.
    * However, there can only be one mutable reference to a particular piece of data in a single scope.  
    * Cannot have a mutable reference while we have an immutable one.
    * Multiple immutable references are ok. 
* Slices:
    * let `s : String`, then `&s[0..x]` where `x <= s.size()` is a string slice. 
    * `&str` is an immutable reference and can be returned by a function (unlike `&String`).
    * A `string` literal is the same thing as a `&str`.
    * A `String` can be passed into `&str` as an argument. 
    * Other arrays can also have slices.
    * E.g., `let a = [1,2,3,4,5];` and `let slice = &a[1..3];`. This slice has the type `&[i32]`.

## Chap 5

* Structs vs Tuples:
    * Similarities: Pieces of a struct can be different types (same as tuples).
    * Differences: In a struct you name each piece of data so that its clear what the values mean. 
    * Use the keyword `struct`.
    ```rust
    struct User {
        name: String,
        sign_ins: u64,
        active: bool,
    }
    let user1 = User {
        name: String::from("hello"),
        sign_ins: 0,
        active: true,
    };
    println!("{}", user1.name);
    ```
* Tuple Structs:
    * Don't have names associated with their fields, only types.
    * Useful when wanting to give the whole tuple a name.
    ```rust
    struct Color(i32, i32, i32);
    let black = Color(0, 0, 0);
    ```
* Derived Traits: 
    * 