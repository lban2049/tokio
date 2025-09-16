# 不稳定功能

只有在启用 `tokio_unstable` 配置标志时，某些功能标志和 API 的一部分才可用。这些功能被认为是不稳定的，这意味着它们的公共 API 可能会在后续的 1.x 版本中更改或删除，而无需遵循标准的语义化版本（semver）约定。

### 如何启用不稳定功能

要使用不稳定功能，你必须在编译期间通过向 `rustc` 传递 `--cfg tokio_unstable` 标志来显式选择启用。这是一个有意的步骤，旨在确保用户知晓可能发生的破坏性变更。

有两种主要方法为你的项目启用此标志：

**1. 使用 Cargo 配置文件**

你可以在项目的 `.cargo/config.toml` 文件中指定该标志。这通常是项目中最方便的方法。

```toml .cargo/config.toml icon=mdi:folder-open
[build]
rustflags = ["--cfg", "tokio_unstable"]
```

<div class="warning">
**重要提示：** 此配置必须放在项目根目录下的 `.cargo/config.toml` 文件中，**而不是**放在 `Cargo.toml` 中。
</div>

**2. 使用环境变量**

或者，你可以在运行 Cargo 命令之前设置 `RUSTFLAGS` 环境变量。

对于大多数类 Unix 的 shell：
```sh RUSTFLAGS
export RUSTFLAGS="--cfg tokio_unstable"
cargo build
```

对于 Windows PowerShell：
```powershell RUSTFLAGS
$Env:RUSTFLAGS="--cfg tokio_unstable"
cargo build
```

这种方法是必要的，因为 Cargo 尚不直接支持按 crate 启用不稳定功能。你可以在 [这篇 Rust 内部讨论](https://internals.rust-lang.org/t/feature-request-unstable-opt-in-non-transitive-crate-features/16193#why-not-a-crate-feature-2) 中阅读更多关于其原因的讨论。

### 不稳定功能标志

以下功能标志需要启用 `tokio_unstable` cfg：

| 功能 | 描述 |
|---|---|
| `tracing` | 启用可被 `tracing-subscriber` 等库使用的内部追踪事件。 |
| `tokio_taskdump` | 启用用于调试的任务转储功能。目前仅在 Linux (`aarch64`, `x86`, `x86_64`) 上受支持。 |


### 不稳定 API

以下 API 仅在启用不稳定功能时可用：

- [`task::Builder`](./tasks-scheduling-spawning.md)
- [`task::JoinSet`](./tasks-scheduling-spawning.md) 上的某些方法
- [`runtime::RuntimeMetrics`](./tasks-scheduling-runtime.md)
- [`runtime::Builder::on_task_spawn`](./tasks-scheduling-runtime.md)
- [`runtime::Builder::on_task_terminate`](./tasks-scheduling-runtime.md)
- [`runtime::Builder::unhandled_panic`](./tasks-scheduling-runtime.md)
- [`runtime::TaskMeta`](./tasks-scheduling-runtime.md)

### 不稳定的 WASM 支持

Tokio 还为一些额外的 WASM 功能提供了不稳定的支持，这需要 `tokio_unstable` 标志。

启用此标志允许在 `wasm32-wasi` 目标上使用 `tokio::net`。然而，存在一些限制。由于 `WASI` 目前不支持在 WASM 模块内创建新套接字，因此必须使用 WASM 主机环境提供的文件描述符，通过 `FromRawFd` trait 来创建套接字。