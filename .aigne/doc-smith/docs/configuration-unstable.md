# Unstable Features

Some feature flags and parts of the API are only available when the `tokio_unstable` configuration flag is enabled. These features are considered unstable, meaning their public API may change or be removed in subsequent 1.x releases without following the standard semantic versioning (semver) conventions.

### How to Enable Unstable Features

To use unstable features, you must explicitly opt-in by passing the `--cfg tokio_unstable` flag to `rustc` during compilation. This is a deliberate step to ensure users acknowledge the potential for breaking changes.

There are two primary ways to enable this flag for your project:

**1. Using Cargo Configuration File**

You can specify the flag in your project's `.cargo/config.toml` file. This is often the most convenient method for a project.

```toml .cargo/config.toml icon=mdi:folder-open
[build]
rustflags = ["--cfg", "tokio_unstable"]
```

<div class="warning">
**Important:** This configuration must be placed in a `.cargo/config.toml` file at the root of your project, **not** in `Cargo.toml`.
</div>

**2. Using Environment Variables**

Alternatively, you can set the `RUSTFLAGS` environment variable before running Cargo commands.

For most Unix-like shells:
```sh RUSTFLAGS
export RUSTFLAGS="--cfg tokio_unstable"
cargo build
```

For Windows PowerShell:
```powershell RUSTFLAGS
$Env:RUSTFLAGS="--cfg tokio_unstable"
cargo build
```

This approach is necessary because Cargo does not yet have direct support for per-crate unstable feature opt-ins. You can read more about the reasoning in [this Rust internals discussion](https://internals.rust-lang.org/t/feature-request-unstable-opt-in-non-transitive-crate-features/16193#why-not-a-crate-feature-2).

### Unstable Feature Flags

The following feature flags require the `tokio_unstable` cfg to be enabled:

| Feature | Description |
|---|---|
| `tracing` | Enables internal tracing events that can be consumed by libraries like `tracing-subscriber`. |
| `tokio_taskdump` | Enables the task dumping feature for debugging. Currently only supported on Linux (`aarch64`, `x86`, `x86_64`). |


### Unstable APIs

The following APIs are only available when unstable features are enabled:

- [`task::Builder`](./tasks-scheduling-spawning.md)
- Certain methods on [`task::JoinSet`](./tasks-scheduling-spawning.md)
- [`runtime::RuntimeMetrics`](./tasks-scheduling-runtime.md)
- [`runtime::Builder::on_task_spawn`](./tasks-scheduling-runtime.md)
- [`runtime::Builder::on_task_terminate`](./tasks-scheduling-runtime.md)
- [`runtime::Builder::unhandled_panic`](./tasks-scheduling-runtime.md)
- [`runtime::TaskMeta`](./tasks-scheduling-runtime.md)

### Unstable WASM Support

Tokio also has unstable support for some additional WASM features, which requires the `tokio_unstable` flag.

Enabling this flag allows the use of `tokio::net` on the `wasm32-wasi` target. However, there are limitations. Since `WASI` does not currently support creating new sockets from within a WASM module, sockets must be created using the `FromRawFd` trait from a file descriptor provided by the WASM host environment.