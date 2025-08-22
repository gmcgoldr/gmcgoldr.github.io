---
layout: default
title: null
author: null
---

# Ergonomic errors in Rust: write fast, debug with ease, handle precisely

Errors show up in three distinct contexts: when you’re writing code, when you’re debugging code, and at runtime when the program needs to handle recoverable errors. And errors are consumed by two distinct consumers  with different needs: the developer debugging an application, and the caller making error handling decisions at runtime.

In this post, we'll explore how [stackerror](https://crates.io/crates/stackerror) is designed to make working in all three contexts easy while providing rich debugging context for developers and structured codes for runtime error-handling. 

In Rust, different tools tend to optimize for some but not all three of these contexts: bare `Result<T, &'static str>` is quick to type but thin on context; `anyhow` speeds propagation and allows you to easily add debug-friendly context, yet pushes you toward downcasts when you need to handle errors at runtime; `thiserror` gives you matchable variants for robust runtime handling but requires additional upfront work when writing errors.

These tools can also conflate the two consumers of errors: error variants often pack data for both debugging and for error handling. But deciding how to handle a recoverable error usually doesn’t require fully structured payloads, just an error code. This pattern of exposing an error code for error handling is used widely across Rust’s ecosystem: `std::io::Error` exposes an `ErrorKind`, `serde_json::Error` has a `Category`, and HTTP clients surface `StatusCode`.

