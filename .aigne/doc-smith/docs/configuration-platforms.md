# Platform Support

This document outlines the platforms on which Tokio is officially supported, as well as specific considerations for environments like WebAssembly (WASM).

## Guaranteed Support

Tokio guarantees support for the following platforms, meaning they are actively tested and maintained. Future releases of Tokio will continue to support these platforms, though minimum version requirements (like libc version on Linux or API level on Android) may change over time.

- Linux
- Windows
- Android (API level 21 and later)
- macOS
- iOS
- FreeBSD

## Other Platforms

Beyond the guaranteed platforms, Tokio is intended to work on all platforms supported by the underlying `mio` crate. For a more extensive list, please refer to [mio's platform documentation](https://crates.io/crates/mio#platforms).

However, support for these additional platforms is not guaranteed and may be discontinued in future releases. Please note that Wine is considered a different platform from Windows; consult the `mio` documentation for details on Wine support.

## WASM Support

Tokio provides limited but dedicated support for the WebAssembly (`WASM`) platform.

### Stable Support

When using a stable toolchain without the `tokio_unstable` flag, the following features are supported on `WASM`:

| Feature | Description |
| :--- | :--- |
| `sync` | Synchronization primitives like channels and mutexes. |
| `macros` | Procedural macros such as `#[tokio::main]`. |
| `io-util` | Core I/O utility traits and functions. |
| `rt` | The single-threaded runtime for scheduling tasks. |
| `time` | Utilities for time-based operations like `sleep` and `interval`. |

Enabling any feature not on this list (including the `full` feature) will result in a compilation failure.

**Important Considerations:**

- The `time` module will only function on `WASM` targets that provide timer support (e.g., `wasm32-wasi`). Calling timing functions on a platform without timer support will cause a panic.
- If the Tokio runtime on `WASM` becomes indefinitely idle, it will panic immediately instead of blocking forever. On platforms without timer support, this means the runtime can never be idle.

### Unstable WASM Support

By enabling the `tokio_unstable` configuration flag, you can access additional, unstable support for `WASM`.

This enables the use of the `tokio::net` module on the `wasm32-wasi` target. However, due to current limitations in the `WASI` standard, creating new sockets from within `WASM` is not supported. Therefore, network sockets must be created using the `FromRawFd` trait from a file descriptor provided by the host environment.