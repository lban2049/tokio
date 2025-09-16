# Feature Flags

Tokio is designed to be modular, allowing you to include only the parts you need. This is managed through a set of feature flags in your `Cargo.toml` file, which helps minimize dependencies and reduce the final compiled binary size. By default, no features are enabled.

## Recommendations for Usage

How you configure feature flags typically depends on whether you are building an application or a library.

### For Applications

When building an application, it's often easiest to get started by enabling all of Tokio's features. This ensures that you have access to the full suite of tools and won't run into missing components as you build. If you're unsure which features you need, `full` is a safe and recommended choice.

```toml Cargo.toml icon=simple-icons:cargotoolbox
[dependencies]
tokio = { version = "1", features = ["full"] }
```

### For Libraries

As a library author, your goal should be to create a lightweight crate. To achieve this, you should only enable the specific features your library requires. This allows users of your library to avoid pulling in unnecessary dependencies.

For example, a library that only needs to spawn tasks and use TCP streams would enable the `rt` and `net` features:

```toml Cargo.toml icon=simple-icons:cargotoolbox
[dependencies]
tokio = { version = "1", features = ["rt", "net"] }
```

## Available Flags

Here is a detailed list of the available feature flags and the functionality they enable.

| Feature Flag | Description |
| :--- | :--- |
| `full` | Enables all public APIs listed below except `test-util` and `tracing`. Recommended for applications. |
| `rt` | Enables `tokio::spawn`, the current-thread scheduler, and related non-scheduler utilities. |
| `rt-multi-thread` | Enables the powerful, multi-threaded, work-stealing scheduler for high-concurrency applications. |
| `macros` | Enables the `#[tokio::main]` and `#[tokio::test]` procedural macros for convenient runtime setup. |
| `sync` | Enables all synchronization primitives in `tokio::sync`, such as channels (`mpsc`, `oneshot`, etc.) and `Mutex`. |
| `time` | Enables time-related utilities in `tokio::time`, including `sleep`, `interval`, and `timeout`. |
| `net` | Enables networking primitives in `tokio::net`, such as `TcpStream`, `UdpSocket`, and `UnixStream`. |
| `fs` | Enables asynchronous filesystem operations in `tokio::fs`. |
| `process` | Enables types for spawning and managing child processes in `tokio::process`. |
| `signal` | Enables handling of OS signals (like `SIGINT`) in `tokio::signal`. |
| `io-util` | Enables I/O utility traits and combinators, providing an async counterpart to `std::io`. |
| `io-std` | Enables asynchronous `Stdout`, `Stdin`, and `Stderr`. |
| `test-util` | Enables testing infrastructure for the Tokio runtime. Not intended for production use. |
| `parking_lot` | Uses the `parking_lot` crate's synchronization primitives internally as a potential optimization. |

**Note:** The core I/O traits, `AsyncRead` and `AsyncWrite`, are always available and do not require any feature flags to be enabled.

## Unstable Features

Tokio also has features that are considered unstable. Their APIs are not bound by semantic versioning and may change in minor releases. These require an explicit opt-in to use.

For a complete guide on how to enable and use these, please see the [Unstable Features](./configuration-unstable.md) documentation.