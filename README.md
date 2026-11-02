# xi-rpc

[![CI](https://github.com/loc-oss/xi-rpc/actions/workflows/ci.yml/badge.svg)](https://github.com/loc-oss/xi-rpc/actions/workflows/ci.yml)
[![Rust](https://img.shields.io/badge/rust-2024-orange.svg)](https://www.rust-lang.org/)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)

Utilities for building peers (both client and server side) for xi's JSON RPC variant.

`xi-rpc` provides a flexible, thread-safe framework for asynchronous and synchronous communication over JSON-RPC. Unlike the strict JSON-RPC 2.0 spec, this implementation allows requests and notifications in both directions, making it perfect for bi-directional communication between an editor core, frontends, and plugins.

## Features

- **Bi-directional:** Both sides can send requests and notifications.
- **Async & Sync:** Send requests asynchronously with callbacks, or synchronously and wait for the result.
- **Idle & Timers:** Built-in support for scheduling background work and timers when the main loop is idle.
- **Thread-Safe:** Utilizes `crossbeam-utils` scoped threads and `Mutex`/`Condvar` for safe state sharing.
- **API Stability:** Fully typed `Handler` and `RpcCtx` traits make integration seamless.
- **Rust 2024 Edition:** Fully migrated, clippy-clean, and modernized.

## Usage

Add `xi-rpc` to your `Cargo.toml` (as it is not on crates.io yet, link to the git tag):

```toml
[dependencies]
xi-rpc = { git = "https://github.com/loc-oss/xi-rpc", tag = "v0.4.0" }
```

### Basic Example

To use `xi-rpc`, you define your RPC methods by implementing the `Handler` trait, and then run the `RpcLoop`.

```rust
use serde_json::{json, Value};
use xi_rpc::{Handler, RpcCtx, RpcLoop, RemoteError, RpcCall};
use std::io::Cursor;

// 1. Define a handler for incoming requests and notifications
struct MyHandler;

impl Handler for MyHandler {
    type Notification = RpcCall;
    type Request = RpcCall;

    fn handle_notification(&mut self, _ctx: &RpcCtx, rpc: Self::Notification) {
        println!("Received notification: {}", rpc.method);
    }

    fn handle_request(&mut self, _ctx: &RpcCtx, rpc: Self::Request) -> Result<Value, RemoteError> {
        // Echo the params back as the result
        Ok(rpc.params)
    }
}

fn main() {
    let mut handler = MyHandler;
    let mut rpc_looper = RpcLoop::new(std::io::stdout());

    // 2. Provide a BufRead stream (e.g., stdin, socket, or a test cursor)
    let input_stream = Cursor::new(r#"{"id": 1, "method": "ping", "params": {"msg": "hello"}}"#);

    // 3. Run the main loop (blocks until EOF or error)
    rpc_looper.mainloop(|| input_stream, &mut handler).unwrap();
}
```

### Advanced Usage: Async Requests & Idle Tasks

The `RpcCtx` passed to every handler method allows you to interact with the peer, send async requests, and schedule background tasks.

```rust
impl Handler for MyHandler {
    type Notification = RpcCall;
    type Request = RpcCall;

    fn handle_request(&mut self, ctx: &RpcCtx, rpc: Self::Request) -> Result<Value, RemoteError> {
        let peer = ctx.get_peer();

        // 1. Send a notification to the other side
        peer.send_rpc_notification("update_ui", &json!({"status": "busy"}));

        // 2. Send an async request (does not block the main loop)
        peer.send_rpc_request_async("fetch_data", &json!({"id": 42}), Box::new(|result| {
            match result {
                Ok(val) => println!("Got data: {:?}", val),
                Err(e) => eprintln!("Error fetching data: {:?}", e),
            }
        }));

        // 3. Schedule an idle task to run when the event loop is quiet
        ctx.schedule_idle(1337); // Will call `self.idle(ctx, 1337)` later

        Ok(json!(null))
    }

    fn idle(&mut self, _ctx: &RpcCtx, token: usize) {
        println!("Running background task: {}", token);
    }
}
```

## Development & Testing

The project is protected against regressions via a comprehensive test suite. To run the same checks as our CI pipeline:

```bash
cargo test
cargo test --all-features
cargo test --no-default-features
cargo clippy --all-targets -- -D warnings
```

Or simply run the provided script:

```bash
chmod +x run_all_test
./run_all_test
```

## License

Apache-2.0 — see [`LICENSE`](LICENSE) and [`NOTICE`](NOTICE) for details.
