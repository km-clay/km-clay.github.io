---
layout: post
title: Using Rust to Write a Unix Shell
date: 2026-06-09
---

  I wrote `shed` in Rust as a sort of educational on-boarding project to the language, and low level Unix programming in general. As far as I can tell, the only other Unix shell that currently exists that was written from the ground up in Rust is `nushell`. `fish` had its rust rewrite, but the codebase still has a great deal of baggage from when it was still a C++ codebase. If you want to see for yourself what I mean by that, just clone the repo and run `grep -R WString`.

  Writing it in Rust has made some things genuinely very pleasant. The `Drop` trait makes managing context trivial. It has proven to be an exceptionally powerful primitive for setup and teardown of execution context. For instance:
  * Setting shell variables before execution, unsetting them after
  * Changing terminal settings before execution, resetting them after
  * Saving the state of the line editor before an edit, comparing the state after, pushing the difference to the undo stack

  Another thing is that Rust's error propagation has allowed for very interesting behavior patterns. For instance, in `shed`'s codebase, the `return`, `break`, and `continue` keywords are actually a part of the `ShErr` enum, which is `shed`'s general `Error` type. When you execute those keywords, `shed` creates an error and that error begins propagating upward. Certain contexts wait to catch those error types via pattern matching, e.g. `exec_func()` waits for `ShErrKind::Return` and stops the propagation if that error type is caught. Otherwise, the error propagates all the way up, and if it actually does reach the top level, that *is an error*, so the behavior is correct there too.

  Rust's enums also make it possible to create very expressive return types. For instance, the combination of `Option` and `Result` allow for 3 different possible expressions for the methods found in the parser struct:
  * `Ok(Some(_))` - "The parse succeeded, here's what I found"
  * `Ok(None)` - "The parse succeeded, but I didn't find a match"
  * `Err(_)` - "The parse encountered an error, here's what happened"
  This, combined with pattern matching, makes for exceptionally clean handling.

  `shed`'s codebase also makes gratuitous use of Rust's macro system. Simply deriving the `ShOptGroup` trait instantly provides all of this for free:
  * getter functions
  * setter functions
  * a query function that plugs directly into the main `ShOpt` struct's query method
  * a function that returns a source-able set of shell commands to construct the group's default state, used in the default `shedrc` file.
  That procedural macro alone probably saves more than 2000 lines of code, and makes the `shopt` system in general extremely easy to maintain. On top of this, both the POSIX compliance test suite and the line editor's test suite both use macros that very cleanly generate entire `#[test]` functions from a single line.

  As for what has been made difficult by the decision to use Rust, it's a usual suspect of programming in Rust: global state management. There are many ways to approach this problem, and I have tackled it in many different ways throughout the project's lifespan. Originally, state was tracked by a single struct that exists at the top level, and is threaded through the function signature of *nearly every function in the codebase* as a mutable borrow. This does have the nice effect of having the compiler enforce borrowing rules, but I found it to be far too restrictive.

  The other approach was to store the shell's state struct in a globally available variable. This of course runs into the common Rust problem of needing to figure out what combination of synchronization primitives you should wrap it in for Rust to allow you to do this. Luckily `shed` is almost entirely single threaded, and any dispatched threads never alter state directly, so we can store it in a thread local variable. That simplifies things considerably, but ultimately we still need to store the value in a `RefCell` so that we can at least get some protection against potential UB.

  Accessing this global through the `RefCell` has the same rules that the borrow checker enforces, only it enforces them at runtime instead of compile time. If a `RefCell` is borrowed mutably more than once, or borrowed immutably while a mutable borrow exists, the program panics. This of course has many implications. While we are accessing the state struct for any reason, we cannot access it again for any reason until that operation is complete. So if we are reading a variable, we cannot check how many jobs there are, etc.

  This approach ended up being more trouble than it was worth as well. Eventually what I settled on was this: instead of storing the state struct in a `RefCell`, I stored each of its fields in its own separate `RefCell`, then created individual accessor functions for each of the state struct's fields. The end result was a very clean interface. For instance, reading a variable looks like this: `Shed::vars(|v| v.get_var("VAR_NAME"))`, which was eventually condensed to just `var!("VAR_NAME")` via `macro_rules`.

  This design has trade-offs of its own, however. The codebase is now split into two parts: a part that can call the state accessor methods, and a part that can be interacted with using the accessor methods. The reason for this split is re-entrancy. Because we are using `RefCell` for the state struct fields, if we do something like calling `Shed::vars_mut()` inside of a function given to `Shed::vars_mut()`, the `RefCell` will be borrowed mutably twice and the program will panic.

  Given the problems of the other approaches, this is trade-off is the one I've decided is the most acceptable.