Using [stackerror](https://crates.io/crates/stackerror) you can attach human-readable messages right where failures occur with minimal boilerplate; your errors stack to provide rich context for debugging; you get structured error codes for control flow (HTTP status codes, `std::io::ErrorKind`, or your own); and your library can expose a single opaque error type that implements `std::error::Error`.

To make things concrete, this post builds up a small working covering IO, HTTP handling, and simple retries. Stick around (or scroll down) to the end to see it.

## Rough edges

Rust has several mature error libraries, each with clear strengths and a few rough edges depending on your needs. Here’s a quick tour of a common error handling journey and the pain points you might experience along the way.

### Starting simple with static strings

It’s fast to write and easy to read. But your debug info is only as good as your static strings, and it’s hard to propagate detail. The typical pattern erases context:

```rust
fn read_config() -> Result<String, &'static str> {
    std::fs::read_to_string("config.json")
        .map_err(|_| "read failed")
}
```

When this fails, you’ve lost all context about why it failed: was it a permission error, is the file missing, etc.?

### Leveling up: anyhow

`anyhow` makes propagation trivial and lets you add dynamic context where failures occur:

```rust
use anyhow::{Context, Result};

fn read_config() -> Result<String> {
    std::fs::read_to_string("config.json")
        .context("read failed")
}
```

This is a step up for writing and debugging. The tradeoff shows up when you need to handle specific errors at runtime or when designing public APIs:

- For recoverable cases (e.g., HTTP 429, file already exists), you often need to downcast.
- Exposing `anyhow::Result<T>` in a library API couples callers to `anyhow` conventions instead of a standard `std::error::Error` type you own.

Downcasting requires you to know the concrete source error type and where it’s wrapped; the compiler can’t help you, and if internals change your branches silently stop matching. Here's a short example of handling “file not found” by downcasting:

```rust
match read_config() {
    Ok(text) => println!("{}", text),
    Err(err) => {
        if let Some(io_err) = err.downcast_ref::<std::io::Error>() {
            if io_err.kind() == std::io::ErrorKind::NotFound {
                // File not found: print nothing
            } else {
                // Other IO error: propagate
                return Err(err);
            }
        } else {
            // Not an IO error: propagate
            return Err(err);
        }
    }
}
```

### Precision with enums: thiserror (and friends)

Libraries like `thiserror` give you strong, matchable variants which are great for runtime handling and stable APIs:

```rust
#[derive(thiserror::Error, Debug)]
pub enum ReadConfigError {
    #[error("read failed")]
    IOError(#[from] std::io::Error)
}

fn read_config() -> Result<String, ReadConfigError> {
    std::fs::read_to_string("config.json")
        .map_err(ReadConfigError::from)
}
```

```rust
match read_config() {
    Ok(text) => println!("{}", text),
    Err(ReadConfigError::IOError(io_err)) if io_err.kind() == std::io::ErrorKind::NotFound => {},
    Err(err) => return Err(err),
}
```

A well-designed enum lets you `match` precisely. The challenge is in coming up with a good design. Poorly designed enums either become too generic or too granular: if you opt for "one big enum", you risk reusing broad variants (e.g., everything becomes a generic network error variant). If you opt for per-callsite enums, every fallible call gets its own variant and writing code turns into an exercise in writing error enums. Additionally, small implementation changes can force public API changes because variants are coupled to internals.

### Write fast, debug with ease, and handle precisely: stackerror

`stackerror` gives you the best of both worlds: write code as quickly as with `anyhow`, but get precise runtime matching like `thiserror`. It uses error codes for matching and stacks context as you propagate errors up the call chain.

```rust
use stackerror::prelude::*;

fn read_config() -> StackResult<String> {
    std::fs::read_to_string("config.json")
        .map_err(StackError::from)
        .stack_err_msg("read failed")
}
```

```rust
match read_config() {
    Ok(text) => println!("{}", text),
    Err(err) if err.err_code() == Some(&ErrorCode::IoNotFound) => {},
    Err(err) => Err(err),
}
```

We’ll spend the rest of this article diving deeper into the three contexts: writing, debugging, and runtime handling. And we’ll see more `stackerror` features along the way.

## Writing (and reading) code

When you’re *writing* code, the fastest path is to keep error handling in the same place as the fallible call. That isn’t just nicer to read, it avoids the “stop coding, go add an enum variant elsewhere, come back” loop that error-enum designs often force.

With `stackerror`, you wrap upstream errors as you go and return a `StackResult<T>`; the helper methods are implemented on `StackError` as well as `Result<_, StackError>` and are no-ops on `Ok`, so the happy path stays untouched.

Below is a minimal program split into small functions: download JSON, parse the first element, write it to a file. Notice how each function just wraps the immediate error (or creates one for an `Option`) and returns. No enums, no boilerplate. You can shorten it even more since the `StackError::from` calls can be replaced with the `?` operator, but laying it out this way makes things more explicit.

```rust
use stackerror::prelude::*;

pub async fn download_json(url: &str) -> StackResult<String> {
    reqwest::get(url)
        .await
        .map_err(StackError::from)?
        .text()
        .await
        .map_err(StackError::from)
}

pub fn parse_first(body: &str) -> StackResult<String> {
    serde_json::from_str::<Vec<String>>(body)
        .map_err(StackError::from_msg)?
        .first()
        .ok_or_else(StackError::new)
        .cloned()
}

pub fn write_output(path: &str, value: &str) -> StackResult<()> {
    std::fs::write(path, value)
        .map_err(StackError::from)
}

#[tokio::main]
async fn main() -> StackResult<()> {
    let body = download_json("https://example.invalid/list.json").await?;
    let first = parse_first(&body)?;
    write_output("first.txt", &first)?;
    Ok(())
}
```

That’s all we need for propagation. You can use `from_msg` to wrap any `std::error::Error` and nearly anything else that might be used as an error. `stackerror` also has specific `From` implementations for `std::io::Error`, and optionally for `reqwest::Error` (behind a feature flag). Using `StackError::from` to wrap these errors has the advantage of populating `StackError`'s error code from the underlying IO error kind or HTTP status code.

By contrast, adding an enum variant every time you add fallible code adds a lot of ceremony to the writing process.

## Debugging code

When something fails, you don’t just want to know *what* went wrong, but also *how* it went wrong. A backtrace gives you this information, but it can be cumbersome to work with. Using `stackerror`, you build your own breadcrumb right next to each fallible call; those messages stack in execution order. The result is a readable output that points you to the right line with little additional noise.

Here is an updated version of the same program from the previous section, now with small co-located messages. Notice how nothing else changes: you still return `StackResult<T>` and use `?`; you just pin a short note to each failure site. `fmt_loc` is a convenience macro provided by `stackerror` that you can optionally use to prepend your error messages with the file name and line number.

```rust
use stackerror::prelude::*;

pub async fn download_json(url: &str) -> StackResult<String> {
    reqwest::get(url)
        .await
        .map_err(StackError::from)
        .stack_err_msg(fmt_loc!("can't get {url}"))?
        .text()
        .await
        .map_err(StackError::from)
        .stack_err_msg(fmt_loc!("response text is invalid"))
}

pub fn parse_first(body: &str) -> StackResult<String> {
    serde_json::from_str::<Vec<String>>(body)
        .map_err(StackError::from_msg)
        .stack_err_msg(fmt_loc!("response isn't a valid JSON array"))?
        .first()
        .ok_or_else(StackError::new)
        .with_err_msg(fmt_loc!("response list is empty"))
        .cloned()
}

pub fn write_output(path: &str, value: &str) -> StackResult<()> {
    std::fs::write(path, value)
        .map_err(StackError::from)
        .stack_err_msg(fmt_loc!("can't write to {path}"))
}

#[tokio::main]
async fn main() -> StackResult<()> {
    let body = download_json("https://example.invalid/list.json").await?;
    let first = parse_first(&body)?;
    write_output("first.txt", &first)?;
    Ok(())
}
```

In this case, if the program failed at the `write_output` stage, you could get an error message like this:

```
Error: Permission denied (os error 13)
src/main.rs:27 can't write to first.txt
```

In a larger program, a permission denied error could happen in many parts of your codebase. When debugging, you want to quickly understand how you got to the error. `stackerror` makes it easy to get the precise context you need to debug quickly.

## Runtime error handling

There are some errors that you might want to handle at runtime instead of bubbling up to your main. A common example of this is handling a 429 HTTP status code: in this case you typically want to wait and retry.

In the above example, you could retry the `download_json` call:

```rust
#[tokio::main]
async fn main() -> StackResult<()> {
    let body = loop {
        match download_json("https://example.invalid/list.json").await {
            Ok(body) => break body,
            Err(err) if err.err_code() == Some(&ErrorCode::HttpTooManyRequests) => {
                tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
                continue;
            }
            Err(err) => return Err(err),
        }
    };
    let first = parse_first(&body)?;
    write_output("first.txt", &first)?;
    Ok(())
}
```

## One error type for your crate

When you publish a library (or just want a stable surface inside a bigger app), it helps to expose one error type and keep the matchable detail in a lightweight error code. Rust already models this pattern widely: `std::io::Error` has `ErrorKind`, `serde_json::Error` has `Category`, HTTP clients have `StatusCode`, etc. `stackerror` gives you that without ceremony.

Create a newtype that wraps `StackError`, and use the `#[derive_stack_error]` macro to get a drop-in replacement for `StackError` that is unique to your library.

```rust
// src/errors.rs
pub use stackerror::prelude::*;

#[derive_stack_error]
pub struct LibError(StackError);

pub type LibResult<T> = Result<T, LibError>;
```

Since you own `LibError`, you can implement traits on it such as `From` implementations to convert from common error types you encounter in your codebase. You can even introduce your own error codes simply by defining an enum called `ErrorCode` in the scope where `#[derive_stack_error]` is used.

## Are error codes just another "one big enum"?

In `stackerror`, codes are small, stable categories you branch on, not a project‑wide catch‑all. For recoverable failures, an error code is often all you need to decide the next step: retry, back off, ask the user, try a different resource etc. Most recoverable errors live at IO boundaries (with HTTP as a subset), so the actionable space is naturally finite and already standardized. `stackerror` maps these directly: IO codes mirror `std::io::ErrorKind` (e.g., not found, already exists, permission denied), HTTP codes mirror status codes (e.g., 429 Too Many Requests, 404 Not Found, 503 Service Unavailable), and a small set of runtime categories covers common non‑IO control flow.

While Rust's type system is able to push most errors to the IO boundary, by making invalid states unpresentable, some errors do still come up when an invalid parameter is used (e.g. an invalid index), or some data is structured incorrectly (e.g. an invalid JSON payload). `stackerror` introduces a few runtime error codes inspired by Python's exceptions to cover these cases. And you can always extend `stackerror`'s built-in error codes to suit your needs.

Codes remain stable while messages evolve. You branch on an `ErrorCode` for control flow, while human‑readable messages carry rich, stacked breadcrumbs for debugging. This separation keeps runtime handling precise without growing a sprawling enum or coupling callers to your crate’s internals.

If your application domain requires a much richer taxonomy of categories to instruct how to handle recoverable errors, then a dedicated enum-based error solution like `thiserror` or `snafu` might be more suitable. But remember not to conflate the needs of the caller and the needs of the debugging programmer.

## Conclusion

Rust gives us powerful building blocks for errors, but day-to-day work benefits from a pattern that stays ergonomic while scaling: *write* with minimal friction, *debug* with clear stacked context, and *handle at runtime* by matching on well-typed codes. `stackerror` is designed for exactly that workflow.

Putting it all together:

```rust
// src/errors.rs
pub use stackerror::prelude::*;
use stackerror::derive_stack_error;

#[derive_stack_error]
pub struct LibError(StackError);

pub type LibResult<T> = Result<T, LibError>;
```

```rust
// src/main.rs
use crate::errors::*;

pub async fn download_json(url: &str) -> LibResult<String> {
    reqwest::get(url)
        .await
        .map_err(LibError::from)
        .stack_err_msg(fmt_loc!("can't get {url}"))?
        .text()
        .await
        .map_err(LibError::from)
        .stack_err_msg(fmt_loc!("response text is invalid"))
}

pub fn parse_first(body: &str) -> LibResult<String> {
    serde_json::from_str::<Vec<String>>(body)
        .map_err(LibError::from_msg)
        .stack_err_msg(fmt_loc!("response isn't a valid JSON array"))?
        .first()
        .ok_or_else(LibError::new)
        .with_err_msg(fmt_loc!("response list is empty"))
        .cloned()
}

pub fn write_output(path: &str, value: &str) -> LibResult<()> {
    std::fs::write(path, value)
        .map_err(LibError::from)
        .stack_err_msg(fmt_loc!("can't write to {path}"))
}

#[tokio::main]
async fn main() -> LibResult<()> {
    let body = loop {
        match download_json("https://example.invalid/list.json").await {
            Ok(body) => break body,
            Err(err) if err.err_code() == Some(&ErrorCode::HttpTooManyRequests) => {
                tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
                continue;
            }
            Err(err) => return Err(err),
        }
    };
    let first = parse_first(&body)?;
    write_output("first.txt", &first)?;
    Ok(())
}
```