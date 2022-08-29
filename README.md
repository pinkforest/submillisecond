# submillisecond

A [lunatic][0] web framework for the Rust language.

Submillisecond is an attempt to build a **backend** web framework around the Rust language,
[WebAssembly's][1] security and the [lunatic scheduler][2].

> This is an early stage project, probably has bugs and the API is still changing. It's also
> important to point out that many Rust crates don't compile to WebAssembly yet and can't be used
> with submillisecond.

# Features

- Fast compilation times
- async-free - All preemption and scheduling is done by [lunatic][2]
- strong security - Each request is handled in a separate _lunatic_ process
- Batteries included - TODO

# Code example

```rust
use submillisecond::{router, Application};

fn index() -> &'static str {
    "Hello :)"
}

fn main() -> std::io::Result<()> {
    Application::new(router! {
        GET "/" => index
    })
    .serve("0.0.0.0:3000")
}

```

### Getting started with lunatic

To run the example you will first need to download the lunatic runtime by following the
installation steps in [this repository][2]. The runtime is just single executable and runs on
Windows, macOS and Linux. If you have already Rust installed, you can get it with:
```bash
cargo install lunatic-runtime
```

[Lunatic][2] applications need to be compiled to [WebAssembly][1] before they can be executed by
the runtime. Rust has great support for WebAssembly and you can build a lunatic compatible
application just by passing the `--target=wasm32-wasi` flag to cargo, e.g:

```bash
# Add the WebAssembly target
rustup target add wasm32-wasi
# Build the app
cargo build --release --target=wasm32-wasi
```

This will generate a .wasm file in the `target/wasm32-wasi/release/` folder inside your project.
You can now run your application by passing the generated .wasm file to Lunatic, e.g:

```
lunatic target/wasm32-wasi/release/<name>.wasm
```

#### Better developer experience

To simplify developing, testing and running lunatic applications with cargo, you can add a
`.cargo/config.toml` file to your project with the following content:

```toml
[build]
target = "wasm32-wasi"

[target.wasm32-wasi]
runner = "lunatic"
```

Now you can just use the commands you are already familiar with, such as `cargo run`, `cargo test`
and cargo is going to automatically build your project as a WebAssembly module and run it inside
`lunatic`.

### Testing

Lunatic provides a macro `#[lunatic::test]` to turn your tests into processes. Check out the
`tests` folder for examples.

### Getting started with submillisecond

## Handlers

Handlers are just functions that can define zero or more extractors.

```rust
fn index(body: Vec<u8>, cookies: Cookies) -> String {
    // ...
}
```

Handlers can return anything that can implements `IntoResponse`.

## Routers

Submillisecond provides a `router!` macro:

```rust
#[derive(NamedParam)]
struct User {
    first_name: String,
    last_name: String,
}

fn hi(user: User) -> String {
    format!("Hi {} {}!", user.first_name, user.last_name)
}

fn main() -> std::io::Result<()> {
    Application::new(router! {
        GET "/hi/:first_name/:last_name" => hi
        POST "/update_data" => update_age
    })
    .serve("0.0.0.0:3000")
}
```

GET parameters can be captured with an extractor.

### Nested routes

Routes can be nested:

```rust
router! {
    "/foo" => {
        GET "/bar" => bar
    }
}
```

`_` can be used as a catch-all helper:

```rust
router! {
    "/foo" => {
        GET "/bar" => bar
        _ => matches_foo_but_not_bar
    }
    _ => not_found
}
```

### Guards

Sub/routes can be protected by a guard:

```rust
struct ContentLengthLimit(u64);

impl Guard for ContentLengthLimit {
    fn check(&self, req: &RequestContext) -> bool {
        // ...
    }
}

fn main() -> std::io::Result<()> {
    Application::new(router! {
        "/short_requests" if ContentLengthGuard(128) => {
             POST "/super" if ContentLengthGuard(64) => super_short
             POST "/" => short
        }
       
    })
    .serve("0.0.0.0:3000")
}
```

Guards can be chained with the `&&` and `||` syntax.

## Middleware

Middleware is just a function, in fact they are handlers (can use extractors).

```rust
fn logger(req: RequestContext) -> Response {
    // before next handler
    let result = req.next_handler();
    // after next handler
    result
}

fn main() -> std::io::Result<()> {
    Application::new(router! {
        with logger;

        GET "/hi/:first_name/:last_name" => hi
    })
    .serve("0.0.0.0:3000")
}
```

Middleware can be chained together or only be used in sub-routes:

```rust
router! {
     with mid1;
     with mid2;

    "/foo" => {
        with foo_mid1;
        with foo_mid2;
    }
}
```

# License

Licensed under either of

- Apache License, Version 2.0, (http://www.apache.org/licenses/LICENSE-2.0)
- MIT license (http://opensource.org/licenses/MIT)

at your option.

[0]: https://lunatic.solutions
[1]: https://webassembly.org
[2]: https://github.com/lunatic-solutions/lunatic

