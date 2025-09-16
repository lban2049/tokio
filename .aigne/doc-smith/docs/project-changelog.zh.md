# 更新日志

本文档详细记录了 Tokio crate 每个版本的所有变更，包括新功能、错误修复和改进。它可作为跟踪该库演进的参考。

## 1.47.1 (2025年8月1日)

### 已修复

- process: 修复了由虚假的 pidfd 唤醒引起的 panic ([#7494])
- sync: 修复了 `SetOnce` 文档中 Python `asyncio.Event` 的损坏链接 ([#7485])

[#7494]: https://github.com/tokio-rs/tokio/pull/7494
[#7485]: https://github.com/tokio-rs/tokio/pull/7485

## 1.47.0 (2025年7月25日)

此版本在 `coop` 模块中添加了用于协作调度的 `poll_proceed` 和 `cooperative`，在 `sync` 模块中添加了提供与 [`std::sync::OnceLock`] 类似功能的 `SetOnce`，并添加了一个新方法 `sync::Notify::notified_owned()`，该方法返回一个不带生命周期参数的 `OwnedNotified`。

### 已添加

- coop: 添加 `cooperative` 和 `poll_proceed` ([#7405])
- sync: 添加 `SetOnce` ([#7418])
- sync: 添加 `sync::Notify::notified_owned()` ([#7465])

### 已更改

- deps: 升级 windows-sys 从 0.52 到 0.59 ([#7117])
- deps: 更新至 socket2 v0.6 ([#7443])
- sync: 改进 `AtomicWaker::wake` 的性能 ([#7450])

### 文档

- metrics: 修复了部分指标所列出的功能要求 ([#7449])
- runtime: 改进 `Readiness<'_>` 的安全性注释 ([#7415])

[#7117]: https://github.com/tokio-rs/tokio/pull/7117
[#7405]: https://github.com/tokio-rs/tokio/pull/7405
[#7415]: https://github.com/tokio-rs/tokio/pull/7415
[#7418]: https://github.com/tokio-rs/tokio/pull/7418
[#7443]: https://github.com/tokio-rs/tokio/pull/7443
[#7449]: https://github.com/tokio-rs/tokio/pull/7449
[#7450]: https://github.com/tokio-rs/tokio/pull/7450
[#7465]: https://github.com/tokio-rs/tokio/pull/7465

## 1.46.1 (2025年7月4日)

此版本修复了使用 `tokio::spawn` 而非 `Runtime::spawn` 生成的任务在运行时任务钩子中产生的生成位置不正确的问题。此问题仅影响 `TaskMeta::spawned_at` 中的生成位置，不影响 Tracing 事件中的任务位置。

### 非稳定版

- runtime: 添加 `TaskMeta::spawn_location` 以跟踪任务的生成位置 ([#7440])

[#7440]: https://github.com/tokio-rs/tokio/pull/7440

## 1.46.0 (2025年7月2日)

### 已修复

- net: 修复了 `TcpStream::shutdown` 在 macOS 上错误返回 error 的问题 ([#7290])

### 已添加

- sync: `mpsc::OwnedPermit::{same_channel, same_channel_as_sender}` 方法 ([#7389])
- macros: 为 `join!` 和 `try_join!` 添加 `biased` 选项，类似于 `select!` ([#7307])
- net: 支持 cygwin ([#7393])
- net: 在 Android 上支持 `pipe::OpenOptions::read_write` ([#7426])
- net: 为 `net::unix::SocketAddr` 添加 `Clone` 实现 ([#7422])

### 已更改

- runtime: 在操作 `queue::Local<T>` 时消除了不必要的 lfence ([#7340])
- task: 禁止在 `LocalSet::{poll,drop}` 中阻塞 ([#7372])

### 非稳定版

- runtime: 添加 `TaskMeta::spawn_location` 以跟踪任务的生成位置 ([#7417])
- runtime: 从 `runtime::Builder::build_local` 的 `LocalOptions` 参数中移除了借用 ([#7346])

### 文档

- io: 阐明了在未使用 `start_seek` 时的寻道行为 ([#7366])
- io: 记录了 `AsyncWriteExt::flush` 的取消安全性 ([#7364])
- net: 修复了 `recv_buffer_size` 方法的文档 ([#7336])
- net: 修复了 `TcpSocket` 文档中 `RawFd` 的损坏链接 ([#7416])
- net: 更新 `AsRawFd` 文档链接至当前 Rust stdlib 位置 ([#7429])
- readme: 修复了反应器描述中的双句点 ([#7363])
- runtime: 添加文档说明 `on_*_task_poll` 是不稳定的 ([#7311])
- sync: 更新了关于分配失败的广播文档 ([#7352])
- time: 添加了 `time::advance` 的一个缺失的 panic 场景 ([#7394])

[#7290]: https://github.com/tokio-rs/tokio/pull/7290
[#7307]: https://github.com/tokio-rs/tokio/pull/7307
[#7311]: https://github.com/tokio-rs/tokio/pull/7311
[#7336]: https://github.com/tokio-rs/tokio/pull/7336
[#7340]: https://github.com/tokio-rs/tokio/pull/7340
[#7346]: https://github.com/tokio-rs/tokio/pull/7346
[#7352]: https://github.com/tokio-rs/tokio/pull/7352
[#7363]: https://github.com/tokio-rs/tokio/pull/7363
[#7364]: https://github.com/tokio-rs/tokio/pull/7364
[#7366]: https://github.com/tokio-rs/tokio/pull/7366
[#7372]: https://github.com/tokio-rs/tokio/pull/7372
[#7389]: https://github.com/tokio-rs/tokio/pull/7389
[#7393]: https://github.com/tokio-rs/tokio/pull/7393
[#7394]: https://github.com/tokio-rs/tokio/pull/7394
[#7416]: https://github.com/tokio-rs/tokio/pull/7416
[#7417]: https://github.com/tokio-rs/tokio/pull/7417
[#7422]: https://github.com/tokio-rs/tokio/pull/7422
[#7426]: https://github.com/tokio-rs/tokio/pull/7426
[#7429]: https://github.com/tokio-rs/tokio/pull/7429

## 1.45.1 (2025年5月24日)

此版本修复了 wasm32-unknown-unknown 目标上的一个回归问题。在该问题中，先前因调用 `Instant::now()` 而不会 panic 的代码开始出现故障。这是由于第一个基于时间的指标已稳定所致。

### 已修复

- 在 wasm32-unknown-unknown 上禁用基于时间的指标 ([#7322])

[#7322]: https://github.com/tokio-rs/tokio/pull/7322

## 1.45.0 (2025年5月5日)

### 已添加

- metrics: 稳定 `worker_total_busy_duration`、`worker_park_count` 和 `worker_unpark_count` ([#6899], [#7276])
- process: 添加 `Command::spawn_with` ([#7249])

### 已更改

- io: 部分 trait 实现不再要求 `Unpin` ([#7204])
- rt: 将 `runtime::Handle` 标记为 unwind safe ([#7230])
- time: 恢复内部 sharding 实现 ([#7226])

### 非稳定版

- rt: 移除备用的多线程运行时 ([#7275])

[#6899]: https://github.com/tokio-rs/tokio/pull/6899
[#7204]: https://github.com/tokio-rs/tokio/pull/7204
[#7226]: https://github.com/tokio-rs/tokio/pull/7226
[#7230]: https://github.com/tokio-rs/tokio/pull/7230
[#7249]: https://github.com/tokio-rs/tokio/pull/7249
[#7275]: https://github.com/tokio-rs/tokio/pull/7275
[#7276]: https://github.com/tokio-rs/tokio/pull/7276

## 1.44.2 (2025年4月5日)

此版本修复了广播通道中的一个健全性问题。该通道接受 `Send` 但 `!Sync` 的值。此前，通道在未同步的情况下对这些值调用 `clone()`。此版本通过同步对 `.clone()` 的调用来修复该通道（感谢 Austin Bonander 发现并报告此问题）。

### 已修复

- sync: 同步广播通道中的 `clone()` 调用 ([#7232])

[#7232]: https://github.com/tokio-rs/tokio/pull/7232

## 1.44.1 (2025年3月13日)

### 已修复

- rt: 在 `block_in_place` 上下文中跳过 defer 队列 ([#7216])

[#7216]: https://github.com/tokio-rs/tokio/pull/7216

## 1.44.0 (2025年3月7日)

此版本更改了套接字上的 `from_std` 方法，以便在提供阻塞套接字时会引发 panic。我们确定此更改不是一个破坏性变更，因为 Tokio 不应在阻塞套接字上运行。这样做会导致运行时挂起，应被视为一个错误。意外地将阻塞套接字传递给 Tokio 是最常见的用户错误之一。如果此更改给您带来了问题，请在 [#7172] 中评论。

### 已添加

 - coop: 添加 `task::coop` 模块 ([#7116])
 - process: 添加 `Command::get_kill_on_drop()` ([#7086])
 - sync: 添加 `broadcast::Sender::closed` ([#6685], [#7090])
 - sync: 添加 `broadcast::WeakSender` ([#7100])
 - sync: 添加 `oneshot::Receiver::is_empty()` ([#7153])
 - sync: 添加 `oneshot::Receiver::is_terminated()` ([#7152])

### 已修复

 - fs: `File` 上的空读取不应启动后台读取 ([#7139])
 - process: 对已退出的子进程调用 `start_kill` 不应失败 ([#7160])
 - signal: 修复 Windows 上的 `CTRL_CLOSE`、`CTRL_LOGOFF`、`CTRL_SHUTDOWN` ([#7122])
 - sync: 正确处理 mpsc drop 期间的 panic ([#7094])

### 变更

 - runtime: 清理注册集中的魔术数字 ([#7112])
 - coop: 使 coop 使用 waker 延迟策略进行让步 ([#7185])
 - macros: 使 `select!` 具有预算感知能力 ([#7164])
 - net: 将阻塞套接字传递给 `from_std` 时 panic ([#7166])
 - io: 清理缓冲区转换 ([#7142])

### 对非稳定版 API 的更改

 - rt: 添加任务轮询前后的回调 ([#7120])
 - tracing: 将任务 tracing API 设为非稳定版公开 ([#6972])

### 文档

 - docs: 修复顶层文档中章节的嵌套 ([#7159])
 - fs: 重命名符号链接和硬链接参数名称 ([#7143])
 - io: 在 simplex 文档测试中交换 reader/writer ([#7176])
 - macros: 关于 `select!` 替代方案的文档 ([#7110])
 - net: 重命名 `send_to` 的参数 ([#7146])
 - process: 添加读取 `Child` stdout 的示例 ([#7141])
 - process: 阐明 `Child::kill` 的行为 ([#7162])
 - process: 修复 `ChildStdin` 结构体文档注释的语法 ([#7192])
 - runtime: 一致地使用 `worker_threads` 而不是 `core_threads` ([#7186])

[#6685]: https://github.com/tokio-rs/tokio/pull/6685
[#6972]: https://github.com/tokio-rs/tokio/pull/6972
[#7086]: https://github.com/tokio-rs/tokio/pull/7086
[#7090]: https://github.com/tokio-rs/tokio/pull/7090
[#7094]: https://github.com/tokio-rs/tokio/pull/7094
[#7100]: https://github.com/tokio-rs/tokio/pull/7100
[#7110]: https://github.com/tokio-rs/tokio/pull/7110
[#7112]: https://github.com/tokio-rs/tokio/pull/7112
[#7116]: https://github.com/tokio-rs/tokio/pull/7116
[#7120]: https://github.com/tokio-rs/tokio/pull/7120
[#7122]: https://github.com/tokio-rs/tokio/pull/7122
[#7139]: https://github.com/tokio-rs/tokio/pull/7139
[#7141]: https://github.com/tokio-rs/tokio/pull/7141
[#7142]: https://github.com/tokio-rs/tokio/pull/7142
[#7143]: https://github.com/tokio-rs/tokio/pull/7143
[#7146]: https://github.com/tokio-rs/tokio/pull/7146
[#7152]: https://github.com/tokio-rs/tokio/pull/7152
[#7153]: https://github.com/tokio-rs/tokio/pull/7153
[#7159]: https://github.com/tokio-rs/tokio/pull/7159
[#7160]: https://github.com/tokio-rs/tokio/pull/7160
[#7162]: https://github.com/tokio-rs/tokio/pull/7162
[#7164]: https://github.com/tokio-rs/tokio/pull/7164
[#7166]: https://github.com/tokio-rs/tokio/pull/7166
[#7172]: https://github.com/tokio-rs/tokio/pull/7172
[#7176]: https://github.com/tokio-rs/tokio/pull/7176
[#7185]: https://github.com/tokio-rs/tokio/pull/7185
[#7186]: https://github.com/tokio-rs/tokio/pull/7186
[#7192]: https://github.com/tokio-rs/tokio/pull/7192

## 1.43.2 (2025年8月1日)

### 已修复

- process: 修复了由虚假的 pidfd 唤醒引起的 panic ([#7494])

[#7494]: https://github.com/tokio-rs/tokio/pull/7494

## 1.43.1 (2025年4月5日)

此版本修复了广播通道中的一个健全性问题。该通道接受 `Send` 但 `!Sync` 的值。此前，通道在未同步的情况下对这些值调用 `clone()`。此版本通过同步对 `.clone()` 的调用来修复该通道（感谢 Austin Bonander 发现并报告此问题）。

### 已修复

- sync: 同步广播通道中的 `clone()` 调用 ([#7232])

[#7232]: https://github.com/tokio-rs/tokio/pull/7232

## 1.43.0 (2025年1月8日)

### 已添加

- net: 添加 `UdpSocket::peek` 方法 ([#7068])
- net: 添加对 Haiku OS 的支持 ([#7042])
- process: 添加 `Command::into_std()` ([#7014])
- signal: 在 illumos 上添加 `SignalKind::info` ([#6995])
- signal: 在 illumos 上添加对实时信号的支持 ([#7029])

### 已修复

- io: 在 `Blocking` 中初始化向量前不要调用 `set_len` ([#7054])
- macros: 在 `#[tokio::main]` 中抑制 `clippy::needless_return` ([#6874])
- runtime: 修复 WebAssembly 上的线程停放 ([#7041])

### 变更

- chore: 为 `unsync_load` 使用 unsync 加载 ([#7073])
- io: 在 `Repeat` 的 read impl 中使用 `Buf::put_bytes` ([#7055])
- task: 尽快丢弃任务的 join waker ([#6986])

### 对非稳定版 API 的更改

- metrics: 提高 H2Histogram 配置的灵活性 ([#6963])
- taskdump: 添加回溯的访问器方法 ([#6975])

### 文档

- io: 阐明 `ReadBuf::uninit` 也允许初始化缓冲区 ([#7053])
- net: 修复 `TcpStream::try_write_vectored` 文档中的歧义 ([#7067])
- runtime: 修复 `LocalRuntime` 文档链接 ([#7074])
- sync: 扩展 `watch::Receiver::wait_for` 的文档 ([#7038])
- sync: 修复 `OnceCell` 文档中的拼写错误 ([#7047])

[#6874]: https://github.com/tokio-rs/tokio/pull/6874
[#6963]: https://github.com/tokio-rs/tokio/pull/6963
[#6975]: https://github.com/tokio-rs/tokio/pull/6975
[#6986]: https://github.com/tokio-rs/tokio/pull/6986
[#6995]: https://github.com/tokio-rs/tokio/pull/6995
[#7014]: https://github.com/tokio-rs/tokio/pull/7014
[#7029]: https://github.com/tokio-rs/tokio/pull/7029
[#7038]: https://github.com/tokio-rs/tokio/pull/7038
[#7041]: https://github.com/tokio-rs/tokio/pull/7041
[#7042]: https://github.com/tokio-rs/tokio/pull/7042
[#7047]: https://github.com/tokio-rs/tokio/pull/7047
[#7053]: https://github.com/tokio-rs/tokio/pull/7053
[#7054]: https://github.com/tokio-rs/tokio/pull/7054
[#7055]: https://github.com/tokio-rs/tokio/pull/7055
[#7067]: https://github.com/tokio-rs/tokio/pull/7067
[#7068]: https://github.com/tokio-rs/tokio/pull/7068
[#7073]: https://github.com/tokio-rs/tokio/pull/7073
[#7074]: https://github.com/tokio-rs/tokio/pull/7074

## 1.42.1 (2025年4月8日)

此版本修复了广播通道中的一个健全性问题。该通道接受 `Send` 但 `!Sync` 的值。此前，通道在未同步的情况下对这些值调用 `clone()`。此版本通过同步对 `.clone()` 的调用来修复该通道（感谢 Austin Bonander 发现并报告此问题）。

### 已修复

- sync: 同步广播通道中的 `clone()` 调用 ([#7232])

[#7232]: https://github.com/tokio-rs/tokio/pull/7232

## 1.42.0 (2024年12月3日)

### 已添加

- io: 添加 `AsyncFd::{try_io, try_io_mut}` ([#6967])

### 已修复

- io: 避免在 RegistrationSet 中出现 `ptr->ref->ptr` 往返 ([#6929])
- runtime: 不要在 `block_in_place` 内部延迟 `yield_now` ([#6999])

### 变更

- io: 简化 io 就绪逻辑 ([#6966])

### 文档

- net: 修复 `tokio::net::unix::{pid_t, gid_t, uid_t}` 的文档 ([#6791])
- time: 修复 `Instant` 文档中的一个拼写错误 ([#6982])

[#6791]: https://github.com/tokio-rs/tokio/pull/6791
[#6929]: https://github.com/tokio-rs/tokio/pull/6929
[#6966]: https://github.com/tokio-rs/tokio/pull/6966
[#6967]: https://github.com/tokio-rs/tokio/pull/6967
[#6982]: https://github.com/tokio-rs/tokio/pull/6982
[#6999]: https://github.com/tokio-rs/tokio/pull/6999

## 1.41.1 (2024年11月7日)

### 已修复

- metrics: 修复直方图桶数错误的 bug ([#6957])
- net: 在文档中显示 `net::UdpSocket` 的 `net` 要求 ([#6938])
- net: 修复 `TcpStream` 内部注释中的拼写错误 ([#6944])

[#6938]: https://github.com/tokio-rs/tokio/pull/6938
[#6944]: https://github.com/tokio-rs/tokio/pull/6944
[#6957]: https://github.com/tokio-rs/tokio/pull/6957

## 1.41.0 (2024年10月22日)

### 已添加

- metrics: 稳定 `global_queue_depth` ([#6854], [#6918])
- net: 添加 unix `SocketAddr` 的转换 ([#6868])
- sync: 添加 `watch::Sender::sender_count` ([#6836])
- sync: 添加 `mpsc::Receiver::blocking_recv_many` ([#6867])
- task: 稳定 `Id` API ([#6793], [#6891])

### 已添加 (非稳定版)

- metrics: 添加 H2 Histogram 选项以提高直方图粒度 ([#6897])
- metrics: 重命名一些直方图 API ([#6924])
- runtime: 添加 `LocalRuntime` ([#6808])

### 已更改

- runtime: 在发布模式下，将大于 16k 的 future 装箱 ([#6826])
- sync: 为 `Notified` 添加 `#[must_use]` ([#6828])
- sync: 使 `watch` 变为协作式 ([#6846])
- sync: 使 `broadcast::Receiver` 变为协作式 ([#6870])
- task: 将任务大小添加到 tracing 检测中 ([#6881])
- wasm: 为 `wasi` 目标启用 `cfg_fs` ([#6822])

### 已修复

- net: 修复 unix 套接字中抽象套接字路径的回归问题 ([#6838])

### 文档

- io: 推荐将 `OwnedFd` 与 `AsyncFd` 一起使用 ([#6821])
- io: 记录 `AsyncFd` 方法的取消安全性 ([#6890])
- macros: 为 `join` 和 `try_join` 渲染更易于理解的文档 ([#6814], [#6841])
- net: 修复 `TcpSocket::set_nodelay` 和 `TcpSocket::nodelay` 的交换示例 ([#6840])
- sync: 记录运行时兼容性 ([#6833])

[#6793]: https://github.com/tokio-rs/tokio/pull/6793
[#6808]: https://github.com/tokio-rs/tokio/pull/6808
[#6814]: https://github.com/tokio-rs/tokio/pull/6814
[#6821]: https://github.com/tokio-rs/tokio/pull/6821
[#6822]: https://github.com/tokio-rs/tokio/pull/6822
[#6826]: https://github.com/tokio-rs/tokio/pull/6826
[#6828]: https://github.com/tokio-rs/tokio/pull/6828
[#6833]: https://github.com/tokio-rs/tokio/pull/6833
[#6836]: https://github.com/tokio-rs/tokio/pull/6836
[#6838]: https://github.com/tokio-rs/tokio/pull/6838
[#6840]: https://github.com/tokio-rs/tokio/pull/6840
[#6841]: https://github.com/tokio-rs/tokio/pull/6841
[#6846]: https://github.com/tokio-rs/tokio/pull/6846
[#6854]: https://github.com/tokio-rs/tokio/pull/6854
[#6867]: https://github.com/tokio-rs/tokio/pull/6867
[#6868]: https://github.com/tokio-rs/tokio/pull/6868
[#6870]: https://github.com/tokio-rs/tokio/pull/6870
[#6881]: https://github.com/tokio-rs/tokio/pull/6881
[#6890]: https://github.com/tokio-rs/tokio/pull/6890
[#6891]: https://github.com/tokio-rs/tokio/pull/6891
[#6897]: https://github.com/tokio-rs/tokio/pull/6897
[#6918]: https://github.com/tokio-rs/tokio/pull/6918
[#6924]: https://github.com/tokio-rs/tokio/pull/6924

## 1.40.0 (2024年8月30日)

### 已添加

- io: 添加 `util::SimplexStream` ([#6589])
- process: 稳定 `Command::process_group` ([#6731])
- sync: 添加 `{TrySendError,SendTimeoutError}::into_inner` ([#6755])
- task: 添加 `JoinSet::join_all` ([#6784])

### 已添加 (非稳定版)

- runtime: 添加 `Builder::{on_task_spawn, on_task_terminate}` ([#6742])

### 已更改

- io: 在可能的情况下为 `write_all_buf` 使用向量化 io ([#6724])
- runtime: 防止 niche-optimization 以避免触发 miri ([#6744])
- sync: 将 mpsc 类型标记为 `UnwindSafe` ([#6783])
- sync,time: 使 `Sleep` 和 `BatchSemaphore` 检测显式根 ([#6727])
- task: 为 `task::Id` 使用 `NonZeroU64` ([#6733])
- task: 打印 `JoinError` 时包含 panic 消息 ([#6753])
- task: 为 `JoinHandle::abort_handle` 添加 `#[must_use]` ([#6762])
- time: 消除计时器轮的分配 ([#6779])

### 文档

- docs: 阐明 `[build]` 部分不应放在 Cargo.toml 中 ([#6728])
- io: 阐明零剩余容量的情况 ([#6790])
- macros: 改进 `select!` 的文档 ([#6774])
- sync: 记录 mpsc 通道的分配行为 ([#6773])

[#6589]: https://github.com/tokio-rs/tokio/pull/6589
[#6724]: https://github.com/tokio-rs/tokio/pull/6724
[#6727]: https://github.com/tokio-rs/tokio/pull/6727
[#6728]: https://github.com/tokio-rs/tokio/pull/6728
[#6731]: https://github.com/tokio-rs/tokio/pull/6731
[#6733]: https://github.com/tokio-rs/tokio/pull/6733
[#6742]: https://github.com/tokio-rs/tokio/pull/6742
[#6744]: https://github.com/tokio-rs/tokio/pull/6744
[#6753]: https://github.com/tokio-rs/tokio/pull/6753
[#6755]: https://github.com/tokio-rs/tokio/pull/6755
[#6762]: https://github.com/tokio-rs/tokio/pull/6762
[#6773]: https://github.com/tokio-rs/tokio/pull/6773
[#6774]: https://github.com/tokio-rs/tokio/pull/6774
[#6779]: https://github.com/tokio-rs/tokio/pull/6779
[#6783]: https://github.com/tokio-rs/tokio/pull/6783
[#6784]: https://github.com/tokio-rs/tokio/pull/6784
[#6790]: https://github.com/tokio-rs/tokio/pull/6790

## 1.39.3 (2024年8月17日)

此版本修复了 unix socket API 停止接受抽象套接字命名空间的回归问题。([#6772])

[#6772]: https://github.com/tokio-rs/tokio/pull/6772

## 1.39.2 (2024年7月27日)

此版本修复了 `select!` 宏停止接受使用临时生命周期扩展的表达式的回归问题。([#6722])

[#6722]: https://github.com/tokio-rs/tokio/pull/6722

## 1.39.1 (2024年7月23日)

此版本恢复了“time: avoid traversing entries in the time wheel twice”，因为它包含一个 bug。([#6715])

[#6715]: https://github.com/tokio-rs/tokio/pull/6715

## 1.39.0 (2024年7月23日)

已撤销。请改用 1.39.1。

- 此版本将 MSRV 提升至 1.70。([#6645])
- 此版本升级至 mio v1。([#6635])
- 此版本升级至 windows-sys v0.52 ([#6154])

### 已添加

- io: 为 `Empty` 实现 `AsyncSeek` ([#6663])
- metrics: 稳定 `num_alive_tasks` ([#6619], [#6667])
- process: 添加 `Command::as_std_mut` ([#6608])
- sync: 添加 `watch::Sender::same_channel` ([#6637])
- sync: 添加 `{Receiver,UnboundedReceiver}::{sender_strong_count,sender_weak_count}` ([#6661])
- sync: 为 `watch::Sender` 实现 `Default` ([#6626])
- task: 为 `AbortHandle` 实现 `Clone` ([#6621])
- task: 稳定 `consume_budget` ([#6622])

### 已更改

- io: 改进 `ReadBuf::put_slice()` 的 panic 消息 ([#6629])
- io: 在 `copy_bidirectional` 和 `copy` 中进行写时读 ([#6532])
- runtime: 将 `num_cpus` 替换为 `available_parallelism` ([#6709])
- task: 避免将大 future 传递给 `block_on` 时发生堆栈溢出 ([#6692])
- time: 避免在时间轮中两次遍历条目 ([#6584])
- time: 支持 `IntoFuture` 与 `timeout` ([#6666])
- macros: 支持 `IntoFuture` 与 `join!` 和 `select!` ([#6710])

### 已修复

- docs: 修复启用 fs 功能时的 docsrs 构建 ([#6585])
- io: 仅在已知兼容的平台上使用短读优化 ([#6668])
- time: 修复使用大持续时间与 `Interval` 时的溢出 panic ([#6612])

### 已添加 (非稳定版)

- macros: 允许 `#[tokio::main]` 和 `#[tokio::test]` 的 `unhandled_panic` 行为 ([#6593])
- metrics: 添加 `spawned_tasks_count` ([#6114])
- metrics: 添加 `worker_park_unpark_count` ([#6696])
- metrics: 添加工作线程 ID ([#6695])

### 文档

- io: 更新 `tokio::io::stdout` 文档 ([#6674])
- macros: 修复 `join.rs` 和 `try_join.rs` 中的拼写错误 ([#6641])
- runtime: 修复 `unhandled_panic` 中的拼写错误 ([#6660])
- task: 记录所有任务都在运行时 `JoinSet::try_join_next` 的行为 ([#6671])

[#6114]: https://github.com/tokio-rs/tokio/pull/6114
[#6154]: https://github.com/tokio-rs/tokio/pull/6154
[#6532]: https://github.com/tokio-rs/tokio/pull/6532
[#6584]: https://github.com/tokio-rs/tokio/pull/6584
[#6585]: https://github.com/tokio-rs/tokio/pull/6585
[#6593]: https://github.com/tokio-rs/tokio/pull/6593
[#6608]: https://github.com/tokio-rs/tokio/pull/6608
[#6612]: https://github.com/tokio-rs/tokio/pull/6612
[#6619]: https://github.com/tokio-rs/tokio/pull/6619
[#6621]: https://github.com/tokio-rs/tokio/pull/6621
[#6622]: https://github.com/tokio-rs/tokio/pull/6622
[#6626]: https://github.com/tokio-rs/tokio/pull/6626
[#6629]: https://github.com/tokio-rs/tokio/pull/6629
[#6635]: https://github.com/tokio-rs/tokio/pull/6635
[#6637]: https://github.com/tokio-rs/tokio/pull/6637
[#6641]: https://github.com/tokio-rs/tokio/pull/6641
[#6645]: https://github.com/tokio-rs/tokio/pull/6645
[#6660]: https://github.com/tokio-rs/tokio/pull/6660
[#6661]: https://github.com/tokio-rs/tokio/pull/6661
[#6663]: https://github.com/tokio-rs/tokio/pull/6663
[#6666]: https://github.com/tokio-rs/tokio/pull/6666
[#6667]: https://github.com/tokio-rs/tokio/pull/6667
[#6668]: https://github.com/tokio-rs/tokio/pull/6668
[#6671]: https://github.com/tokio-rs/tokio/pull/6671
[#6674]: https://github.com/tokio-rs/tokio/pull/6674
[#6692]: https://github.com/tokio-rs/tokio/pull/6692
[#6695]: https://github.com/tokio-rs/tokio/pull/6695
[#6696]: https://github.com/tokio-rs/tokio/pull/6696
[#6709]: https://github.com/tokio-rs/tokio/pull/6709
[#6710]: https://github.com/tokio-rs/tokio/pull/6710

## 1.38.2 (2025年4月2日)

此版本修复了广播通道中的一个健全性问题。该通道接受 `Send` 但 `!Sync` 的值。此前，通道在未同步的情况下对这些值调用 `clone()`。此版本通过同步对 `.clone()` 的调用来修复该通道（感谢 Austin Bonander 发现并报告此问题）。

### 已修复

- sync: 同步广播通道中的 `clone()` 调用 ([#7232])

[#7232]: https://github.com/tokio-rs/tokio/pull/7232

## 1.38.1 (2024年7月16日)

此版本修复了被识别为 ([#6682]) 的 bug，该 bug 导致计时器未在应触发时触发。

### 已修复

- time: 在持有分片时间轮的所有锁时更新 `wake_up` ([#6683])

[#6682]: https://github.com/tokio-rs/tokio/pull/6682
[#6683]: https://github.com/tokio-rs/tokio/pull/6683

## 1.38.0 (2024年5月30日)

此版本标志着运行时指标稳定化的开始。它稳定了 `RuntimeMetrics::worker_count`。未来的版本将继续稳定更多的指标。

### 已添加

- fs: 添加 `File::create_new` ([#6573])
- io: 添加 `copy_bidirectional_with_sizes` ([#6500])
- io: 为 `Join` 实现 `AsyncBufRead` ([#6449])
- net: 添加对 Apple visionOS 的支持 ([#6465])
- net: 为 `NamedPipeInfo` 实现 `Clone` ([#6586])
- net: 支持 QNX OS ([#6421])
- sync: 添加 `Notify::notify_last` ([#6520])
- sync: 添加 `mpsc::Receiver::{capacity,max_capacity}` ([#6511])
- sync: 向信号量许可添加 `split` 方法 ([#6472], [#6478])
- task: 添加 `tokio::task::join_set::Builder::spawn_blocking` ([#6578])
- wasm: 支持 rt-multi-thread 与 wasm32-wasi-preview1-threads ([#6510])

### 已更改

- macros: 使 `#[tokio::test]` 将 `#[test]` 附加到属性列表的末尾 ([#6497])
- metrics: 修复 `blocking_threads` 计数 ([#6551])
- metrics: 稳定 `RuntimeMetrics::worker_count` ([#6556])
- runtime: 在 `block_in_place` 中将任务移出 `lifo_slot` ([#6596])
- runtime: 如果 `global_queue_interval` 为零则 panic ([#6445])
- sync: 在 oneshot receiver 的析构函数中始终丢弃消息 ([#6558])
- sync: 为任务转储检测 `Semaphore` ([#6499])
- sync: 在唤醒成批 waker 时使用 FIFO 顺序 ([#6521])
- task: 使 `LocalKey::get` 与 Clone 类型一起工作 ([#6433])
- tests: 更新 nix 和 mio-aio 开发依赖 ([#6552])
- time: 清理实现 ([#6517])
- time: 在首次轮询时懒初始化计时器 ([#6512])
- time: 移除 `TimerShared` 中的 `true_when` 字段 ([#6563])
- time: 使用分片实现计时器 ([#6534])

### 已修复

- taskdump: 允许在非 unix 机器上构建 taskdump 文档 ([#6564])
- time: 检查 `Interval::poll_tick` 中的溢出 ([#6487])
- sync: 修复 mpsc 块边界上不正确的 `is_empty` ([#6603])

### 文档

- fs: 重写文件系统文档 ([#6467])
- io: 修复 `stdin` 文档 ([#6581])
- io: 修复 `ReadHalf::unsplit()` 文档中的过时引用 ([#6498])
- macros: 为 `select!` 渲染更易于理解的文档 ([#6468])
- net: 将缺失的类型添加到模块文档中 ([#6482])
- net: 修复误导性的 `NamedPipeServer` 示例 ([#6590])
- sync: 添加 `SemaphorePermit`, `OwnedSemaphorePermit` 的示例 ([#6477])
- sync: 记录 `Barrier::wait` 不是取消安全的 ([#6494])
- sync: 解释 `watch::Sender::{subscribe,closed}` 之间的关系 ([#6490])
- task: 阐明不能中止 `spawn_blocking` 任务 ([#6571])
- task: 修复 `LocalSet::run_until` 文档中的一个拼写错误 ([#6599])
- time: 修复文档中 pause 和 resume 的 test-util 要求 ([#6503])

[#6421]: https://github.com/tokio-rs/tokio/pull/6421
[#6433]: https://github.com/tokio-rs/tokio/pull/6433
[#6445]: https://github.com/tokio-rs/tokio/pull/6445
[#6449]: https://github.com/tokio-rs/tokio/pull/6449
[#6465]: https://github.com/tokio-rs/tokio/pull/6465
[#6467]: https://github.com/tokio-rs/tokio/pull/6467
[#6468]: https://github.com/tokio-rs/tokio/pull/6468
[#6472]: https://github.com/tokio-rs/tokio/pull/6472
[#6477]: https://github.com/tokio-rs/tokio/pull/6477
[#6478]: https://github.com/tokio-rs/tokio/pull/6478
[#6482]: https://github.com/tokio-rs/tokio/pull/6482
[#6487]: https://github.com/tokio-rs/tokio/pull/6487
[#6490]: https://github.com/tokio-rs/tokio/pull/6490
[#6494]: https://github.com/tokio-rs/tokio/pull/6494
[#6497]: https://github.com/tokio-rs/tokio/pull/6497
[#6498]: https://github.com/tokio-rs/tokio/pull/6498
[#6499]: https://github.com/tokio-rs/tokio/pull/6499
[#6500]: https://github.com/tokio-rs/tokio/pull/6500
[#6503]: https://github.com/tokio-rs/tokio/pull/6503
[#6510]: https://github.com/tokio-rs/tokio/pull/6510
[#6511]: https://github.com/tokio-rs/tokio/pull/6511
[#6512]: https://github.com/tokio-rs/tokio/pull/6512
[#6517]: https://github.com/tokio-rs/tokio/pull/6517
[#6520]: https://github.com/tokio-rs/tokio/pull/6520
[#6521]: https://github.com/tokio-rs/tokio/pull/6521
[#6534]: https://github.com/tokio-rs/tokio/pull/6534
[#6551]: https://github.com/tokio-rs/tokio/pull/6551
[#6552]: https://github.com/tokio-rs/tokio/pull/6552
[#6556]: https://github.com/tokio-rs/tokio/pull/6556
[#6558]: https://github.com/tokio-rs/tokio/pull/6558
[#6563]: https://github.com/tokio-rs/tokio/pull/6563
[#6564]: https://github.com/tokio-rs/tokio/pull/6564
[#6571]: https://github.com/tokio-rs/tokio/pull/6571
[#6573]: https://github.com/tokio-rs/tokio/pull/6573
[#6578]: https://github.com/tokio-rs/tokio/pull/6578
[#6581]: https://github.com/tokio-rs/tokio/pull/6581
[#6586]: https://github.com/tokio-rs/tokio/pull/6586
[#6590]: https://github.com/tokio-rs/tokio/pull/6590
[#6596]: https://github.com/tokio-rs/tokio/pull/6596
[#6599]: https://github.com/tokio-rs/tokio/pull/6599
[#6603]: https://github.com/tokio-rs/tokio/pull/6603

## 1.37.0 (2024年3月28日)

### 已添加

- fs: 向 `tokio::fs::File` 添加 `set_max_buf_size` ([#6411])
- io: 向 `AsyncFd` 添加 `try_new` 和 `try_with_interest` ([#6345])
- sync: 向信号量添加 `forget_permits` 方法 ([#6331])
- sync: 向 mpsc 接收器添加 `is_closed`、`is_empty` 和 `len` ([#6348])
- sync: 向拥有的 `RwLock` 守卫添加 `rwlock()` 方法 ([#6418])
- sync: 暴露 mpsc 发送器句柄的强弱计数 ([#6405])
- sync: 为 `watch::Sender` 实现 `Clone` ([#6388])
- task: 添加 `TaskLocalFuture::take_value` ([#6340])
- task: 为 `JoinSet` 实现 `FromIterator` ([#6300])

### 已更改

- io: 使 `io::split` 使用互斥锁而不是自旋锁 ([#6403])

### 已修复

- docs: 修复没有 net 功能的 docsrs 构建 ([#6360])
- macros: 允许只有 else 分支的 select ([#6339])
- runtime: 修复 os 注册失败时泄漏注册条目的问题 ([#6329])

### 文档

- io: 记录 `AsyncBufReadExt::fill_buf` 的取消安全性 ([#6431])
- io: 记录 `AsyncReadExt` 的原始读取函数的取消安全性 ([#6337])
- runtime: 添加从 `Runtime` 到 `#[tokio::main]` 的文档链接 ([#6366])
- runtime: 使 `enter` 示例具有确定性 ([#6351])
- sync: 添加用于限制传出请求数量的 Semaphore 示例 ([#6419])
- sync: 修复广播文档中缺失的句点 ([#6377])
- sync: 用 `#[must_use]` 标记 `mpsc::Sender::downgrade` ([#6326])
- sync: 在 `new_with` 之前重新排序 `const_new` ([#6392])
- sync: 更新 watch 通道文档 ([#6395])
- task: 修复文档链接 ([#6336])

### 已更改 (非稳定版)

- runtime: 在 taskdumps 中包含任务 `Id` ([#6328])
- runtime: 如果不支持 `unhandled_panic` 但已启用，则 panic ([#6410])

[#6300]: https://github.com/tokio-rs/tokio/pull/6300
[#6326]: https://github.com/tokio-rs/tokio/pull/6326
[#6328]: https://github.com/tokio-rs/tokio/pull/6328
[#6329]: https://github.com/tokio-rs/tokio/pull/6329
[#6331]: https://github.com/tokio-rs/tokio/pull/6331
[#6336]: https://github.com/tokio-rs/tokio/pull/6336
[#6337]: https://github.com/tokio-rs/tokio/pull/6337
[#6339]: https://github.com/tokio-rs/tokio/pull/6339
[#6340]: https://github.com/tokio-rs/tokio/pull/6340
[#6345]: https://github.com/tokio-rs/tokio/pull/6345
[#6348]: https://github.com/tokio-rs/tokio/pull/6348
[#6351]: https://github.com/tokio-rs/tokio/pull/6351
[#6360]: https://github.com/tokio-rs/tokio/pull/6360
[#6366]: https://github.com/tokio-rs/tokio/pull/6366
[#6377]: https://github.com/tokio-rs/tokio/pull/6377
[#6388]: https://github.com/tokio-rs/tokio/pull/6388
[#6392]: https://github.com/tokio-rs/tokio/pull/6392
[#6395]: https://github.com/tokio-rs/tokio/pull/6395
[#6403]: https://github.com/tokio-rs/tokio/pull/6403
[#6405]: https://github.com/tokio-rs/tokio/pull/6405
[#6410]: https://github.com/tokio-rs/tokio/pull/6410
[#6411]: https://github.com/tokio-rs/tokio/pull/6411
[#6418]: https://github.com/tokio-rs/tokio/pull/6418
[#6419]: https://github.com/tokio-rs/tokio/pull/6419
[#6431]: https://github.com/tokio-rs/tokio/pull/6431

## 1.36.0 (2024年2月2日)

### 已添加

- io: 添加 `tokio::io::Join` ([#6220])
- io: 为 `Empty` 实现 `AsyncWrite` ([#6235])
- net: 添加对匿名 unix 管道的支持 ([#6127])
- net: 添加 `UnixSocket` ([#6290])
- net: 在 `TcpSocket` 上暴露 keepalive 选项 ([#6311])
- sync: 添加 `{Receiver,UnboundedReceiver}::poll_recv_many` ([#6236])
- sync: 添加 `Sender::{try_,}reserve_many` ([#6205])
- sync: 添加 `watch::Receiver::mark_unchanged` ([#6252])
- task: 添加 `JoinSet::try_join_next` ([#6280])

### 已更改

- io: 使 `copy` 协作 ([#6265])
- io: 使 `repeat` 和 `sink` 协作 ([#6254])
- io: 简化空切片的检查 ([#6293])
- process: 在 Linux 上可用时使用 pidfd ([#6152])
- sync: 在广播通道 future 中使用 AtomicBool ([#6298])

### 文档

- io: 阐明 `clear_ready` 文档 ([#6304])
- net: 记录 `TcpSocket` 上的 `*Fd` trait 仅限 unix ([#6294])
- sync: 记录 `tokio::sync::Mutex` 的 FIFO 行为 ([#6279])
- chore: 排版改进 ([#6262])
- runtime: 移除过时的注释 ([#6303])
- task: 修复拼写错误 ([#6261])

[#6127]: https://github.com/tokio-rs/tokio/pull/6127
[#6152]: https://github.com/tokio-rs/tokio/pull/6152
[#6205]: https://github.com/tokio-rs/tokio/pull/6205
[#6220]: https://github.com/tokio-rs/tokio/pull/6220
[#6235]: https://github.com/tokio-rs/tokio/pull/6235
[#6236]: https://github.com/tokio-rs/tokio/pull/6236
[#6252]: https://github.com/tokio-rs/tokio/pull/6252
[#6254]: https://github.com/tokio-rs/tokio/pull/6254
[#6261]: https://github.com/tokio-rs/tokio/pull/6261
[#6262]: https://github.com/tokio-rs/tokio/pull/6262
[#6265]: https://github.com/tokio-rs/tokio/pull/6265
[#6279]: https://github.com/tokio-rs/tokio/pull/6279
[#6280]: https://github.com/tokio-rs/tokio/pull/6280
[#6290]: https://github.com/tokio-rs/tokio/pull/6290
[#6293]: https://github.com/tokio-rs/tokio/pull/6293
[#6294]: https://github.com/tokio-rs/tokio/pull/6294
[#6298]: https://github.com/tokio-rs/tokio/pull/6298
[#6303]: https://github.com/tokio-rs/tokio/pull/6303
[#6304]: https://github.com/tokio-rs/tokio/pull/6304
[#6311]: https://github.com/tokio-rs/tokio/pull/6311

## 1.35.1 (2023年12月19日)

这是已向后移植到 1.25.3 的更改的前向部分。

### 已修复

- io: 为 `tokio::runtime::io::registration::async_io` 添加预算 ([#6221])

[#6221]: https://github.com/tokio-rs/tokio/pull/6221

## 1.35.0 (2023年12月8日)

### 已添加

- net: 添加对 Apple watchOS 的支持 ([#6176])

### 已更改

- io: 从 `AsyncReadExt.read_buf` 中删除 `Sized` 要求 ([#6169])
- runtime: 使 `Runtime` unwind 安全 ([#6189])
- runtime: 减少任务生成中的锁争用 ([#6001])
- tokio: 将 nix 依赖更新至 0.27.1 ([#6190])

### 已修复

- chore: 使 `--cfg docsrs` 在没有 net 功能的情况下工作 ([#6166])
- chore: 在 miri 上为 `unsync_load` 使用宽松加载 ([#6179])
- runtime: 处理唤醒时缺失的上下文 ([#6148])
- taskdump: 修复 taskdump cargo 配置示例 ([#6150])
- taskdump: 在 taskdump 期间跳过已通知的任务 ([#6194])
- tracing: 避免使用当前父级创建资源跨度，而是使用 None 父级 ([#6107])
- tracing: 使任务跨度成为显式根 ([#6158])

### 文档

- io: 在 `AsyncWriteExt` 示例中刷新 ([#6149])
- runtime: 记录公平性保证和当前行为 ([#6145])
- task: 记录 `LocalSet::run_until` 的取消安全性 ([#6147])

[#6001]: https://github.com/tokio-rs/tokio/pull/6001
[#6107]: https://github.com/tokio-rs/tokio/pull/6107
[#6145]: https://github.com/tokio-rs/tokio/pull/6145
[#6147]: https://github.com/tokio-rs/tokio/pull/6147
[#6148]: https://github.com/tokio-rs/tokio/pull/6148
[#6149]: https://github.com/tokio-rs/tokio/pull/6149
[#6150]: https://github.com/tokio-rs/tokio/pull/6150
[#6158]: https://github.com/tokio-rs/tokio/pull/6158
[#6166]: https://github.com/tokio-rs/tokio/pull/6166
[#6169]: https://github.com/tokio-rs/tokio/pull/6169
[#6176]: https://github.com/tokio-rs/tokio/pull/6176
[#6179]: https://github.com/tokio-rs/tokio/pull/6179
[#6189]: https://github.com/tokio-rs/tokio/pull/6189
[#6190]: https://github.com/tokio-rs/tokio/pull/6190
[#6194]: https://github.com/tokio-rs/tokio/pull/6194

## 1.34.0 (2023年11月19日)

### 已修复

- io: 允许在 io 驱动程序关闭后调用 `clear_readiness` ([#6067])
- io: 修复 `take` 中的整数溢出 ([#6080])
- io: 修复 I/O 资源挂起问题 ([#6134])
- sync: 修复 `broadcast::channel` 链接 ([#6100])

### 已更改

- macros: 在 `tokio::test` 宏中使用 `::core` 限定的导入，而不是 `::std` ([#5973])

### 已添加

- fs: 在 `fs::read_dir` 中更新 cfg attr 以包含 `aix` ([#6075])
- sync: 添加 `mpsc::Receiver::recv_many` ([#6010])
- tokio: 添加了 vita 目标支持 ([#6094])

[#5973]: https://github.com/tokio-rs/tokio/pull/5973
[#6010]: https://github.com/tokio-rs/tokio/pull/6010
[#6067]: https://github.com/tokio-rs/tokio/pull/6067
[#6075]: https://github.com/tokio-rs/tokio/pull/6075
[#6080]: https://github.com/tokio-rs/tokio/pull/6080
[#6094]: https://github.com/tokio-rs/tokio/pull/6094
[#6100]: https://github.com/tokio-rs/tokio/pull/6100
[#6134]: https://github.com/tokio-rs/tokio/pull/6134

## 1.33.0 (2023年10月9日)

### 已修复

- io: 用 `#[must_use]` 标记 `Interest::add` ([#6037])
- runtime: 修复 RISC-V 的缓存行大小 ([#5994])
- sync: 防止 `watch::Receiver::wait_for` 中的锁中毒 ([#6021])
- task: 修复 `spawn_local` 源位置 ([#5984])

### 已更改

- sync: 在 `watch` 中使用 Acquire/Release 顺序代替 SeqCst ([#6018])

### 已添加

- fs: 为 `tokio::fs::File` 添加向量化写入 ([#5958])
- io: 添加 `Interest::remove` 方法 ([#5906])
- io: 为 `DuplexStream` 添加向量化写入 ([#5985])
- net: 添加对 Apple tvOS 的支持 ([#6045])
- sync: 为 `{MutexGuard,OwnedMutexGuard}::map` 添加 `?Sized` 绑定 ([#5997])
- sync: 添加 `watch::Receiver::mark_unseen` ([#5962], [#6014], [#6017])
- sync: 添加 `watch::Sender::new` ([#5998])
- sync: 添加 const fn `OnceCell::from_value` ([#5903])

### 已移除

- 移除未使用的 `stats` 功能 ([#5952])

### 文档

- 添加代码示例中缺失的反引号 ([#5938], [#6056])
- 修复拼写错误 ([#5988], [#6030])
- process: 记录 `Child::wait` 是取消安全的 ([#5977])
- sync: 为 `Semaphore` 添加示例 ([#5939], [#5956], [#5978], [#6031], [#6032], [#6050])
- sync: 记录 `broadcast` 的容量是下限 ([#6042])
- sync: 记录 `const_new` 未被检测 ([#6002])
- sync: 改进 `mpsc::Sender::send` 的取消安全文档 ([#5947])
- sync: 改进 `watch` 通道的文档 ([#5954])
- taskdump: 在 docs.rs 上渲染 taskdump 文档 ([#5972])

### 非稳定版

- taskdump: 修复潜在的死锁 ([#6036])

[#5903]: https://github.com/tokio-rs/tokio/pull/5903
[#5906]: https://github.com/tokio-rs/tokio/pull/5906
[#5938]: https://github.com/tokio-rs/tokio/pull/5938
[#5939]: https://github.com/tokio-rs/tokio/pull/5939
[#5947]: https://github.com/tokio-rs/tokio/pull/5947
[#5952]: https://github.com/tokio-rs/tokio/pull/5952
[#5954]: https://github.com/tokio-rs/tokio/pull/5954
[#5956]: https://github.com/tokio-rs/tokio/pull/5956
[#5958]: https://github.com/tokio-rs/tokio/pull/5958
[#5962]: https://github.com/tokio-rs/tokio/pull/5962
[#5972]: https://github.com/tokio-rs/tokio/pull/5972
[#5977]: https://github.com/tokio-rs/tokio/pull/5977
[#5978]: https://github.com/tokio-rs/tokio/pull/5978
[#5984]: https://github.com/tokio-rs/tokio/pull/5984
[#5985]: https://github.com/tokio-rs/tokio/pull/5985
[#5988]: https://github.com/tokio-rs/tokio/pull/5988
[#5994]: https://github.com/tokio-rs/tokio/pull/5994
[#5997]: https://github.com/tokio-rs/tokio/pull/5997
[#5998]: https://github.com/tokio-rs/tokio/pull/5998
[#6002]: https://github.com/tokio-rs/tokio/pull/6002
[#6014]: https://github.com/tokio-rs/tokio/pull/6014
[#6017]: https://github.com/tokio-rs/tokio/pull/6017
[#6018]: https://github.com/tokio-rs/tokio/pull/6018
[#6021]: https://github.com/tokio-rs/tokio/pull/6021
[#6030]: https://github.com/tokio-rs/tokio/pull/6030
[#6031]: https://github.com/tokio-rs/tokio/pull/6031
[#6032]: https://github.com/tokio-rs/tokio/pull/6032
[#6036]: https://github.com/tokio-rs/tokio/pull/6036
[#6037]: https://github.com/tokio-rs/tokio/pull/6037
[#6042]: https://github.com/tokio-rs/tokio/pull/6042
[#6045]: https://github.com/tokio-rs/tokio/pull/6045
[#6050]: https://github.com/tokio-rs/tokio/pull/6050
[#6056]: https://github.com/tokio-rs/tokio/pull/6056

## 1.32.1 (2023年12月19日)

这是已向后移植到 1.25.3 的更改的前向部分。

### 已修复

- io: 为 `tokio::runtime::io::registration::async_io` 添加预算 ([#6221])

[#6221]: https://github.com/tokio-rs/tokio/pull/6221

## 1.32.0 (2023年8月16日)

### 已修复

- sync: 修复 `broadcast::Receiver` 中潜在的二次行为 ([#5925])

### 已添加

- process: 稳定 `Command::raw_arg` ([#5930])
- io: 启用等待错误就绪 ([#5781])

### 非稳定版

- rt(alt): 随着核心数量的增长，提高 alt 运行时的可扩展性 ([#5935])

[#5781]: https://github.com/tokio-rs/tokio/pull/5781
[#5925]: https://github.com/tokio-rs/tokio/pull/5925
[#5930]: https://github.com/tokio-rs/tokio/pull/5930
[#5935]: https://github.com/tokio-rs/tokio/pull/5935

## 1.31.0 (2023年8月10日)

### 已修复

* io: 委托 `WriteHalf::poll_write_vectored` ([#5914])

### 非稳定版

* rt(alt): 修复不稳定的下一代调度器原型中的内存泄漏 ([#5911])
* rt: 暴露平均任务轮询时间指标 ([#5927])

[#5911]: https://github.com/tokio-rs/tokio/pull/5911
[#5914]: https://github.com/tokio-rs/tokio/pull/5914
[#5927]: https://github.com/tokio-rs/tokio/pull/5927

## 1.30.0 (2023年8月9日)

此版本将 Tokio 的 MSRV 提升至 1.63。 ([#5887])

### 已更改

- tokio: 减少 LLVM 代码生成 ([#5859])
- io: 支持 `--cfg mio_unsupported_force_poll_poll` 标志 ([#5881])
- sync: 使 `const_new` 方法始终可用 ([#5885])
- sync: 避免 mpsc 通道中的伪共享 ([#5829])
- rt: 从注入队列中至少弹出一个任务 ([#5908])

### 已添加

- sync: 添加 `broadcast::Sender::new` ([#5824])
- net: 为 espidf 实现 `UCred` ([#5868])
- fs: 添加 `File::options()` ([#5869])
- time: 为 `Interval` 实现额外的重置变体 ([#5878])
- process: 添加 `{ChildStd*}::into_owned_{fd, handle}` ([#5899])

### 已移除

- tokio: 移除了未使用的 `tokio_*` cfgs ([#5890])
- 移除构建脚本以加快编译速度 ([#5887])

### 文档

- sync: 在 `broadcast::send` 的文档中提及滞后 ([#5820])
- runtime: 扩展共享运行时文档 ([#5858])
- io: 在 `AsyncReadExt::read_exact` 的示例中使用 vec ([#5863])
- time: 在文档中将 `Sleep` 标记为 `!Unpin` ([#5916])
- process: 修复 `raw_arg` 未在文档中显示的问题 ([#5865])

### 非稳定版

- rt: 添加运行时 ID ([#5864])
- rt: 新线程运行时的初始实现 ([#5823])

[#5820]: https://github.com/tokio-rs/tokio/pull/5820
[#5823]: https://github.com/tokio-rs/tokio/pull/5823
[#5824]: https://github.com/tokio-rs/tokio/pull/5824
[#5829]: https://github.com/tokio-rs/tokio/pull/5829
[#5858]: https://github.com/tokio-rs/tokio/pull/5858
[#5859]: https://github.com/tokio-rs/tokio/pull/5859
[#5863]: https://github.com/tokio-rs/tokio/pull/5863
[#5864]: https://github.com/tokio-rs/tokio/pull/5864
[#5865]: https://github.com/tokio-rs/tokio/pull/5865
[#5868]: https://github.com/tokio-rs/tokio/pull/5868
[#5869]: https://github.com/tokio-rs/tokio/pull/5869
[#5878]: https://github.com/tokio-rs/tokio/pull/5878
[#5881]: https://github.com/tokio-rs/tokio/pull/5881
[#5885]: https://github.com/tokio-rs/tokio/pull/5885
[#5887]: https://github.com/tokio-rs/tokio/pull/5887
[#5890]: https://github.com/tokio-rs/tokio/pull/5890
[#5899]: https://github.com/tokio-rs/tokio/pull/5899
[#5908]: https://github.com/tokio-rs/tokio/pull/5908
[#5916]: https://github.com/tokio-rs/tokio/pull/5916

## 1.29.1 (2023年6月29日)

### 已修复

- rt: 修复在两个 `block_in_place` 之间嵌套一个 `block_on` 的问题 ([#5837])

[#5837]: https://github.com/tokio-rs/tokio/pull/5837

## 1.29.0 (2023年6月27日)

技术上是破坏性变更，`Send` 的实现从 `runtime::EnterGuard` 中移除。此更改修复了一个 bug，并且不应影响大多数用户。

### 破坏性变更

- rt: `EnterGuard` 不应是 `Send` ([#5766])

### 已修复

- fs: 减少 `fs::read_dir` 中的阻塞操作 ([#5653])
- rt: 修复可能的饥饿问题 ([#5686], [#5712])
- rt: 修复 `JoinSet` 中的堆叠借用问题 ([#5693])
- rt: 如果 `EnterGuard` 以不正确的顺序被丢弃则 panic ([#5772])
- time: 不要溢出到信号值 ([#5710])
- fs: 在克隆 `File` 之前等待进行中的操作 ([#5803])

### 已更改

- rt: 减少轮询从运行时外部调度的任务的时间 ([#5705], [#5720])

### 已添加

- net: 为 unix 套接字添加 uds 文档别名 ([#5659])
- rt: 添加任务数量的指标 ([#5628])
- sync: 为通道错误实现更多 trait ([#5666])
- net: 在 TcpSocket 上添加 nodelay 方法 ([#5672])
- sync: 添加 `broadcast::Receiver::blocking_recv` ([#5690])
- process: 向 `Command` 添加 `raw_arg` 方法 ([#5704])
- io: 支持 PRIORITY epoll 事件 ([#5566])
- task: 添加 `JoinSet::poll_join_next` ([#5721])
- net: 添加对 Redox OS 的支持 ([#5790])


### 非稳定版

- rt: 添加转储任务回溯的能力 ([#5608], [#5676], [#5708], [#5717])
- rt: 使用直方图来测量任务轮询时间 ([#5685])

[#5566]: https://github.com/tokio-rs/tokio/pull/5566
[#5608]: https://github.com/tokio-rs/tokio/pull/5608
[#5628]: https://github.com/tokio-rs/tokio/pull/5628
[#5653]: https://github.com/tokio-rs/tokio/pull/5653
[#5659]: https://github.com/tokio-rs/tokio/pull/5659
[#5666]: https://github.com/tokio-rs/tokio/pull/5666
[#5672]: https://github.com/tokio-rs/tokio/pull/5672
[#5676]: https://github.com/tokio-rs/tokio/pull/5676
[#5685]: https://github.com/tokio-rs/tokio/pull/5685
[#5686]: https://github.com/tokio-rs/tokio/pull/5686
[#5690]: https://github.com/tokio-rs/tokio/pull/5690
[#5693]: https://github.com/tokio-rs/tokio/pull/5693
[#5704]: https://github.com/tokio-rs/tokio/pull/5704
[#5705]: https://github.com/tokio-rs/tokio/pull/5705
[#5708]: https://github.com/tokio-rs/tokio/pull/5708
[#5710]: https://github.com/tokio-rs/tokio/pull/5710
[#5712]: https://github.com/tokio-rs/tokio/pull/5712
[#5717]: https://github.com/tokio-rs/tokio/pull/5717
[#5720]: https://github.com/tokio-rs/tokio/pull/5720
[#5721]: https://github.com/tokio-rs/tokio/pull/5721
[#5766]: https://github.com/tokio-rs/tokio/pull/5766
[#5772]: https://github.com/tokio-rs/tokio/pull/5772
[#5790]: https://github.com/tokio-rs/tokio/pull/5790
[#5803]: https://github.com/tokio-rs/tokio/pull/5803

## 1.28.2 (2023年5月28日)

前向移植 1.18.6 的更改。

### 已修复

- deps: 禁用 mio 的默认功能 ([#5728])

[#5728]: https://github.com/tokio-rs/tokio/pull/5728

## 1.28.1 (2023年5月10日)

此版本修复了构建脚本中的一个错误，该错误导致在 Rust 1.63 上 `AsFd` 实现不可用。([#5677])

[#5677]: https://github.com/tokio-rs/tokio/pull/5677

## 1.28.0 (2023年4月25日)

### 已添加

- io: 添加 `AsyncFd::async_io` ([#5542])
- io: 为 ReadBuf 实现 BufMut ([#5590])
- net: 为 `UdpSocket` 和 `UnixDatagram` 添加 `recv_buf` ([#5583])
- sync: 添加 `OwnedSemaphorePermit::semaphore` ([#5618])
- sync: 为广播通道添加 `same_channel` ([#5607])
- sync: 添加 `watch::Receiver::wait_for` ([#5611])
- task: 添加 `JoinSet::spawn_blocking` 和 `JoinSet::spawn_blocking_on` ([#5612])

### 已更改

- deps: 将 windows-sys 更新至 0.48 ([#5591])
- io: 使 `read_to_end` 不会不必要地增长 ([#5610])
- macros: 使入口点更高效 ([#5621])
- sync: 改进 `RwLock` 的 Debug 实现 ([#5647])
- sync: 减少 `Notify` 中的争用 ([#5503])

### 已修复

- net: 支持在 AIX 上 `get_peer_cred` ([#5065])
- sync: 避免 `broadcast` 中自定义 waker 的死锁 ([#5578])

### 文档

- sync: 修复 `Semaphore::MAX_PERMITS` 中的拼写错误 ([#5645])
- sync: 修复 `tokio::sync::watch::Sender` 文档中的拼写错误 ([#5587])

[#5065]: https://github.com/tokio-rs/tokio/pull/5065
[#5503]: https://github.com/tokio-rs/tokio/pull/5503
[#5542]: https://github.com/tokio-rs/tokio/pull/5542
[#5578]: https://github.com/tokio-rs/tokio/pull/5578
[#5583]: https://github.com/tokio-rs/tokio/pull/5583
[#5587]: https://github.com/tokio-rs/tokio/pull/5587
[#5590]: https://github.com/tokio-rs/tokio/pull/5590
[#5591]: https://github.com/tokio-rs/tokio/pull/5591
[#5607]: https://github.com/tokio-rs/tokio/pull/5607
[#5610]: https://github.com/tokio-rs/tokio/pull/5610
[#5611]: https://github.com/tokio-rs/tokio/pull/5611
[#5612]: https://github.com/tokio-rs/tokio/pull/5612
[#5618]: https://github.com/tokio-rs/tokio/pull/5618
[#5621]: https://github.com/tokio-rs/tokio/pull/5621
[#5645]: https://github.com/tokio-rs/tokio/pull/5645
[#5647]: https://github.com/tokio-rs/tokio/pull/5647

## 1.27.0 (2023年3月27日)

此版本将 Tokio 的 MSRV 提升至 1.56。 ([#5559])

### 已添加

- io: 向套接字添加 `async_io` 辅助方法 ([#5512])
- io: 添加 `AsFd`/`AsHandle`/`AsSocket` 的实现 ([#5514], [#5540])
- net: 添加 `UdpSocket::peek_sender()` ([#5520])
- sync: 添加 `RwLockWriteGuard::{downgrade_map, try_downgrade_map}` ([#5527])
- task: 添加 `JoinHandle::abort_handle` ([#5543])

### 已更改

- io: 使用 `libc` 中的 `memchr` ([#5558])
- macros: 在 `#[tokio::main]` 中接受路径作为 crate 重命名 ([#5557])
- macros: 更新至 syn 2.0.0 ([#5572])
- time: 当 `Interval` 返回 `Ready` 时不注册唤醒 ([#5553])

### 已修复

- fs: 在 `ReadDir` 中融合 std 迭代器 ([#5555])
- tracing: 修复 `spawn_blocking` 位置字段 ([#5573])
- time: 清理 `Wheel::poll()` 中的冗余检查 ([#5574])

### 文档

- macros: 定义取消安全性 ([#5525])
- io: 在 `tokio::io::copy[_buf]` 的文档中添加细节 ([#5575])
- io: 在模块文档中引用 `ReaderStream` 和 `StreamReader` ([#5576])

[#5512]: https://github.com/tokio-rs/tokio/pull/5512
[#5514]: https://github.com/tokio-rs/tokio/pull/5514
[#5520]: https://github.com/tokio-rs/tokio/pull/5520
[#5525]: https://github.com/tokio-rs/tokio/pull/5525
[#5527]: https://github.com/tokio-rs/tokio/pull/5527
[#5540]: https://github.com/tokio-rs/tokio/pull/5540
[#5543]: https://github.com/tokio-rs/tokio/pull/5543
[#5553]: https://github.com/tokio-rs/tokio/pull/5553
[#5555]: https://github.com/tokio-rs/tokio/pull/5555
[#5557]: https://github.com/tokio-rs/tokio/pull/5557
[#5558]: https://github.com/tokio-rs/tokio/pull/5558
[#5559]: https://github.com/tokio-rs/tokio/pull/5559
[#5572]: https://github.com/tokio-rs/tokio/pull/5572
[#5573]: https://github.com/tokio-rs/tokio/pull/5573
[#5574]: https://github.com/tokio-rs/tokio/pull/5574
[#5575]: https://github.com/tokio-rs/tokio/pull/5575
[#5576]: https://github.com/tokio-rs/tokio/pull/5576

## 1.26.0 (2023年3月1日)

### 已修复

- macros: 修复空的 `join!` 和 `try_join!` ([#5504])
- sync: 不要在互斥锁守卫中泄漏 tracing spans ([#5469])
- sync: 在 Notify 中解锁互斥锁后丢弃 wakers ([#5471])
- sync: 在信号量中的锁外丢弃 wakers ([#5475])

### 已添加

- fs: 添加 `fs::try_exists` ([#4299])
- net: 添加命名 unix 管道的类型 ([#5351])
- sync: 添加 `MappedOwnedMutexGuard` ([#5474])

### 已更改

- chore: 将 windows-sys 更新至 0.45 ([#5386])
- net: 为命名管道使用消息读取模式 ([#5350])
- sync: 用 `#[clippy::has_significant_drop]` 标记锁守卫 ([#5422])
- sync: 减少 watch 通道中的争用 ([#5464])
- time: 移除计时器条目中的缓存填充 ([#5468])
- time: 改进 `Instant::now()` 与 test-util 的性能 ([#5513])

### 内部变更

- io: 在 `copy_bidirectional` 中使用 `poll_fn` ([#5486])
- net: 重构命名管道构建器以不使用位域 ([#5477])
- rt: 从 Clock 中移除 Arc ([#5434])
- sync: 使 `notify_waiters` 调用原子化 ([#5458])
- time: 不要在 sleep 条目中存储两次截止时间 ([#5410])

### 非稳定版

- metrics: 添加一个新的用于预算耗尽让步的指标 ([#5517])

### 文档

- io: 改进 AsyncFd 示例 ([#5481])
- runtime: 记录 main future 的性质 ([#5494])
- runtime: 移除文档中多余的句点 ([#5511])
- signal: 更新信号的文档 ([#5459])
- sync: 为 `blocking_*` 方法添加文档别名 ([#5448])
- sync: 修复 broadcast 中 Send/Sync 绑定的文档 ([#5480])
- sync: 记录通道的丢弃行为 ([#5497])
- task: 阐明在运行时关闭期间生成的任务会发生什么 ([#5394])
- task: 阐明 `process::Command` 文档 ([#5413])
- task: 修复 'unsend' 的措辞 ([#5452])
- time: 记录超时的立即完成保证 ([#5509])
- tokio: 记录支持的平台 ([#5483])

[#4299]: https://github.com/tokio-rs/tokio/pull/4299
[#5350]: https://github.com/tokio-rs/tokio/pull/5350
[#5351]: https://github.com/tokio-rs/tokio/pull/5351
[#5386]: https://github.com/tokio-rs/tokio/pull/5386
[#5394]: https://github.com/tokio-rs/tokio/pull/5394
[#5410]: https://github.com/tokio-rs/tokio/pull/5410
[#5413]: https://github.com/tokio-rs/tokio/pull/5413
[#5422]: https://github.com/tokio-rs/tokio/pull/5422
[#5434]: https://github.com/tokio-rs/tokio/pull/5434
[#5448]: https://github.com/tokio-rs/tokio/pull/5448
[#5452]: https://github.com/tokio-rs/tokio/pull/5452
[#5458]: https://github.com/tokio-rs/tokio/pull/5458
[#5459]: https://github.com/tokio-rs/tokio/pull/5459
[#5464]: https://github.com/tokio-rs/tokio/pull/5464
[#5468]: https://github.com/tokio-rs/tokio/pull/5468
[#5469]: https://github.com/tokio-rs/tokio/pull/5469
[#5471]: https://github.com/tokio-rs/tokio/pull/5471
[#5474]: https://github.com/tokio-rs/tokio/pull/5474
[#5475]: https://github.com/tokio-rs/tokio/pull/5475
[#5477]: https://github.com/tokio-rs/tokio/pull/5477
[#5480]: https://github.com/tokio-rs/tokio/pull/5480
[#5481]: https://github.com/tokio-rs/tokio/pull/5481
[#5483]: https://github.com/tokio-rs/tokio/pull/5483
[#5486]: https://github.com/tokio-rs/tokio/pull/5486
[#5494]: https://github.com/tokio-rs/tokio/pull/5494
[#5497]: https://github.com/tokio-rs/tokio/pull/5497
[#5504]: https://github.com/tokio-rs/tokio/pull/5504
[#5509]: https://github.com/tokio-rs/tokio/pull/5509
[#5511]: https://github.com/tokio-rs/tokio/pull/5511
[#5513]: https://github.com/tokio-rs/tokio/pull/5513
[#5517]: https://github.com/tokio-rs/tokio/pull/5517

## 1.25.3 (2023年12月17日)

### 已修复
- io: 为 `tokio::runtime::io::registration::async_io` 添加预算 ([#6221])

[#6221]: https://github.com/tokio-rs/tokio/pull/6221

## 1.25.2 (2023年9月22日)

前向移植 1.20.6 的更改。

### 已更改

- io: 使用 `libc` 中的 `memchr` ([#5960])

[#5960]: https://github.com/tokio-rs/tokio/pull/5960

## 1.25.1 (2023年5月28日)

前向移植 1.18.6 的更改。

### 已修复

- deps: 禁用 mio 的默认功能 ([#5728])

[#5728]: https://github.com/tokio-rs/tokio/pull/5728

## 1.25.0 (2023年1月28日)

### 已修复

- rt: 修复运行时指标报告 ([#5330])

### 已添加

- sync: 添加 `broadcast::Sender::len` ([#5343])

### 已更改

- fs: 将最大读取缓冲区大小增加到 2MiB ([#5397])

[#5330]: https://github.com/tokio-rs/tokio/pull/5330
[#5343]: https://github.com/tokio-rs/tokio/pull/5343
[#5397]: https://github.com/tokio-rs/tokio/pull/5397

## 1.24.2 (2023年1月17日)

前向移植 1.18.5 的更改。

### 已修复

- io: 修复 `ReadHalf::unsplit` 中的不健全性 ([#5375])

[#5375]: https://github.com/tokio-rs/tokio/pull/5375

## 1.24.1 (2022年1月6日)

此版本修复了在使用早于 1.63 的 rustc 时，在没有 `AtomicU64` 的目标上编译失败的问题。([#5356])

[#5356]: https://github.com/tokio-rs/tokio/pull/5356

## 1.24.0 (2022年1月5日)

### 已修复
 - rt: 改进原生 `AtomicU64` 支持检测 ([#5284])

### 已添加
 - rt: 添加配置选项，用于设置每次 tick 从操作系统轮询的最大 I/O 事件数 ([#5186])
 - rt: 添加环境变量，用于配置每个运行时实例的默认工作线程数 ([#4250])

### 已更改
 - sync: 减少 MPSC 通道堆栈使用 ([#5294])
 - io: 减少 I/O 操作中的锁争用 ([#5300])
 - fs: 通过分块操作加快 `read_dir()` 的速度 ([#5309])
 - rt: 使用内部 `ThreadId` 实现 ([#5329])
 - test: 当 `spawn_blocking` 任务正在运行时，不要自动推进时间 ([#5115])

[#4250]: https://github.com/tokio-rs/tokio/pull/4250
[#5115]: https://github.com/tokio-rs/tokio/pull/5115
[#5186]: https://github.com/tokio-rs/tokio/pull/5186
[#5284]: https://github.com/tokio-rs/tokio/pull/5284
[#5294]: https://github.com/tokio-rs/tokio/pull/5294
[#5300]: https://github.com/tokio-rs/tokio/pull/5300
[#5309]: https://github.com/tokio-rs/tokio/pull/5309
[#5329]: https://github.com/tokio-rs/tokio/pull/5329

## 1.23.1 (2022年1月4日)

此版本前向移植了 1.18.4 的更改。

### 已修复

- net: 修复 Windows 命名管道服务器构建器，以在切换管道模式时保持选项 ([#5336])。

[#5336]: https://github.com/tokio-rs/tokio/pull/5336

## 1.23.0 (2022年12月5日)

### 已修复

 - net: 修复 Windows 命名管道连接 ([#5208])
 - io: 支持 `ChildStdin` 的向量化写入 ([#5216])
 - io: 修复 `async fn ready()` 对特定于操作系统的事件的误报 ([#5231])

 ### 已更改
 - runtime: `yield_now` 将任务推迟到驱动程序轮询之后 ([#5223])
 - runtime: 减少每个生成任务所需的代码生成量 ([#5213])
 - windows: 将 `winapi` 依赖项替换为 `windows-sys` ([#5204])

[#5204]: https://github.com/tokio-rs/tokio/pull/5204
[#5208]: https://github.com/tokio-rs/tokio/pull/5208
[#5213]: https://github.com/tokio-rs/tokio/pull/5213
[#5216]: https://github.com/tokio-rs/tokio/pull/5216
[#5223]: https://github.com/tokio-rs/tokio/pull/5223
[#5231]: https://github.com/tokio-rs/tokio/pull/5231

## 1.22.0 (2022年11月17日)

### 已添加
 - runtime: 添加 `Handle::runtime_flavor` ([#5138])
 - sync: 添加 `Mutex::blocking_lock_owned` ([#5130])
 - sync: 添加 `Semaphore::MAX_PERMITS` ([#5144])
 - sync: 将 `merge()` 添加到信号量许可 ([#4948])
 - sync: 添加 `mpsc::WeakUnboundedSender` ([#5189])

### 已添加 (非稳定版)

 - process: 添加 `Command::process_group` ([#5114])
 - runtime: 导出有关阻塞线程池的指标 ([#5161])
 - task: 添加 `task::id()` 和 `task::try_id()` ([#5171])

### 已修复
 - macros: 不要在宏中获取 future 的所有权 ([#5087])
 - runtime: 修复 `LocalOwnedTasks` 中的堆叠借用冲突 ([#5099])
 - runtime: 在可能的情况下，使用 32 位队列索引来缓解 ABA 问题 ([#5042])
 - task: 当同一线程唤醒本地任务时，将其唤醒到本地队列 ([#5095])
 - time: 在非法调用 `mark_pending` 时，在发布模式下 panic ([#5093])
 - runtime: 修复 expect 消息中的拼写错误 ([#5169])
 - runtime: 修复原子类型上的 `unsync_load` ([#5175])
 - task: 详细说明任务释放中的安全注释 ([#5172])
 - runtime: 修复线程本地中 `LocalSet` 的 drop ([#5179])
 - net: 从公共 API 中移除 libc 类型泄漏 ([#5191])
 - runtime: 更新 `CachePadded` 的对齐方式 ([#5106])

### 已更改
 - io: 使 `tokio::io::copy` 在写入器停顿时继续填充缓冲区 ([#5066])
 - runtime: 从 `LocalSet::run_until` 中移除 `coop::budget` ([#5155])
 - sync: 使 `Notify` panic 安全 ([#5154])

### 文档
 - io: 修复 `write_i8` 的文档以使用有符号整数 ([#5040])
 - net: 修复 TCP 和 UDP `set_tos` 方法的文档拼写错误 ([#5073])
 - net: 修复 `UdpSocket::recv` 文档中的函数名 ([#5150])
 - sync: `RwLock::try_write` 的 `TryLockError` 中的拼写错误 ([#5160])
 - task: 记录生成的任务会立即执行 ([#5117])
 - time: 记录 `timeout` 的返回类型 ([#5118])
 - time: 记录 `timeout` 仅在轮询前检查 ([#5126])
 - sync: 在文档中指定 `oneshot::Receiver` 的返回类型 ([#5198])

[#4948]: https://github.com/tokio-rs/tokio/pull/4948
[#5040]: https://github.com/tokio-rs/tokio/pull/5040
[#5042]: https://github.com/tokio-rs/tokio/pull/5042
[#5066]: https://github.com/tokio-rs/tokio/pull/5066
[#5073]: https://github.com/tokio-rs/tokio/pull/5073
[#5087]: https://github.com/tokio-rs/tokio/pull/5087
[#5093]: https://github.com/tokio-rs/tokio/pull/5093
[#5095]: https://github.com/tokio-rs/tokio/pull/5095
[#5099]: https://github.com/tokio-rs/tokio/pull/5099
[#5106]: https://github.com/tokio-rs/tokio/pull/5106
[#5114]: https://github.com/tokio-rs/tokio/pull/5114
[#5117]: https://github.com/tokio-rs/tokio/pull/5117
[#5118]: https://github.com/tokio-rs/tokio/pull/5118
[#5126]: https://github.com/tokio-rs/tokio/pull/5126
[#5130]: https://github.com/tokio-rs/tokio/pull/5130
[#5138]: https://github.com/tokio-rs/tokio/pull/5138
[#5144]: https://github.com/tokio-rs/tokio/pull/5144
[#5150]: https://github.com/tokio-rs/tokio/pull/5150
[#5154]: https://github.com/tokio-rs/tokio/pull/5154
[#5155]: https://github.com/tokio-rs/tokio/pull/5155
[#5160]: https://github.com/tokio-rs/tokio/pull/5160
[#5161]: https://github.com/tokio-rs/tokio/pull/5161
[#5169]: https://github.com/tokio-rs/tokio/pull/5169
[#5171]: https://github.com/tokio-rs/tokio/pull/5171
[#5172]: https://github.com/tokio-rs/tokio/pull/5172
[#5175]: https://github.com/tokio-rs/tokio/pull/5175
[#5179]: https://github.com/tokio-rs/tokio/pull/5179
[#5189]: https://github.com/tokio-rs/tokio/pull/5189
[#5191]: https://github.com/tokio-rs/tokio/pull/5191
[#5198]: https://github.com/tokio-rs/tokio/pull/5198

## 1.21.2 (2022年9月27日)

此版本移除了对 `once_cell` crate 的依赖，以恢复 1.21.x 的 MSRV，这是发布时的最新次要版本。 ([#5048])

[#5048]: https://github.com/tokio-rs/tokio/pull/5048

## 1.21.1 (2022年9月13日)

### 已修复

- net: 修复 socket2 的依赖解析 ([#5000])
- task: 在 `LocalSet` Drop 中忽略设置 TLS 的失败 ([#4976])

[#4976]: https://github.com/tokio-rs/tokio/pull/4976
[#5000]: https://github.com/tokio-rs/tokio/pull/5000

## 1.21.0 (2022年9月2日)

此版本是 Tokio 第一个有意支持 WASM 的版本。`sync,macros,io-util,rt,time` 功能在 WASM 上已稳定。此外，wasm32-wasi 目标对 `net` 功能提供了不稳定的支持。

### 已添加

- net: 为 TCP/UDP 套接字添加 `device` 和 `bind_device` 方法 ([#4882])
- net: 为 TCP 和 UDP 套接字添加 `tos` 和 `set_tos` 方法 ([#4877])
- net: 为命名管道 `ServerOptions` 添加安全标志 ([#4845])
- signal: 添加更多 windows 信号处理程序 ([#4924])
- sync: 添加 `mpsc::Sender::max_capacity` 方法 ([#4904])
- sync: 实现 `mpsc::Sender` 的 Weak 版本 ([#4595])
- task: 添加 `LocalSet::enter` ([#4765])
- task: 稳定 `JoinSet` 和 `AbortHandle` ([#4920])
- tokio: 为公共 API 添加 `track_caller` ([#4805], [#4848], [#4852])
- wasm: 初步支持 `wasm32-wasi` 目标 ([#4716])

### 已修复

- miri: 通过在 `linked_list::Link` impls 中避免临时引用来提高 miri 兼容性 ([#4841])
- signal: 不要在信号管道上注册写兴趣 ([#4898])
- sync: 为锁守卫添加 `#[must_use]` ([#4886])
- sync: 修复在已关闭并重新打开的广播通道上调用 `recv` 时的挂起问题 ([#4867])
- task: 在任务本地上广播属性 ([#4837])

### 已更改

- fs: 将 `File::start_seek` 中的 panic 改为错误 ([#4897])
- io: 减少 `poll_read` 中的系统调用 ([#4840])
- process: 对子进程 stdio I/O 使用阻塞线程池 ([#4824])
- signal: 使 `SignalKind` 方法为 const ([#4956])

### 文档

- chore: 修复拼写错误和语法 ([#4858], [#4894], [#4928])
- io: 修复 `AsyncSeekExt::rewind` 文档中的拼写错误 ([#4893])
- net: 为 `try_read()` 的零长度缓冲区添加文档 ([#4937])
- runtime: 移除 `Builder::worker_threads` 的不正确的 panic 部分 ([#4849])
- sync: 改进了 `watch::Sender::send` 的文档 ([#4959])
- task: 为 `JoinHandle` 添加取消安全文档 ([#4901])
- task: 扩展关于取消 `spawn_blocking` 的内容 ([#4811])
- time: 阐明 `Interval::tick` 的第一次 tick 会立即发生 ([#4951])

### 非稳定版

- rt: 添加禁用 LIFO slot 的不稳定选项 ([#4936])
- task: 修复 `Builder::spawn_on` 中的不正确签名 ([#4953])
- task: 使 `task::Builder::spawn*` 方法可失败 ([#4823])

[#4595]: https://github.com/tokio-rs/tokio/pull/4595
[#4716]: https://github.com/tokio-rs/tokio/pull/4716
[#4765]: https://github.com/tokio-rs/tokio/pull/4765
[#4805]: https://github.com/tokio-rs/tokio/pull/4805
[#4811]: https://github.com/tokio-rs/tokio/pull/4811
[#4823]: https://github.com/tokio-rs/tokio/pull/4823
[#4824]: https://github.com/tokio-rs/tokio/pull/4824
[#4837]: https://github.com/tokio-rs/tokio/pull/4837
[#4840]: https://github.com/tokio-rs/tokio/pull/4840
[#4841]: https://github.com/tokio-rs/tokio/pull/4841
[#4845]: https://github.com/tokio-rs/tokio/pull/4845
[#4848]: https://github.com/tokio-rs/tokio/pull/4848
[#4849]: https://github.com/tokio-rs/tokio/pull/4849
[#4852]: https://github.com/tokio-rs/tokio/pull/4852
[#4858]: https://github.com/tokio-rs/tokio/pull/4858
[#4867]: https://github.com/tokio-rs/tokio/pull/4867
[#4877]: https://github.com/tokio-rs/tokio/pull/4877
[#4882]: https://github.com/tokio-rs/tokio/pull/4882
[#4886]: https://github.com/tokio-rs/tokio/pull/4886
[#4893]: https://github.com/tokio-rs/tokio/pull/4893
[#4894]: https://github.com/tokio-rs/tokio/pull/4894
[#4897]: https://github.com/tokio-rs/tokio/pull/4897
[#4898]: https://github.com/tokio-rs/tokio/pull/4898
[#4901]: https://github.com/tokio-rs/tokio/pull/4901
[#4904]: https://github.com/tokio-rs/tokio/pull/4904
[#4920]: https://github.com/tokio-rs/tokio/pull/4920
[#4924]: https://github.com/tokio-rs/tokio/pull/4924
[#4928]: https://github.com/tokio-rs/tokio/pull/4928
[#4935]: https://github.com/tokio-rs/tokio/pull/4935
[#4936]: https://github.com/tokio-rs/tokio/pull/4936
[#4937]: https://github.com/tokio-rs/tokio/pull/4937
[#4951]: https://github.com/tokio-rs/tokio/pull/4951
[#4953]: https://github.com/tokio-rs/tokio/pull/4953
[#4956]: https://github.com/tokio-rs/tokio/pull/4956
[#4959]: https://github.com/tokio-rs/tokio/pull/4959

## 1.20.6 (2023年9月22日)

这是 1.27.0 中一项更改的向后移植。

### 已更改

- io: 使用 `libc` 中的 `memchr` ([#5960])

[#5960]: https://github.com/tokio-rs/tokio/pull/5960

## 1.20.5 (2023年5月28日)

前向移植 1.18.6 的更改。

### 已修复

- deps: 禁用 mio 的默认功能 ([#5728])

[#5728]: https://github.com/tokio-rs/tokio/pull/5728

## 1.20.4 (2023年1月17日)

前向移植 1.18.5 的更改。

### 已修复

- io: 修复 `ReadHalf::unsplit` 中的不健全性 ([#5375])

[#5375]: https://github.com/tokio-rs/tokio/pull/5375

## 1.20.3 (2022年1月3日)

此版本前向移植了 1.18.4 的更改。

### 已修复

- net: 修复 Windows 命名管道服务器构建器，以在切换管道模式时保持选项 ([#5336])。

[#5336]: https://github.com/tokio-rs/tokio/pull/5336

## 1.20.2 (2022年9月27日)

此版本移除了对 `once_cell` crate 的依赖，以恢复 1.20.x LTS 版本的 MSRV。 ([#5048])

[#5048]: https://github.com/tokio-rs/tokio/pull/5048

## 1.20.1 (2022年7月25日)

### 已修复

- chore: 修复构建脚本中的版本检测 ([#4860])

[#4860]: https://github.com/tokio-rs/tokio/pull/4860

## 1.20.0 (2022年7月12日)

### 已添加
- tokio: 为公共 API 添加 `track_caller` ([#4772], [#4791], [#4793], [#4806], [#4808])
- sync: 为 `watch::Ref` 添加 `has_changed` 方法 ([#4758])

### 已更改

- time: 移除 `src/time/driver/wheel/stack.rs` ([#4766])
- rt: 清理传递给基本调度器的参数 ([#4767])
- net: 更具体地说明 winapi 功能 ([#4764])
- tokio: 在可能的情况下使用 const 初始化的线程本地变量 ([#4677])
- task: 对 LocalKey 的各种小改进 ([#4795])

### 文档

- fs: 警告性能陷阱 ([#4762])
- chore: 修复拼写 ([#4769])
- sync: 记录 oneshot 中的虚假失败 ([#4777])
- sync: 为非 Send future 中的 watch 添加警告 ([#4741])
- chore: 修复拼写错误 ([#4798])

### 非稳定版

- joinset: 将 `join_one` 重命名为 `join_next` ([#4755])
- rt: 为当前线程 rt 添加未处理的 panic 配置 ([#4770])

[#4677]: https://github.com/tokio-rs/tokio/pull/4677
[#4741]: https://github.com/tokio-rs/tokio/pull/4741
[#4755]: https://github.com/tokio-rs/tokio/pull/4755
[#4758]: https://github.com/tokio-rs/tokio/pull/4758
[#4762]: https://github.com/tokio-rs/tokio/pull/4762
[#4764]: https://github.com/tokio-rs/tokio/pull/4764
[#4766]: https://github.com/tokio-rs/tokio/pull/4766
[#4767]: https://github.com/tokio-rs/tokio/pull/4767
[#4769]: https://github.com/tokio-rs/tokio/pull/4769
[#4770]: https://github.com/tokio-rs/tokio/pull/4770
[#4772]: https://github.com/tokio-rs/tokio/pull/4772
[#4777]: https://github.com/tokio-rs/tokio/pull/4777
[#4791]: https://github.com/tokio-rs/tokio/pull/4791
[#4793]: https://github.com/tokio-rs/tokio/pull/4793
[#4795]: https://github.com/tokio-rs/tokio/pull/4795
[#4798]: https://github.com/tokio-rs/tokio/pull/4798
[#4806]: https://github.com/tokio-rs/tokio/pull/4806
[#4808]: https://github.com/tokio-rs/tokio/pull/4808

## 1.19.2 (2022年6月6日)

此版本修复了 `Notified::enable` 中的另一个 bug。([#4751])

[#4751]: https://github.com/tokio-rs/tokio/pull/4751

## 1.19.1 (2022年6月5日)

此版本修复了 `Notified::enable` 中的一个 bug。([#4747])

[#4747]: https://github.com/tokio-rs/tokio/pull/4747

## 1.19.0 (2022年6月3日)

### 已添加

- runtime: 为 `JoinHandle` 和 `AbortHandle` 添加 `is_finished` 方法 ([#4709])
- runtime: 使全局队列和事件轮询间隔可配置 ([#4671])
- sync: 添加 `Notified::enable` ([#4705])
- sync: 添加 `watch::Sender::send_if_modified` ([#4591])
- sync: 为 broadcast::Receiver 添加 resubscribe 方法 ([#4607])
- net: 为 `TcpSocket` 和 `TcpStream` 添加 `take_error` ([#4739])

### 已更改

- io: 重构 io 句柄中 Weak 的使用 ([#4656])

### 已修复

- macros: 避免 `join!` 和 `try_join!` 中的饥饿问题 ([#4624])

### 文档

- runtime: 阐明任务生命周期超过 `block_on` 的语义 ([#4729])
- time: 修复 `MissedTickBehavior::Burst` 的示例 ([#4713])

### 非稳定版

- metrics: 正确更新 `IoDriverMetrics` 中的原子操作 ([#4725])
- metrics: 修复在没有 net 的情况下，使用 unstable、process 和 rt 的编译问题 ([#4682])
- task: 为 `JoinSet`/`JoinMap` 添加 `#[track_caller]` ([#4697])
- task: 添加 `Builder::{spawn_on, spawn_local_on, spawn_blocking_on}` ([#4683])
- task: 为协作调度添加 `consume_budget` ([#4498])
- task: 添加 `join_set::Builder` 以配置 `JoinSet` 任务 ([#4687])
- task: 更新 `JoinSet::join_one` 的返回值 ([#4726])

[#4498]: https://github.com/tokio-rs/tokio/pull/4498
[#4591]: https://github.com/tokio-rs/tokio/pull/4591
[#4607]: https://github.com/tokio-rs/tokio/pull/4607
[#4624]: https://github.com/tokio-rs/tokio/pull/4624
[#4656]: https://github.com/tokio-rs/tokio/pull/4656
[#4671]: https://github.com/tokio-rs/tokio/pull/4671
[#4682]: https://github.com/tokio-rs/tokio/pull/4682
[#4683]: https://github.com/tokio-rs/tokio/pull/4683
[#4687]: https://github.com/tokio-rs/tokio/pull/4687
[#4697]: https://github.com/tokio-rs/tokio/pull/4697
[#4705]: https://github.com/tokio-rs/tokio/pull/4705
[#4709]: https://github.com/tokio-rs/tokio/pull/4709
[#4713]: https://github.com/tokio-rs/tokio/pull/4713
[#4725]: https://github.com/tokio-rs/tokio/pull/4725
[#4726]: https://github.com/tokio-rs/tokio/pull/4726
[#4729]: https://github.com/tokio-rs/tokio/pull/4729
[#4739]: https://github.com/tokio-rs/tokio/pull/4739

## 1.18.6 (2023年5月28日)

### 已修复

- deps: 禁用 mio 的默认功能 ([#5728])

[#5728]: https://github.com/tokio-rs/tokio/pull/5728

## 1.18.5 (2023年1月17日)

### 已修复

- io: 修复 `ReadHalf::unsplit` 中的不健全性 ([#5375])

[#5375]: https://github.com/tokio-rs/tokio/pull/5375

## 1.18.4 (2022年1月3日)

### 已修复

- net: 修复 Windows 命名管道服务器构建器，以在切换管道模式时保持选项 ([#5336])。

[#5336]: https://github.com/tokio-rs/tokio/pull/5336

## 1.18.3 (2022年9月27日)

此版本移除了对 `once_cell` crate 的依赖，以恢复 1.18.x LTS 版本的 MSRV。 ([#5048])

[#5048]: https://github.com/tokio-rs/tokio/pull/5048

## 1.18.2 (2022年5月5日)

为 `winapi` 依赖项添加缺失的功能。 ([#4663])

[#4663]: https://github.com/tokio-rs/tokio/pull/4663

## 1.18.1 (2022年5月2日)

1.18.0 版本在使用 `tokio_unstable` 构建时，在没有 64 位原子操作的目标上破坏了构建。此版本修复了该问题。 ([#4649])

[#4649]: https://github.com/tokio-rs/tokio/pull/4649

## 1.18.0 (2022年4月27日)

此版本在 `tokio::net`、`tokio::signal` 和 `tokio::sync` 中添加了许多新 API。此外，它还向 `tokio::task` 中添加了新的不稳定 API（用于唯一标识任务的 `Id` 和用于远程取消任务的 `AbortHandle`），并修复了许多 bug。

### 已修复

- blocking: 为 `spawn_blocking` 添加缺失的 `#[track_caller]` ([#4616])
- macros: 修复 `select` 宏以处理 64 个分支 ([#4519])
- net: 修复 `try_io` 方法未在内部调用 Mio 的 `try_io` 的问题 ([#4582])
- runtime: 在 OS 无法生成新线程时进行恢复 ([#4485])

### 已添加

- net: 添加 `UdpSocket::peer_addr` ([#4611])
- net: 为命名管道添加 `try_read_buf` 方法 ([#4626])
- signal: 添加 `SignalKind` 的 `Hash`/`Eq` 实现和 `c_int` 转换 ([#4540])
- signal: 添加对最高 `SIGRTMAX` 信号的支持 ([#4555])
- sync: 添加 `watch::Sender::send_modify` 方法 ([#4310])
- sync: 添加 `broadcast::Receiver::len` 方法 ([#4542])
- sync: 添加 `watch::Receiver::same_channel` 方法 ([#4581])
- sync: 为 `RecvError` 类型实现 `Clone` ([#4560])

### 已更改

- 更新 `mio` 至 0.8.1 ([#4582])
- macros: 重命名 `tokio::select!` 的内部 `util` 模块 ([#4543])
- runtime: 在构建运行时使用 `Vec::with_capacity` ([#4553])

### 文档

- 改进 `tokio_unstable` 的文档 ([#4524])
- runtime: 包含更多关于 thread_pool/worker 的文档 ([#4511])
- runtime: 更新 `Handle::current` 的文档以提及 `EnterGuard` ([#4567])
- time: 阐明特定平台的计时器分辨率 ([#4474])
- signal: 记录 `Signal::recv` 是取消安全的 ([#4634])
- sync: `UnboundedReceiver` 关闭文档 ([#4548])

### 非稳定版

以下更改仅在构建时使用 `--cfg tokio_unstable` 时适用：

- task: 添加 `task::Id` 类型 ([#4630])
- task: 添加 `AbortHandle` 类型，用于取消 `JoinSet` 中的任务 ([#4530], [#4640])
- task: 修复 `JoinSet` 缺失的 `doc(cfg(...))` 属性 ([#4531])
- task: 修复 `AbortHandle` RustDoc 中的损坏链接 ([#4545])
- metrics: 添加初始 IO 驱动程序指标 ([#4507])


[#4310]: https://github.com/tokio-rs/tokio/pull/4310
[#4474]: https://github.com/tokio-rs/tokio/pull/4474
[#4485]: https://github.com/tokio-rs/tokio/pull/4485
[#4507]: https://github.com/tokio-rs/tokio/pull/4507
[#4511]: https://github.com/tokio-rs/tokio/pull/4511
[#4519]: https://github.com/tokio-rs/tokio/pull/4519
[#4524]: https://github.com/tokio-rs/tokio/pull/4524
[#4530]: https://github.com/tokio-rs/tokio/pull/4530
[#4531]: https://github.com/tokio-rs/tokio/pull/4531
[#4540]: https://github.com/tokio-rs/tokio/pull/4540
[#4542]: https://github.com/tokio-rs/tokio/pull/4542
[#4543]: https://github.com/tokio-rs/tokio/pull/4543
[#4545]: https://github.com/tokio-rs/tokio/pull/4545
[#4548]: https://github.com/tokio-rs/tokio/pull/4548
[#4553]: https://github.com/tokio-rs/tokio/pull/4553
[#4555]: https://github.com/tokio-rs/tokio/pull/4555
[#4560]: https://github.com/tokio-rs/tokio/pull/4560
[#4567]: https://github.com/tokio-rs/tokio/pull/4567
[#4581]: https://github.com/tokio-rs/tokio/pull/4581
[#4582]: https://github.com/tokio-rs/tokio/pull/4582
[#4611]: https://github.com/tokio-rs/tokio/pull/4611
[#4616]: https://github.com/tokio-rs/tokio/pull/4616
[#4626]: https://github.com/tokio-rs/tokio/pull/4626
[#4630]: https://github.com/tokio-rs/tokio/pull/4630
[#4634]: https://github.com/tokio-rs/tokio/pull/4634
[#4640]: https://github.com/tokio-rs/tokio/pull/4640

## 1.17.0 (2022年2月16日)

此版本将最低支持的 Rust 版本 (MSRV) 更新为 1.49，`mio` 依赖项更新为 v0.8，以及（可选）`parking_lot` 依赖项更新为 v0.12。此外，它还包含多个 bug 修复，以及内部重构和性能改进。

### 已修复

- time: 防止在 `sleep` 中使用大持续时间时 panic ([#4495])
- time: 在 `Instant::now` 不是单调的平台上，消除 `Instant` 算术中潜在的 panic ([#4461])
- io: 修复 `DuplexStream` 未参与协作让步的问题 ([#4478])
- rt: 修复在丢弃 `JoinHandle` 时可能出现的双重 panic ([#4430])

### 已更改

- 将最低支持的 Rust 版本更新为 1.49 ([#4457])
- 将 `parking_lot` 依赖项更新为 v0.12.0 ([#4459])
- 将 `mio` 依赖项更新为 v0.8 ([#4449])
- rt: 移除阻塞池中不必要的锁 ([#4436])
- rt: 移除基本调度器中不必要的枚举 ([#4462])
- time: 使用位操作代替模运算以提高性能 ([#4480])
- net: 使用 `std::future::Ready` 代替我们自己的 `Ready` future ([#4271])
- 将已弃用的 `atomic::spin_loop_hint` 替换为 `hint::spin_loop` ([#4491])
- 修复侵入式链表中的 miri 失败 ([#4397])

### 文档

- io: 为 `tokio::process::ChildStdin` 添加示例 ([#4479])

### 非稳定版

以下更改仅在构建时使用 `--cfg tokio_unstable` 时适用：

- task: 修复由 `spawn_local` 生成的 `tracing` 跨度中缺少的位置信息 ([#4483])
- task: 添加 `JoinSet` 用于管理任务集 ([#4335])
- metrics: 修复 MIPS 上的编译错误 ([#4475])
- metrics: 修复 arm32v7 上的编译错误 ([#4453])

[#4271]: https://github.com/tokio-rs/tokio/pull/4271
[#4335]: https://github.com/tokio-rs/tokio/pull/4335
[#4397]: https://github.com/tokio-rs/tokio/pull/4397
[#4430]: https://github.com/tokio-rs/tokio/pull/4430
[#4436]: https://github.com/tokio-rs/tokio/pull/4436
[#4449]: https://github.com/tokio-rs/tokio/pull/4449
[#4453]: https://github.com/tokio-rs/tokio/pull/4453
[#4457]: https://github.com/tokio-rs/tokio/pull/4457
[#4459]: https://github.com/tokio-rs/tokio/pull/4459
[#4461]: https://github.com/tokio-rs/tokio/pull/4461
[#4462]: https://github.com/tokio-rs/tokio/pull/4462
[#4475]: https://github.com/tokio-rs/tokio/pull/4475
[#4478]: https://github.com/tokio-rs/tokio/pull/4478
[#4479]: https://github.com/tokio-rs/tokio/pull/4479
[#4480]: https://github.com/tokio-rs/tokio/pull/4480
[#4483]: https://github.com/tokio-rs/tokio/pull/4483
[#4491]: https://github.com/tokio-rs/tokio/pull/4491
[#4495]: https://github.com/tokio-rs/tokio/pull/4495

## 1.16.1 (2022年1月28日)

此版本通过更改 [#4437] 修复了 [#4428] 中的一个 bug。

[#4428]: https://github.com/tokio-rs/tokio/pull/4428
[#4437]: https://github.com/tokio-rs/tokio/pull/4437

## 1.16.0 (2022年1月27日)

修复了 `io::Take` 中的一个健全性 bug ([#4428])。当在给定的 `AsyncRead` 实现中泄漏内存，然后覆盖提供的缓冲区时，会暴露此不健全性：

```rust
impl AsyncRead for Buggy {
    fn poll_read(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
        buf: &mut ReadBuf<'_>
    ) -> Poll<Result<()>> {
      let new_buf = vec![0; 5].leak();
      *buf = ReadBuf::new(new_buf);
      buf.put_slice(b"hello");
      Poll::Ready(Ok(()))
    }
}
```

此外，此版本还包括对多线程调度器的改进，在某些情况下可将吞吐量提高多达 20% ([#4383])。

### 已修复

- io: **健全性** 在边缘情况下使用 `io::Take` 时不暴露未初始化的内存 ([#4428])
- fs: 确保在运行时关闭时，`File::write` 的结果是 `write` 系统调用 ([#4316])
- process: 在 `wait_with_output` 中，子进程退出后丢弃管道 ([#4315])
- rt: 改进生成线程失败时的错误消息 ([#4398])
- rt: 减少多线程调度器中错误的线程唤醒 ([#4383])
- sync: 不从 `parking_lot::*Guard` 继承 `Send` ([#4359])

### 已添加

- net: `TcpSocket::linger()` 和 `set_linger()` ([#4324])
- net: 为套接字类型实现 `UnwindSafe` ([#4384])
- rt: 为 `JoinHandle` 实现 `UnwindSafe` ([#4418])
- sync: `watch::Receiver::has_changed()` ([#4342])
- sync: `oneshot::Receiver::blocking_recv()` ([#4334])
- sync: `RwLock` 阻塞操作 ([#4425])

### 非稳定版

以下更改仅在构建时使用 `--cfg tokio_unstable` 时适用

- rt: **破坏性变更** 全面修改运行时指标 API ([#4373])

[#4315]: https://github.com/tokio-rs/tokio/pull/4315
[#4316]: https://github.com/tokio-rs/tokio/pull/4316
[#4324]: https://github.com/tokio-rs/tokio/pull/4324
[#4334]: https://github.com/tokio-rs/tokio/pull/4334
[#4342]: https://github.com/tokio-rs/tokio/pull/4342
[#4359]: https://github.com/tokio-rs/tokio/pull/4359
[#4373]: https://github.com/tokio-rs/tokio/pull/4373
[#4383]: https://github.com/tokio-rs/tokio/pull/4383
[#4384]: https://github.com/tokio-rs/tokio/pull/4384
[#4398]: https://github.com/tokio-rs/tokio/pull/4398
[#4418]: https://github.com/tokio-rs/tokio/pull/4418
[#4425]: https://github.com/tokio-rs/tokio/pull/4425
[#4428]: https://github.com/tokio-rs/tokio/pull/4428

## 1.15.0 (2021年12月15日)

### 已修复

- io: 为 `io::empty()` 添加协作让步支持 ([#4300])
- time: 使 timeout 对耗尽预算的任务具有鲁棒性 ([#4314])

### 已更改

- 将最低支持的 Rust 版本更新为 1.46。

### 已添加

- time: 添加 `Interval::reset()` ([#4248])
- io: 为 `AsyncFdReadyGuard` 添加显式生命周期 ([#4267])
- process: 添加 `Command::as_std()` ([#4295])

### 已添加 (非稳定版)

- tracing: 检测 `tokio::sync` 类型 ([#4302])

[#4248]: https://github.com/tokio-rs/tokio/pull/4248
[#4267]: https://github.com/tokio-rs/tokio/pull/4267
[#4295]: https://github.com/tokio-rs/tokio/pull/4295
[#4300]: https://github.com/tokio-rs/tokio/pull/4300
[#4302]: https://github.com/tokio-rs/tokio/pull/4302
[#4314]: https://github.com/tokio-rs/tokio/pull/4314

## 1.14.0 (2021年11月15日)

### 已修复

- macros: 修复在 `select!` 中使用 `mut` 模式时的编译器错误 ([#4211])
- sync: 修复 `oneshot::Sender::send` 和等待已关闭的 `oneshot::Receiver` 之间的数据竞争 ([#4226])
- sync: 使 `AtomicWaker` panic 安全 ([#3689])
- runtime: 修复基本调度器在运行时上下文之外丢弃任务的问题 ([#4213])

### 已添加

- stats: 添加 `RuntimeStats::busy_duration_total` ([#4179], [#4223])

### 已更改

- io: 更新 `copy` 缓冲区大小以匹配 `std::io::copy` ([#4209])

### 文档

-  io: 在文档测试中将 buffer 重命名为 file ([#4230])
-  sync: 修复 Notify 示例 ([#4212])

[#3689]: https://github.com/tokio-rs/tokio/pull/3689
[#4179]: https://github.com/tokio-rs/tokio/pull/4179
[#4209]: https://github.com/tokio-rs/tokio/pull/4209
[#4211]: https://github.com/tokio-rs/tokio/pull/4211
[#4212]: https://github.com/tokio-rs/tokio/pull/4212
[#4213]: https://github.com/tokio-rs/tokio/pull/4213
[#4223]: https://github.com/tokio-rs/tokio/pull/4223
[#4226]: https://github.com/tokio-rs/tokio/pull/4226
[#4230]: https://github.com/tokio-rs/tokio/pull/4230

## 1.13.1 (2021年11月15日)

### 已修复

- sync: 修复 `oneshot::Sender::send` 和等待已关闭的 `oneshot::Receiver` 之间的数据竞争 ([#4226])

[#4226]: https://github.com/tokio-rs/tokio/pull/4226

## 1.13.0 (2021年10月29日)

### 已修复

- sync: 修复 `Notify` 在锁定其等待者列表之前克隆 waker 的问题 ([#4129])
- tokio: 将 riscv32 添加到非 atomic64 架构 ([#4185])

### 已添加

- net: 为 `udp` 和 `uds_datagram` 添加 `poll_{recv,send}_ready` 方法 ([#4131])
- net: 为分割的一半添加 `try_*`、`readable`、`writable`、`ready` 和 `peer_addr` 方法 ([#4120])
- sync: 为 `Mutex` 添加 `blocking_lock` ([#4130])
- sync: 添加 `watch::Sender::send_replace` ([#3962], [#4195])
- sync: 将 `Mutex<T>` 的 `Debug` 实现扩展到未定大小的 `T` ([#4134])
- tracing: 检测 time::Sleep ([#4072])
- tracing: 为生成的任务使用结构化位置字段 ([#4128])

### 已更改

- io: 在 `copy_bidirectional` 中添加断言，以确保 `poll_write` 是合理的 ([#4125])
- macros: 在 `select!` 中轮询时使用限定语法 ([#4192])
- runtime: 更好地处理 `block_on` 唤醒 ([#4157])
- task: 在调试模式下立即在堆上分配回调 ([#4203])
- tokio: 在构建时断言平台最低要求 ([#3797])

### 文档

- docs: 将文档注释转换为指示性语气 ([#4174])
- docs: 为 `try_join!` 添加在第一个错误时返回的示例 ([#4133])
- docs: 修复 `tokio/src/lib.rs` 中的损坏链接 ([#4132])
- signal: 添加带有后台侦听器的示例 ([#4171])
- sync: 添加更多 oneshot 示例 ([#4153])
- time: 记录 `Interval::tick` 的取消安全性 ([#4152])

[#3797]: https://github.com/tokio-rs/tokio/pull/3797
[#3962]: https://github.com/tokio-rs/tokio/pull/3962
[#4072]: https://github.com/tokio-rs/tokio/pull/4072
[#4120]: https://github.com/tokio-rs/tokio/pull/4120
[#4125]: https://github.com/tokio-rs/tokio/pull/4125
[#4128]: https://github.com/tokio-rs/tokio/pull/4128
[#4129]: https://github.com/tokio-rs/tokio/pull/4129
[#4130]: https://github.com/tokio-rs/tokio/pull/4130
[#4131]: https://github.com/tokio-rs/tokio/pull/4131
[#4132]: https://github.com/tokio-rs/tokio/pull/4132
[#4133]: https://github.com/tokio-rs/tokio/pull/4133
[#4134]: https://github.com/tokio-rs/tokio/pull/4134
[#4152]: https://github.com/tokio-rs/tokio/pull/4152
[#4153]: https://github.com/tokio-rs/tokio/pull/4153
[#4157]: https://github.com/tokio-rs/tokio/pull/4157
[#4171]: https://github.com/tokio-rs/tokio/pull/4171
[#4174]: https://github.com/tokio-rs/tokio/pull/4174
[#4185]: https://github.com/tokio-rs/tokio/pull/4185
[#4192]: https://github.com/tokio-rs/tokio/pull/4192
[#4195]: https://github.com/tokio-rs/tokio/pull/4195
[#4203]: https://github.com/tokio-rs/tokio/pull/4203

## 1.12.0 (2021年9月21日)

### 已修复

- mpsc: 确保 `try_reserve` 错误与 `try_send` 一致 ([#4119])
- mpsc: 使用 `spin_loop_hint` 而不是 `yield_now` ([#4115])
- sync: 将 `SendError` 字段设为公共 ([#4097])

### 已添加

- io: 在 FreeBSD 上添加 POSIX AIO ([#4054])
- io: 添加便利方法 `AsyncSeekExt::rewind` ([#4107])
- runtime: 为 `block_on` future 添加 tracing span ([#4094])
- runtime: 当 worker park 和 unpark 时回调 ([#4070])
- sync: 为 mpsc 通道实现 `try_recv` ([#4113])

### 文档

- docs: 阐明 Tokio 上的 CPU 密集型任务 ([#4105])
- mpsc: 记录 `poll_recv` 上的虚假失败 ([#4117])
- mpsc: 记录 `PollSender` 实现了 `Sink` ([#4110])
- task: 记录 `yield_now` 的非保证 ([#4091])
- time: 更好地记录暂停时间的细节 ([#4061], [#4103])

[#4054]: https://github.com/tokio-rs/tokio/pull/4054
[#4061]: https://github.com/tokio-rs/tokio/pull/4061
[#4070]: https://github.com/tokio-rs/tokio/pull/4070
[#4091]: https://github.com/tokio-rs/tokio/pull/4091
[#4094]: https://github.com/tokio-rs/tokio/pull/4094
[#4097]: https://github.com/tokio-rs/tokio/pull/4097
[#4103]: https://github.com/tokio-rs/tokio/pull/4103
[#4105]: https://github.com/tokio-rs/tokio/pull/4105
[#4107]: https://github.com/tokio-rs/tokio/pull/4107
[#4110]: https://github.com/tokio-rs/tokio/pull/4110
[#4113]: https://github.com/tokio-rs/tokio/pull/4113
[#4115]: https://github.com/tokio-rs/tokio/pull/4115
[#4117]: https://github.com/tokio-rs/tokio/pull/4117
[#4119]: https://github.com/tokio-rs/tokio/pull/4119

## 1.11.0 (2021年8月31日)

### 已修复

 - time: 当 Instant 不是单调时不要 panic ([#4044])
 - io: 修复 `fill_buf` 中的 panic，通过不调用 `poll_fill_buf` 两次 ([#4084])

### 已添加

 - watch: 添加 `watch::Sender::subscribe` ([#3800])
 - process: 为 `ChildStd*` 添加 `from_std` ([#4045])
 - stats: 运行时统计的初步工作 ([#4043])

### 已更改

 - tracing: 将 span 命名更改为新的控制台约定 ([#4042])
 - io: 通过使用未初始化的数组来加快唤醒速度 ([#4055], [#4071], [#4075])

### 文档

 - time: 使 Sleep 示例更容易找到 ([#4040])

[#3800]: https://github.com/tokio-rs/tokio/pull/3800
[#4040]: https://github.com/tokio-rs/tokio/pull/4040
[#4042]: https://github.com/tokio-rs/tokio/pull/4042
[#4043]: https://github.com/tokio-rs/tokio/pull/4043
[#4044]: https://github.com/tokio-rs/tokio/pull/4044
[#4045]: https://github.com/tokio-rs/tokio/pull/4045
[#4055]: https://github.com/tokio-rs/tokio/pull/4055
[#4071]: https://github.com/tokio-rs/tokio/pull/4071
[#4075]: https://github.com/tokio-rs/tokio/pull/4075
[#4084]: https://github.com/tokio-rs/tokio/pull/4084

## 1.10.1 (2021年8月24日)

### 已修复

 - runtime: 修复 UnownedTask 中的泄漏 ([#4063])

[#4063]: https://github.com/tokio-rs/tokio/pull/4063

## 1.10.0 (2021年8月12日)

### 已添加

 - io: 添加 `(read|write)_f(32|64)[_le]` 方法 ([#4022])
 - io: 向 `AsyncBufReadExt` 添加 `fill_buf` 和 `consume` ([#3991])
 - process: 在 windows 上添加 `Child::raw_handle()` ([#3998])

### 已修复

 - doc: 修复使用 `--cfg docsrs` 的非文档构建 ([#4020])
 - io: 在 `io::copy` 中急切刷新 ([#4001])
 - runtime: 关闭期间有时会触发调试断言 ([#4005])
 - sync: 在 mpsc 中使用 `spin_loop_hint` 代替 `yield_now` ([#4037])
 - tokio: test-util 功能依赖于 rt、sync 和 time ([#4036])

### 变更

 - runtime: 重组部分运行时 ([#3979], [#4005])
 - signal: 使 signal 模块的 windows 文档在 unix 构建中显示 ([#3770])
 - task: 在调试模式下快速将任务发送到堆 ([#4009])

### 文档

 - io: 记录 `AsyncBufReadExt` 的取消安全性 ([#3997])
 - sync: 记录 `watch::send` 失败的时间 ([#4021])

[#3770]: https://github.com/tokio-rs/tokio/pull/3770
[#3979]: https://github.com/tokio-rs/tokio/pull/3979
[#3991]: https://github.com/tokio-rs/tokio/pull/3991
[#3997]: https://github.com/tokio-rs/tokio/pull/3997
[#3998]: https://github.com/tokio-rs/tokio/pull/3998
[#4001]: https://github.com/tokio-rs/tokio/pull/4001
[#4005]: https://github.com/tokio-rs/tokio/pull/4005
[#4009]: https://github.com/tokio-rs/tokio/pull/4009
[#4020]: https://github.com/tokio-rs/tokio/pull/4020
[#4021]: https://github.com/tokio-rs/tokio/pull/4021
[#4022]: https://github.com/tokio-rs/tokio/pull/4022
[#4036]: https://github.com/tokio-rs/tokio/pull/4036
[#4037]: https://github.com/tokio-rs/tokio/pull/4037

## 1.9.0 (2021年7月22日)

### 已添加

 - net: 允许 `TcpStream` 的自定义 I/O 操作 ([#3888])
 - sync: 添加从 guard 获取互斥锁的 getter ([#3928])
 - task: 为 `TaskLocal::scope` 暴露可命名的 future ([#3273])

### 已修复

 - 如果 future 的输出在 drop 时 panic，则修复泄漏 ([#3967])
 - 修复 `LocalSet` 中的泄漏 ([#3978])

### 变更

 - runtime: 重组部分运行时 ([#3909], [#3939], [#3950], [#3955], [#3980])
 - sync: 清理 `OnceCell` ([#3945])
 - task: 移除 `JoinError` 中的互斥锁 ([#3959])

[#3273]: https://github.com/tokio-rs/tokio/pull/3273
[#3888]: https://github.com/tokio-rs/tokio/pull/3888
[#3909]: https://github.com/tokio-rs/tokio/pull/3909
[#3928]: https://github.com/tokio-rs/tokio/pull/3928
[#3939]: https://github.com/tokio-rs/tokio/pull/3939
[#3945]: https://github.com/tokio-rs/tokio/pull/3945
[#3950]: https://github.com/tokio-rs/tokio/pull/3950
[#3955]: https://github.com/tokio-rs/tokio/pull/3955
[#3959]: https://github.com/tokio-rs/tokio/pull/3959
[#3967]: https://github.com/tokio-rs/tokio/pull/3967
[#3978]: https://github.com/tokio-rs/tokio/pull/3978
[#3980]: https://github.com/tokio-rs/tokio/pull/3980

## 1.8.3 (2021年7月26日)

此版本向后移植了 1.9.0 的两项修复

### 已修复

 - 如果 future 的输出在 drop 时 panic，则修复泄漏 ([#3967])
 - 修复 `LocalSet` 中的泄漏 ([#3978])

[#3967]: https://github.com/tokio-rs/tokio/pull/3967
[#3978]: https://github.com/tokio-rs/tokio/pull/3978

## 1.8.2 (2021年7月19日)

修复了 1.8.1 中遗漏的边缘情况。

### 已修复

- runtime: 在下一次轮询时丢弃已取消的 future ([#3965])

[#3965]: https://github.com/tokio-rs/tokio/pull/3965

## 1.8.1 (2021年7月6日)

前向移植 1.5.1 的修复。

### 已修复

- runtime: 在 `JoinHandle::abort` 上远程中止任务 ([#3934])

[#3934]: https://github.com/tokio-rs/tokio/pull/3934

## 1.8.0 (2021年7月2日)

### 已添加

- io: 为 `AsyncFdReadyGuard` 和 `AsyncFdReadyMutGuard` 添加 `get_{ref,mut}` 方法 ([#3807])
- io: 为 `BufWriter` 实现高效的向量化写入 ([#3163])
- net: 为 `NamedPipe{Client,Server}` 添加 ready/try 方法 ([#3866], [#3899])
- sync: 添加 `watch::Receiver::borrow_and_update` ([#3813])
- sync: 为 `OnceCell<T>` 实现 `From<T>` ([#3877])
- time: 允许用户指定延迟时 Interval 的行为 ([#3721])

### 已添加 (非稳定版)

- rt: 添加 `tokio::task::Builder` ([#3881])

### 已修复

- net: 使用 `UnixStream` 处理 HUP 事件 ([#3898])

### 文档

- doc: 记录取消安全性 ([#3900])
- time: 为 sleep 添加 wait 别名 ([#3897])
- time: 记录运行时的自动推进行为 ([#3763])

[#3163]: https://github.com/tokio-rs/tokio/pull/3163
[#3721]: https://github.com/tokio-rs/tokio/pull/3721
[#3763]: https://github.com/tokio-rs/tokio/pull/3763
[#3807]: https://github.com/tokio-rs/tokio/pull/3807
[#3813]: https://github.com/tokio-rs/tokio/pull/3813
[#3866]: https://github.com/tokio-rs/tokio/pull/3866
[#3877]: https://github.com/tokio-rs/tokio/pull/3877
[#3881]: https://github.com/tokio-rs/tokio/pull/3881
[#3897]: https://github.com/tokio-rs/tokio/pull/3897
[#3898]: https://github.com/tokio-rs/tokio/pull/3898
[#3899]: https://github.com/tokio-rs/tokio/pull/3899
[#3900]: https://github.com/tokio-rs/tokio/pull/3900

## 1.7.2 (2021年7月6日)

前向移植 1.5.1 的修复。

### 已修复

- runtime: 在 `JoinHandle::abort` 上远程中止任务 ([#3934])

[#3934]: https://github.com/tokio-rs/tokio/pull/3934

## 1.7.1 (2021年6月18日)

### 已修复

- runtime: 修复运行时关闭期间任务提前关闭的问题 ([#3870])

[#3870]: https://github.com/tokio-rs/tokio/pull/3870

## 1.7.0 (2021年6月15日)

### 已添加

- net: 在 windows 上添加命名管道 ([#3760])
- net: 添加从 `std::net::TcpStream` 转换的 `TcpSocket` ([#3838])
- sync: 为 `watch::Sender` 添加 `receiver_count` ([#3729])
- sync: 公开导出 `sync::notify::Notified` future ([#3840])
- tracing: 检测任务 waker ([#3836])

### 已修复

- macros: 在生成的代码中抑制 `clippy::default_numeric_fallback` lint ([#3831])
- runtime: 在运行时关闭时立即丢弃新任务 ([#3752])
- sync: 废弃未使用的 `mpsc::RecvError` 类型 ([#3833])

### 文档

- io: 阐明 `AsyncReadExt::read_buf` 的 EOF 条件 ([#3850])
- io: 阐明 `AsyncWrite::poll_write` 返回值的限制 ([#3820])
- sync: 为 Semaphore 添加示例 ([#3808])

[#3729]: https://github.com/tokio-rs/tokio/pull/3729
[#3752]: https://github.com/tokio-rs/tokio/pull/3752
[#3760]: https://github.com/tokio-rs/tokio/pull/3760
[#3808]: https://github.com/tokio-rs/tokio/pull/3808
[#3820]: https://github.com/tokio-rs/tokio/pull/3820
[#3831]: https://github.com/tokio-rs/tokio/pull/3831
[#3833]: https://github.com/tokio-rs/tokio/pull/3833
[#3836]: https://github.com/tokio-rs/tokio/pull/3836
[#3838]: https://github.com/tokio-rs/tokio/pull/3838
[#3840]: https://github.com/tokio-rs/tokio/pull/3840
[#3850]: https://github.com/tokio-rs/tokio/pull/3850

## 1.6.3 (2021年7月6日)

前向移植 1.5.1 的修复。

### 已修复

- runtime: 在 `JoinHandle::abort` 上远程中止任务 ([#3934])

[#3934]: https://github.com/tokio-rs/tokio/pull/3934

## 1.6.2 (2021年6月14日)

### 修复

- test: 1.6 中引入的亚毫秒级 `time:advance` 回归 ([#3852])

[#3852]: https://github.com/tokio-rs/tokio/pull/3852

## 1.6.1 (2021年5月28日)

此版本恢复了 [#3518]，因为它在某些内核上由于内核 bug 而无法工作。([#3803])

[#3518]: https://github.com/tokio-rs/tokio/issues/3518
[#3803]: https://github.com/tokio-rs/tokio/issues/3803

## 1.6.0 (2021年5月14日)

### 已添加

- fs: 在转交给线程池之前尝试进行非阻塞读取 ([#3518])
- io: 向 `AsyncWriteExt` 添加 `write_all_buf` ([#3737])
- io: 为 `BufReader`、`BufWriter` 和 `BufStream` 实现 `AsyncSeek` ([#3491])
- net: 支持非阻塞向量化 I/O ([#3761])
- sync: 添加 `mpsc::Sender::{reserve_owned, try_reserve_owned}` ([#3704])
- sync: 添加一个返回 `MappedMutexGuard` 的 `MutexGuard::map` 方法 ([#2472])
- time: 添加获取 Interval 周期的 getter ([#3705])

### 已修复

- io: 在 `DuplexStream` 关闭时唤醒待处理的写入器 ([#3756])
- process: 避免在回收孤儿进程时进行冗余操作 ([#3743])
- signal: 在公共 API 上使用 `std::os::raw::c_int` 而不是 `libc::c_int` ([#3774])
- sync: 在 `notify_waiters` 中保留许可状态 ([#3660])
- task: 更新 `JoinHandle` panic 消息 ([#3727])
- time: 防止 `time::advance` 走得太远 ([#3712])

### 文档

- net: 从文档中隐藏 `net::unix::datagram` 模块 ([#3775])
- process: 更新了示例 ([#3748])
- sync: `Barrier` 文档应使用任务，而不是线程 ([#3780])
- task: 更新关于 `block_in_place` 的文档 ([#3753])

[#2472]: https://github.com/tokio-rs/tokio/pull/2472
[#3491]: https://github.com/tokio-rs/tokio/pull/3491
[#3518]: https://github.com/tokio-rs/tokio/pull/3518
[#3660]: https://github.com/tokio-rs/tokio/pull/3660
[#3704]: https://github.com/tokio-rs/tokio/pull/3704
[#3705]: https://github.com/tokio-rs/tokio/pull/3705
[#3712]: https://github.com/tokio-rs/tokio/pull/3712
[#3727]: https://github.com/tokio-rs/tokio/pull/3727
[#3737]: https://github.com/tokio-rs/tokio/pull/3737
[#3743]: https://github.com/tokio-rs/tokio/pull/3743
[#3748]: https://github.com/tokio-rs/tokio/pull/3748
[#3753]: https://github.com/tokio-rs/tokio/pull/3753
[#3756]: https://github.com/tokio-rs/tokio/pull/3756
[#3761]: https://github.com/tokio-rs/tokio/pull/3761
[#3774]: https://github.com/tokio-rs/tokio/pull/3774
[#3775]: https://github.com/tokio-rs/tokio/pull/3775
[#3780]: https://github.com/tokio-rs/tokio/pull/3780

## 1.5.1 (2021年7月6日)

### 已修复

- runtime: 在 `JoinHandle::abort` 上远程中止任务 ([#3934])

[#3934]: https://github.com/tokio-rs/tokio/pull/3934

## 1.5.0 (2021年4月12日)

### 已添加

- io: 添加 `AsyncSeekExt::stream_position` ([#3650])
- io: 添加 `AsyncWriteExt::write_vectored` ([#3678])
- io: 添加 `copy_bidirectional` 工具 ([#3572])
- net: 为 `TcpSocket` 实现 `IntoRawFd` ([#3684])
- sync: 添加 `OnceCell` ([#3591])
- sync: 添加 `OwnedRwLockReadGuard` 和 `OwnedRwLockWriteGuard` ([#3340])
- sync: 添加 `Semaphore::is_closed` ([#3673])
- sync: 添加 `mpsc::Sender::capacity` ([#3690])
- sync: 允许配置 `RwLock` 最大读取数 ([#3644])
- task: 为 `LocalKey` 添加 `sync_scope` ([#3612])

### 已修复

- chore: 尝试避免在侵入式链表上使用 `noalias` 属性 ([#3654])
- rt: 修复从其他线程调用时 `JoinHandle::abort()` 中的 panic ([#3672])
- sync: 不要在 `oneshot::try_recv` 中 panic ([#3674])
- sync: 修复接收器丢弃时通知被丢弃的问题 ([#3652])
- sync: 修复 `Semaphore` 许可溢出计算 ([#3644])

### 文档

- io: 阐明 `AsyncFd` 的要求 ([#3635])
- runtime: 修复 `{Handle,Runtime}::block_on` 的不清晰文档 ([#3628])
- sync: 记录 `Semaphore` 是公平的 ([#3693])
- sync: 改进阻塞互斥锁的文档 ([#3645])

[#3340]: https://github.com/tokio-rs/tokio/pull/3340
[#3572]: https://github.com/tokio-rs/tokio/pull/3572
[#3591]: https://github.com/tokio-rs/tokio/pull/3591
[#3612]: https://github.com/tokio-rs/tokio/pull/3612
[#3628]: https://github.com/tokio-rs/tokio/pull/3628
[#3635]: https://github.com/tokio-rs/tokio/pull/3635
[#3644]: https://github.com/tokio-rs/tokio/pull/3644
[#3645]: https://github.com/tokio-rs/tokio/pull/3645
[#3650]: https://github.com/tokio-rs/tokio/pull/3650
[#3652]: https://github.com/tokio-rs/tokio/pull/3652
[#3654]: https://github.com/tokio-rs/tokio/pull/3654
[#3672]: https://github.com/tokio-rs/tokio/pull/3672
[#3673]: https://github.com/tokio-rs/tokio/pull/3673
[#3674]: https://github.com/tokio-rs/tokio/pull/3674
[#3678]: https://github.com/tokio-rs/tokio/pull/3678
[#3684]: https://github.com/tokio-rs/tokio/pull/3684
[#3690]: https://github.com/tokio-rs/tokio/pull/3690
[#3693]: https://github.com/tokio-rs/tokio/pull/3693

## 1.4.0 (2021年3月20日)

### 已添加

- macros: 为 `select!` 引入 biased 参数 ([#3603])
- runtime: 添加 `Handle::block_on` ([#3569])

### 已修复

- runtime: 避免不必要的 `block_on` future 轮询 ([#3582])
- runtime: 修复创建许多运行时时的内存泄漏/增长问题 ([#3564])
- runtime: 用 `must_use` 标记 `EnterGuard` ([#3609])

### 文档

- chore: 在贡献指南中提及构建文档的修复方法 ([#3618])
- doc: 添加到 `PollSender` 的链接 ([#3613])
- doc: 将 sleep 别名为 delay ([#3604])
- sync: 改进 `Mutex` FIFO 解释 ([#3615])
- timer: 修复模块文档中的双换行 ([#3617])

[#3564]: https://github.com/tokio-rs/tokio/pull/3564
[#3569]: https://github.com/tokio-rs/tokio/pull/3569
[#3582]: https://github.com/tokio-rs/tokio/pull/3582
[#3603]: https://github.com/tokio-rs/tokio/pull/3603
[#3604]: https://github.com/tokio-rs/tokio/pull/3604
[#3609]: https://github.com/tokio-rs/tokio/pull/3609
[#3613]: https://github.com/tokio-rs/tokio/pull/3613
[#3615]: https://github.com/tokio-rs/tokio/pull/3615
[#3617]: https://github.com/tokio-rs/tokio/pull/3617
[#3618]: https://github.com/tokio-rs/tokio/pull/3618

## 1.3.0 (2021年3月9日)

### 已添加

- coop: 暴露一个 `unconstrained()` 选择退出 ([#3547])
- net: 为没有 `into_std` 的网络类型添加该方法 ([#3509])
- sync: 为 `mpsc::Sender` 添加 `same_channel` 方法 ([#3532])
- sync: 为 `Semaphore` 添加 `{try_,}acquire_many_owned` ([#3535])
- sync: 重新添加 `RwLockWriteGuard::map` 和 `RwLockWriteGuard::try_map` ([#3348])

### 已修复

- sync: 允许在成功的 `try_recv` 后调用 `oneshot::Receiver::close` ([#3552])
- time: 在 `timeout(Duration::MAX)` 时不要 panic ([#3551])

### 文档

- doc: 为 1.0 之前的函数名添加文档别名 ([#3523])
- io: 修复拼写错误 ([#3541])
- io: 注意 `read_until` 的 EOF 行为 ([#3536])
- io: 更新 `AsyncRead::poll_read` 文档 ([#3557])
- net: 更新 `UdpSocket` 拆分文档 ([#3517])
- runtime: 在 `new_current_thread` 上添加 `LocalSet` 链接 ([#3508])
- runtime: 更新线程限制的文档 ([#3527])
- sync: 不推荐为 `Barrier` 使用 `join_all` ([#3514])
- sync: `oneshot` 的文档 ([#3592])
- sync: 将 `notify` 重命名为 `notify_one` ([#3526])
- time: 修复 `Sleep` 文档中的拼写错误 ([#3515])
- time: 同步 `interval.rs` 和 `time/mod.rs` 文档 ([#3533])

[#3348]: https://github.com/tokio-rs/tokio/pull/3348
[#3508]: https://github.com/tokio-rs/tokio/pull/3508
[#3509]: https://github.com/tokio-rs/tokio/pull/3509
[#3514]: https://github.com/tokio-rs/tokio/pull/3514
[#3515]: https://github.com/tokio-rs/tokio/pull/3515
[#3517]: https://github.com/tokio-rs/tokio/pull/3517
[#3523]: https://github.com/tokio-rs/tokio/pull/3523
[#3526]: https://github.com/tokio-rs/tokio/pull/3526
[#3527]: https://github.com/tokio-rs/tokio/pull/3527
[#3532]: https://github.com/tokio-rs/tokio/pull/3532
[#3533]: https://github.com/tokio-rs/tokio/pull/3533
[#3535]: https://github.com/tokio-rs/tokio/pull/3535
[#3536]: https://github.com/tokio-rs/tokio/pull/3536
[#3541]: https://github.com/tokio-rs/tokio/pull/3541
[#3547]: https://github.com/tokio-rs/tokio/pull/3547
[#3551]: https://github.com/tokio-rs/tokio/pull/3551
[#3552]: https://github.com/tokio-rs/tokio/pull/3552
[#3557]: https://github.com/tokio-rs/tokio/pull/3557
[#3592]: https://github.com/tokio-rs/tokio/pull/3592

## 1.2.0 (2021年2月5日)

### 已添加

- signal: 将 `Signal::poll_recv` 方法设为公共 ([#3383])

### 已修复

- time: 使 `test-util` 暂停时间完全确定 ([#3492])

### 文档

- sync: 链接到新的广播和 watch 包装器 ([#3504])

[#3383]: https://github.com/tokio-rs/tokio/pull/3383
[#3492]: https://github.com/tokio-rs/tokio/pull/3492
[#3504]: https://github.com/tokio-rs/tokio/pull/3504

## 1.1.1 (2021年1月29日)

前向移植 1.0.3 的修复。

### 已修复
- io: 关闭期间的内存泄漏 ([#3477])。

[#3477]: https://github.com/tokio-rs/tokio/pull/3477

## 1.1.0 (2021年1月22日)

### 已添加

- net: 添加 `try_read_buf` 和 `try_recv_buf` ([#3351])
- mpsc: 添加 `Sender::try_reserve` 函数 ([#3418])
- sync: 添加 `RwLock` 的 `try_read` 和 `try_write` 方法 ([#3400])
- io: 添加 `ReadBuf::inner_mut` ([#3443])

### 已更改

- macros: 改进 `select!` 错误消息 ([#3352])
- io: 在 `read_to_end` 中跟踪已初始化的字节 ([#3426])
- runtime: 整合上下文缺失的错误 ([#3441])

### 已修复

- task: 在 `spawn_local` 时唤醒 `LocalSet` ([#3369])
- sync: 修复 broadcast::Receiver drop 中的 panic ([#3434])

### 文档
- stream: 在 `tokio-stream` 中链接到新的 `Stream` 包装器 ([#3343])
- docs: 提及 `test-util` 功能未与 full 一起启用 ([#3397])
- process: 为 process::Child 字段添加文档 ([#3437])
- io: 阐明 `AsyncFd` 文档中关于内部 fd 更改的内容 ([#3430])
- net: 更新数据报的拆分文档 ([#3448])
- time: 记录 `Sleep` 不是 `Unpin` ([#3457])
- sync: 添加到 `PollSemaphore` 的链接 ([#3456])
- task: 添加 `LocalSet` 示例 ([#3438])
- sync: 改进有界 `mpsc` 文档 ([#3458])

[#3343]: https://github.com/tokio-rs/tokio/pull/3343
[#3351]: https://github.com/tokio-rs/tokio/pull/3351
[#3352]: https://github.com/tokio-rs/tokio/pull/3352
[#3369]: https://github.com/tokio-rs/tokio/pull/3369
[#3397]: https://github.com/tokio-rs/tokio/pull/3397
[#3400]: https://github.com/tokio-rs/tokio/pull/3400
[#3418]: https://github.com/tokio-rs/tokio/pull/3418
[#3426]: https://github.com/tokio-rs/tokio/pull/3426
[#3430]: https://github.com/tokio-rs/tokio/pull/3430
[#3434]: https://github.com/tokio-rs/tokio/pull/3434
[#3437]: https://github.com/tokio-rs/tokio/pull/3437
[#3438]: https://github.com/tokio-rs/tokio/pull/3438
[#3441]: https://github.com/tokio-rs/tokio/pull/3441
[#3443]: https://github.com/tokio-rs/tokio/pull/3443
[#3448]: https://github.com/tokio-rs/tokio/pull/3448
[#3456]: https://github.com/tokio-rs/tokio/pull/3456
[#3457]: https://github.com/tokio-rs/tokio/pull/3457
[#3458]: https://github.com/tokio-rs/tokio/pull/3458

## 1.0.3 (2021年1月28日)

### 已修复
- io: 关闭期间的内存泄漏 ([#3477])。

[#3477]: https://github.com/tokio-rs/tokio/pull/3477

## 1.0.2 (2021年1月14日)

### 已修复
- io: `read_to_end` 中的健全性问题 ([#3428])。

[#3428]: https://github.com/tokio-rs/tokio/pull/3428

## 1.0.1 (2020年12月25日)

此版本通过移除 `map` 函数修复了由 `RwLockWriteGuard::map` 和 `RwLockWriteGuard::downgrade` 组合引起的健全性漏洞。这是一个破坏性变更，但根据我们的 semver 策略，当需要修复健全性漏洞时，允许进行破坏性变更。（更多信息请参见 [此 RFC][semver]。）

请注意，我们选择不进行弃用周期或类似操作，因为 Tokio 1.0.0 是在两天前发布的，因此影响应该很小。

由于健全性漏洞，我们还撤销了 Tokio 1.0.0 版本。

### 已移除

- sync: 移除 `RwLockWriteGuard::map` 和 `RwLockWriteGuard::try_map` ([#3345])

### 已修复

- docs: 从文档中移除 stream 功能 ([#3335])

[semver]: https://github.com/rust-lang/rfcs/blob/master/text/1122-language-semver.md#soundness-changes
[#3335]: https://github.com/tokio-rs/tokio/pull/3335
[#3345]: https://github.com/tokio-rs/tokio/pull/3345

## 1.0.0 (2020年12月23日)

致力于 API 和长期支持。

### 已修复

- sync: `watch` 中的虚假唤醒 ([#3234])。

### 已更改

- io: 将 `AsyncFd::with_io()` 重命名为 `try_io()` ([#3306])
- fs: 避免使用特定于操作系统的 `*Ext` trait，而是在条件上定义 fn ([#3264])。
- fs: `Sleep` 是 `!Unpin` ([#3278])。
- net: 按值传递 `SocketAddr` ([#3125])。
- net: `TcpStream::poll_peek` 接受 `ReadBuf` ([#3259])。
- rt: 将 `runtime::Builder::max_threads()` 重命名为 `max_blocking_threads()` ([#3287])。
- time: 调用 `time::pause()` 时需要 `current_thread` 运行时 ([#3289])。

### 已移除

- 移除 `tokio::prelude` ([#3299])。
- io: 移除 `AsyncFd::with_poll()` ([#3306])。
- net: 移除 `{Tcp,Unix}Stream::shutdown()`，改用 `AsyncWrite::shutdown()` ([#3298])。
- stream: 将所有 stream 工具移至 `tokio-stream`，直到 `Stream` 添加到 `std` ([#3277])。
- sync: mpsc `try_recv()` 因意外行为而被移除 ([#3263])。
- tracing: 由于 `tracing-core` 尚未达到 1.0，设为不稳定 ([#3266])。

### 已添加

- fs: 将 `poll_*` fn 添加到 `DirEntry` ([#3308])。
- io: 将 `poll_*` fn 添加到 `io::Lines`、`io::Split` ([#3308])。
- io: 将 `_mut` 方法变体添加到 `AsyncFd` ([#3304])。
- net: 将 `poll_*` fn 添加到 `UnixDatagram` ([#3223])。
- net: `UnixStream` 就绪和非阻塞操作 ([#3246])。
- sync: `UnboundedReceiver::blocking_recv()` ([#3262])。
- sync: `watch::Sender::borrow()` ([#3269])。
- sync: `Semaphore::close()` ([#3065])。
- sync: 将 `poll_recv` fn 添加到 `mpsc::Receiver`、`mpsc::UnboundedReceiver` ([#3308])。
- time: 将 `poll_tick` fn 添加到 `time::Interval` ([#3316])。

[#3065]: https://github.com/tokio-rs/tokio/pull/3065
[#3125]: https://github.com/tokio-rs/tokio/pull/3125
[#3223]: https://github.com/tokio-rs/tokio/pull/3223
[#3234]: https://github.com/tokio-rs/tokio/pull/3234
[#3246]: https://github.com/tokio-rs/tokio/pull/3246
[#3259]: https://github.com/tokio-rs/tokio/pull/3259
[#3262]: https://github.com/tokio-rs/tokio/pull/3262
[#3263]: https://github.com/tokio-rs/tokio/pull/3263
[#3264]: https://github.com/tokio-rs/tokio/pull/3264
[#3266]: https://github.com/tokio-rs/tokio/pull/3266
[#3269]: https://github.com/tokio-rs/tokio/pull/3269
[#3277]: https://github.com/tokio-rs/tokio/pull/3277
[#3278]: https://github.com/tokio-rs/tokio/pull/3278
[#3287]: https://github.com/tokio-rs/tokio/pull/3287
[#3289]: https://github.com/tokio-rs/tokio/pull/3289
[#3298]: https://github.com/tokio-rs/tokio/pull/3298
[#3299]: https://github.com/tokio-rs/tokio/pull/3299
[#3304]: https://github.com/tokio-rs/tokio/pull/3304
[#3306]: https://github.com/tokio-rs/tokio/pull/3306
[#3308]: https://github.com/tokio-rs/tokio/pull/3308
[#3316]: https://github.com/tokio-rs/tokio/pull/3316

## 0.3.6 (2020年12月14日)

### 已修复

- rt: 修复关闭时的死锁 ([#3228])
- rt: 修复在 rt 外中止任务时的 panic ([#3159])
- sync: 使 `add_permits` 在 usize::MAX >> 3 个许可时 panic ([#3188])
- time: 修复计时器丢弃时的竞争条件 ([#3229])
- watch: 修复虚假唤醒 ([#3244])

### 已添加

- example: 添加回 udp-codec 示例 ([#3205])
- net: 添加 `TcpStream::into_std` ([#3189])

[#3159]: https://github.com/tokio-rs/tokio/pull/3159
[#3188]: https://github.com/tokio-rs/tokio/pull/3188
[#3189]: https://github.com/tokio-rs/tokio/pull/3189
[#3205]: https://github.com/tokio-rs/tokio/pull/3205
[#3228]: https://github.com/tokio-rs/tokio/pull/3228
[#3229]: https://github.com/tokio-rs/tokio/pull/3229
[#3244]: https://github.com/tokio-rs/tokio/pull/3244

## 0.3.5 (2020年11月30日)

### 已修复

- rt: 修复 `shutdown_timeout(0)` ([#3196])。
- time: 修复了小睡眠时间的竞争条件 ([#3069])。

### 已添加

- io: `AsyncFd::with_interest()` ([#3167])。
- signal: windows 上的 `CtrlC` 流 ([#3186])。

[#3069]: https://github.com/tokio-rs/tokio/pull/3069
[#3167]: https://github.com/tokio-rs/tokio/pull/3167
[#3186]: https://github.com/tokio-rs/tokio/pull/3186
[#3196]: https://github.com/tokio-rs/tokio/pull/3196

## 0.3.4 (2020年11月18日)

### 已修复

- stream: `StreamMap` `Default` 实现绑定 ([#3093])。
- io: `AsyncFd::into_inner()` 应取消注册 FD ([#3104])。

### 已更改

- meta: `parking_lot` 功能通过 `full` 启用 ([#3119])。

### 已添加

- io: `AsyncWrite` 向量化写入 ([#3149])。
- net: TCP/UDP 就绪和非阻塞操作 ([#3130], [#2743], [#3138])。
- net: TCP 套接字选项 (linger, send/recv buf size) ([#3145], [#3143])。
- net: 在 solaris/illumos 上的 `UCred` 中添加 PID 字段 ([#3085])。
- rt: `runtime::Handle` 允许生成到运行时 ([#3079])。
- sync: `Notify::notify_waiters()` ([#3098])。
- sync: 将 `acquire_many()`、`try_acquire_many()` 添加到 `Semaphore` ([#3067])。

[#2743]: https://github.com/tokio-rs/tokio/pull/2743
[#3067]: https://github.com/tokio-rs/tokio/pull/3067
[#3079]: https://github.com/tokio-rs/tokio/pull/3079
[#3085]: https://github.com/tokio-rs/tokio/pull/3085
[#3093]: https://github.com/tokio-rs/tokio/pull/3093
[#3098]: https://github.com/tokio-rs/tokio/pull/3098
[#3104]: https://github.com/tokio-rs/tokio/pull/3104
[#3119]: https://github.com/tokio-rs/tokio/pull/3119
[#3130]: https://github.com/tokio-rs/tokio/pull/3130
[#3138]: https://github.com/tokio-rs/tokio/pull/3138
[#3143]: https://github.com/tokio-rs/tokio/pull/3143
[#3145]: https://github.com/tokio-rs/tokio/pull/3145
[#3149]: https://github.com/tokio-rs/tokio/pull/3149

## 0.3.3 (2020年11月2日)

通过向 `Runtime::spawn_blocking()` 添加缺失的 `Send` 绑定来修复一个健全性漏洞。

### 已修复

- rt: 包含缺失的 `Send`，修复健全性漏洞 ([#3089])。
- tracing: 避免巨大的 trace span 名称 ([#3074])。

### 已添加

- net: `TcpSocket::reuseport()`、`TcpSocket::set_reuseport()` ([#3083])。
- net: `TcpSocket::reuseaddr()` ([#3093])。
- net: `TcpSocket::local_addr()` ([#3093])。
- net: 将 pid 添加到 `UCred` ([#2633])。

[#2633]: https://github.com/tokio-rs/tokio/pull/2633
[#3074]: https://github.com/tokio-rs/tokio/pull/3074
[#3083]: https://github.com/tokio-rs/tokio/pull/3083
[#3089]: https://github.com/tokio-rs/tokio/pull/3089
[#3093]: https://github.com/tokio-rs/tokio/pull/3093

## 0.3.2 (2020年10月27日)

添加 `AsyncFd` 作为 v0.2 的 `PollEvented` 的替代品。

### 已修复

- io: 修复关闭 I/O 驱动程序时可能出现的死锁 ([#2903])。
- sync: `RwLockWriteGuard::downgrade()` bug ([#2957])。

### 已添加

- io: `AsyncFd` 用于在原始 FD 上接收就绪事件 ([#2903])。
- net: `UdpSocket` 上的 `poll_*` 函数 ([#2981])。
- net: `UdpSocket::take_error()` ([#3051])。
- sync: `oneshot::Sender::poll_closed()` ([#3032])。

[#2903]: https://github.com/tokio-rs/tokio/pull/2903
[#2957]: https://github.com/tokio-rs/tokio/pull/2957
[#2981]: https://github.com/tokio-rs/tokio/pull/2981
[#3032]: https://github.com/tokio-rs/tokio/pull/3032
[#3051]: https://github.com/tokio-rs/tokio/pull/3051

## 0.3.1 (2020年10月21日)

此版本修复了 IO 驱动程序中的一个 use-after-free 问题。此外，`read_buf` 和 `write_buf` 方法已重新添加到 IO trait 中，因为 bytes crate 现在正与 Tokio 一起迈向 1.0 版本。

### 已修复

- net: 修复 use-after-free ([#3019])。
- fs: 确保在关闭时写入缓冲数据 ([#3009])。

### 已添加

- io: `copy_buf()` ([#2884])。
- io: `AsyncReadExt::read_buf()`、`AsyncReadExt::write_buf()` 用于处理 `Buf`/`BufMut` ([#3003])。
- rt: `Runtime::spawn_blocking()` ([#2980])。
- sync: `watch::Sender::is_closed()` ([#2991])。

[#2884]: https://github.com/tokio-rs/tokio/pull/2884
[#2980]: https://github.com/tokio-rs/tokio/pull/2980
[#2991]: https://github.com/tokio-rs/tokio/pull/2991
[#3003]: https://github.com/tokio-rs/tokio/pull/3003
[#3009]: https://github.com/tokio-rs/tokio/pull/3009
[#3019]: https://github.com/tokio-rs/tokio/pull/3019

## 0.3.0 (2020年10月15日)

这代表一个 1.0 测试版。API 经过打磨并面向未来。未包含在 1.0 稳定版中的 API 已被移除。

最大的变化是：

- I/O 驱动程序内部重写。Windows 实现包含重大更改。
- 运行时 API 经过打磨，特别是与功能标志组合的交互方式。
- 简化了功能标志
  - `rt-core` 和 `rt-util` 合并为 `rt`
  - `rt-threaded` 重命名为 `rt-multi-thread` 以匹配构建器 API
  - `tcp`、`udp`、`uds`、`dns` 合并为 `net`。
  - `parking_lot` 包含在 `full` 中

### 变更

- meta: 最低支持的 Rust 版本现在是 1.45。
- io: `AsyncRead` trait 现在接受 `ReadBuf` 以安全地处理读取到未初始化的内存 ([#2758])。
- io: 内部 I/O 驱动程序存储现在能够压缩 ([#2757])。
- rt: `Runtime::block_on` 现在接受 `&self` ([#2782])。
- sync: `watch` 重构以解耦接收更改通知和接收值 ([#2814], [#2806])。
- sync: `Notify::notify` 重命名为 `notify_one` ([#2822])。
- process: `Child::kill` 现在是一个清理僵尸进程的 `async fn` ([#2823])。
- sync: 尽可能使用 `const fn` 构造函数 ([#2833], [#2790])
- signal: 减少跨线程通知 ([#2835])。
- net: tcp,udp,uds 类型支持使用 `&self` 的操作 ([#2828], [#2919], [#2934])。
- sync: 阻塞 `mpsc` 通道支持使用 `&self` 的 `send` ([#2861])。
- time: 将 `delay_for` 和 `delay_until` 重命名为 `sleep` 和 `sleep_until` ([#2826])。
- io: 升级到 `mio` 0.7 ([#2893])。
- io: `AsyncSeek` trait 进行了调整 ([#2885])。
- fs: `File` 操作接受 `&self` ([#2930])。
- rt: 运行时 API 和 `#[tokio::main]` 宏的打磨 ([#2876])
- rt: `Runtime::enter` 使用 RAII 守卫而不是闭包 ([#2954])。
- net: 所有套接字上的 `from_std` 函数不再将套接字设置为非阻塞模式 ([#2893])

### 已添加

- sync: 将 `map` 函数添加到锁守卫 ([#2445])。
- sync: 将 `blocking_recv` 和 `blocking_send` fn 添加到 `mpsc` 以在 Tokio 之外使用 ([#2685])。
- rt: `Builder::thread_name_fn` 用于配置线程名称 ([#1921])。
- fs: 为 `File` 实现 `FromRawFd` 和 `FromRawHandle` ([#2792])。
- process: `Child::wait` 和 `Child::try_wait` ([#2796])。
- rt: 支持配置线程保活时长 ([#2809])。
- rt: `task::JoinHandle::abort` 强制取消生成的任务 ([#2474])。
- sync: `RwLock` 写守卫到读守卫的降级 ([#2733])。
- net: 将接受 `&self` 的 `poll_*` 函数添加到所有网络类型 ([#2845])
- sync: `Mutex`、`RwLock` 的 `get_mut()` ([#2856])。
- sync: `mpsc::Sender::closed()` 等待 `Receiver` 半部关闭 ([#2840])。
- sync: 如果 `Receiver` 半部已关闭，`mpsc::Sender::is_closed()` 返回 true ([#2726])。
- stream: 将 `iter` 和 `iter_mut` 添加到 `StreamMap` ([#2890])。
- net: 在 windows 上实现 `AsRawSocket` ([#2911])。
- net: `TcpSocket` 创建一个未绑定或监听的套接字 ([#2920])。

### 已移除

- io: 从 `AsyncRead`、`AsyncWrite` trait 中移除了向量化操作 ([#2882])。
- io: 从公共 API 中移除了 `mio`。`PollEvented` 和 `Registration` 被移除 ([#2893])。
- io: 从公共 API 中移除 `bytes`。`Buf` 和 `BufMut` 的实现被移除 ([#2908])。
- time: `DelayQueue` 移至 `tokio-util` ([#2897])。

### 已修复

- io: windows 上的 `stdout` 和 `stderr` 缓冲 ([#2734])。

[#1921]: https://github.com/tokio-rs/tokio/pull/1921
[#2445]: https://github.com/tokio-rs/tokio/pull/2445
[#2474]: https://github.com/tokio-rs/tokio/pull/2474
[#2685]: https://github.com/tokio-rs/tokio/pull/2685
[#2726]: https://github.com/tokio-rs/tokio/pull/2726
[#2733]: https://github.com/tokio-rs/tokio/pull/2733
[#2734]: https://github.com/tokio-rs/tokio/pull/2734
[#2757]: https://github.com/tokio-rs/tokio/pull/2757
[#2758]: https://github.com/tokio-rs/tokio/pull/2758
[#2782]: https://github.com/tokio-rs/tokio/pull/2782
[#2790]: https://github.com/tokio-rs/tokio/pull/2790
[#2792]: https://github.com/tokio-rs/tokio/pull/2792
[#2796]: https://github.com/tokio-rs/tokio/pull/2796
[#2806]: https://github.com/tokio-rs/tokio/pull/2806
[#2809]: https://github.com/tokio-rs/tokio/pull/2809
[#2814]: https://github.com/tokio-rs/tokio/pull/2814
[#2822]: https://github.com/tokio-rs/tokio/pull/2822
[#2823]: https://github.com/tokio-rs/tokio/pull/2823
[#2826]: https://github.com/tokio-rs/tokio/pull/2826
[#2828]: https://github.com/tokio-rs/tokio/pull/2828
[#2833]: https://github.com/tokio-rs/tokio/pull/2833
[#2835]: https://github.com/tokio-rs/tokio/pull/2835
[#2840]: https://github.com/tokio-rs/tokio/pull/2840
[#2845]: https://github.com/tokio-rs/tokio/pull/2845
[#2856]: https://github.com/tokio-rs/tokio/pull/2856
[#2861]: https://github.com/tokio-rs/tokio/pull/2861
[#2876]: https://github.com/tokio-rs/tokio/pull/2876
[#2882]: https://github.com/tokio-rs/tokio/pull/2882
[#2885]: https://github.com/tokio-rs/tokio/pull/2885
[#2890]: https://github.com/tokio-rs/tokio/pull/2890
[#2893]: https://github.com/tokio-rs/tokio/pull/2893
[#2897]: https://github.com/tokio-rs/tokio/pull/2897
[#2908]: https://github.com/tokio-rs/tokio/pull/2908
[#2911]: https://github.com/tokio-rs/tokio/pull/2911
[#2919]: https://github.com/tokio-rs/tokio/pull/2919
[#2920]: https://github.com/tokio-rs/tokio/pull/2920
[#2930]: https://github.com/tokio-rs/tokio/pull/2930
[#2934]: https://github.com/tokio-rs/tokio/pull/2934
[#2954]: https://github.com/tokio-rs/tokio/pull/2954

## 0.2.22 (2020年7月21日)

### 修复

- docs: 杂项改进 ([#2572], [#2658], [#2663], [#2656], [#2647], [#2630], [#2487], [#2621],
  [#2624], [#2600], [#2623], [#2622], [#2577], [#2569], [#2589], [#2575], [#2540], [#2564], [#2567],
  [#2520], [#2521], [#2493])
- rt: 允许在 `block_in_place` 调用内部调用 `block_on`，而这些调用本身在 `block_on` 内部 ([#2645])
- net: 修复在丢弃 `TcpStream` `OwnedWriteHalf` 时的非可移植行为 ([#2597])
- io: 通过直接在堆上分配大缓冲区来改善堆栈使用 ([#2634])
- io: 修复 `AsyncReadExt::read_buf` 和 `AsyncWriteExt::write_buf` 中的不健全 pin 投影 ([#2612])
- io: 修复 `AsyncRead` 实现者的不必要清零 ([#2525])
- io: 修复 `BufReader` 未正确转发 `poll_write_buf` 的问题 ([#2654])
- io: 修复 `AsyncReadExt::read_line` 中的 panic ([#2541])

### 变更

- coop: 返回 `Poll::Pending` 不再减少任务预算 ([#2549])

### 已添加

- io: `AsyncReadExt` 和 `AsyncWriteExt` 方法的小端变体 ([#1915])
- task: 为生成的任务添加 [`tracing`] 检测 ([#2655])
- sync: 允许在 `Mutex` 和 `RwLock` 中使用未定大小的类型 (通过 `default` 构造函数) ([#2615])
- net: 为 `&[SocketAddr]` 添加 `ToSocketAddrs` 实现 ([#2604])
- fs: 为 `OpenOptions` 添加 `OpenOptionsExt` ([#2515])
- fs: 添加 `DirBuilder` ([#2524])

[`tracing`]: https://crates.io/crates/tracing
[#1915]: https://github.com/tokio-rs/tokio/pull/1915
[#2487]: https://github.com/tokio-rs/tokio/pull/2487
[#2493]: https://github.com/tokio-rs/tokio/pull/2493
[#2515]: https://github.com/tokio-rs/tokio/pull/2515
[#2520]: https://github.com/tokio-rs/tokio/pull/2520
[#2521]: https://github.com/tokio-rs/tokio/pull/2521
[#2524]: https://github.com/tokio-rs/tokio/pull/2524
[#2525]: https://github.com/tokio-rs/tokio/pull/2525
[#2540]: https://github.com/tokio-rs/tokio/pull/2540
[#2541]: https://github.com/tokio-rs/tokio/pull/2541
[#2549]: https://github.com/tokio-rs/tokio/pull/2549
[#2564]: https://github.com/tokio-rs/tokio/pull/2564
[#2567]: https://github.com/tokio-rs/tokio/pull/2567
[#2569]: https://github.com/tokio-rs/tokio/pull/2569
[#2572]: https://github.com/tokio-rs/tokio/pull/2572
[#2575]: https://github.com/tokio-rs/tokio/pull/2575
[#2577]: https://github.com/tokio-rs/tokio/pull/2577
[#2589]: https://github.com/tokio-rs/tokio/pull/2589
[#2597]: https://github.com/tokio-rs/tokio/pull/2597
[#2600]: https://github.com/tokio-rs/tokio/pull/2600
[#2604]: https://github.com/tokio-rs/tokio/pull/2604
[#2612]: https://github.com/tokio-rs/tokio/pull/2612
[#2615]: https://github.com/tokio-rs/tokio/pull/2615
[#2621]: https://github.com/tokio-rs/tokio/pull/2621
[#2622]: https://github.com/tokio-rs/tokio/pull/2622
[#2623]: https://github.com/tokio-rs/tokio/pull/2623
[#2624]: https://github.com/tokio-rs/tokio/pull/2624
[#2630]: https://github.com/tokio-rs/tokio/pull/2630
[#2634]: https://github.com/tokio-rs/tokio/pull/2634
[#2645]: https://github.com/tokio-rs/tokio/pull/2645
[#2647]: https://github.com/tokio-rs/tokio/pull/2647
[#2654]: https://github.com/tokio-rs/tokio/pull/2654
[#2655]: https://github.com/tokio-rs/tokio/pull/2655
[#2656]: https://github.com/tokio-rs/tokio/pull/2656
[#2658]: https://github.com/tokio-rs/tokio/pull/2658
[#2663]: https://github.com/tokio-rs/tokio/pull/2663

## 0.2.21 (2020年5月13日)

### 修复

- macros: 在宏展开中消除内置 `#[test]` 属性的歧义 ([#2503])
- rt: `LocalSet` 和任务预算 ([#2462])。
- rt: 使用 `block_in_place` 的任务预算 ([#2502])。
- sync: 在不发送值的情况下释放 `broadcast` 通道内存 ([#2509])。
- time: 在将 `Delay` 重置为过去的时间时通知 ([#2290])

### 已添加

- io: `Lines` 的 `get_mut`、`get_ref` 和 `into_inner` ([#2450])。
- io: `mio::Ready` 参数到 `PollEvented` ([#2419])。
- os: illumos 支持 ([#2486])。
- rt: `Handle::spawn_blocking` ([#2501])。
- sync: `Arc<Mutex<T>>` 的 `OwnedMutexGuard` ([#2455])。

[#2290]: https://github.com/tokio-rs/tokio/pull/2290
[#2419]: https://github.com/tokio-rs/tokio/pull/2419
[#2450]: https://github.com/tokio-rs/tokio/pull/2450
[#2455]: https://github.com/tokio-rs/tokio/pull/2455
[#2462]: https://github.com/tokio-rs/tokio/pull/2462
[#2486]: https://github.com/tokio-rs/tokio/pull/2486
[#2501]: https://github.com/tokio-rs/tokio/pull/2501
[#2502]: https://github.com/tokio-rs/tokio/pull/2502
[#2503]: https://github.com/tokio-rs/tokio/pull/2503
[#2509]: https://github.com/tokio-rs/tokio/pull/2509

## 0.2.20 (2020年4月28日)

### 修复

- sync: `broadcast` 关闭通道不再需要容量 ([#2448])。
- rt: 在使用小于 CPU 数量的 `max_threads` 配置运行时时的回归 ([#2457])。

[#2448]: https://github.com/tokio-rs/tokio/pull/2448
[#2457]: https://github.com/tokio-rs/tokio/pull/2457

## 0.2.19 (2020年4月24日)

### 修复

- docs: 杂项改进 ([#2400], [#2405], [#2414], [#2420], [#2423], [#2426], [#2427], [#2434], [#2436], [#2440])。
- rt: 在更多上下文中支持 `block_in_place` ([#2409], [#2410])。
- stream: 在使用 `size_hint()` 时，`merge()` 和 `chain()` 中不会 panic ([#2430])。
- task: 在定义任务本地时包含可见性修饰符 ([#2416])。

### 已添加

- rt: `runtime::Handle::block_on` ([#2437])。
- sync: 拥有的 `Semaphore` 许可 ([#2421])。
- tcp: 拥有的拆分 ([#2270])。

[#2270]: https://github.com/tokio-rs/tokio/pull/2270
[#2400]: https://github.com/tokio-rs/tokio/pull/2400
[#2405]: https://github.com/tokio-rs/tokio/pull/2405
[#2409]: https://github.com/tokio-rs/tokio/pull/2409
[#2410]: https://github.com/tokio-rs/tokio/pull/2410
[#2414]: https://github.com/tokio-rs/tokio/pull/2414
[#2416]: https://github.com/tokio-rs/tokio/pull/2416
[#2420]: https://github.com/tokio-rs/tokio/pull/2420
[#2421]: https://github.com/tokio-rs/tokio/pull/2421
[#2423]: https://github.com/tokio-rs/tokio/pull/2423
[#2426]: https://github.com/tokio-rs/tokio/pull/2426
[#2427]: https://github.com/tokio-rs/tokio/pull/2427
[#2430]: https://github.com/tokio-rs/tokio/pull/2430
[#2434]: https://github.com/tokio-rs/tokio/pull/2434
[#2436]: https://github.com/tokio-rs/tokio/pull/2436
[#2437]: https://github.com/tokio-rs/tokio/pull/2437
[#2440]: https://github.com/tokio-rs/tokio/pull/2440

## 0.2.18 (2020年4月12日)

### 修复

- task: `LocalSet` 被错误地标记为 `Send` ([#2398])
- io: 在 `write_int` 中正确报告 `WriteZero` 失败 ([#2334])

[#2334]: https://github.com/tokio-rs/tokio/pull/2334
[#2398]: https://github.com/tokio-rs/tokio/pull/2398

## 0.2.17 (2020年4月9日)

### 修复

- rt: 工作窃取队列中的 bug ([#2387])

### 变更

- rt: 线程池默认使用逻辑 CPU 数量而不是物理 CPU 数量 ([#2391])

[#2387]: https://github.com/tokio-rs/tokio/pull/2387
[#2391]: https://github.com/tokio-rs/tokio/pull/2391

## 0.2.16 (2020年4月3日)

### 修复

- sync: 修复了一个回归问题，即 `Mutex`、`Semaphore` 和 `RwLock` future 不再实现 `Sync` ([#2375])
- fs: 修复 `fs::copy` 未复制文件权限的问题 ([#2354])

### 已添加

- time: 为 `delay_queue::Expired` 添加了 `deadline` 方法 ([#2300])
- io: 添加了 `StreamReader` ([#2052])

[#2052]: https://github.com/tokio-rs/tokio/pull/2052
[#2300]: https://github.com/tokio-rs/tokio/pull/2300
[#2354]: https://github.com/tokio-rs/tokio/pull/2354
[#2375]: https://github.com/tokio-rs/tokio/pull/2375

## 0.2.15 (2020年4月2日)

### 修复

- rt: 修复队列回归 ([#2362])。

### 已添加

- sync: 为 `mpsc::Sender` 添加 disarm ([#2358])。

[#2358]: https://github.com/tokio-rs/tokio/pull/2358
[#2362]: https://github.com/tokio-rs/tokio/pull/2362

## 0.2.14 (2020年4月1日)

### 修复

- rt: 调度器中的并发 bug ([#2273])。
- rt: shell 运行时的并发 bug ([#2333])。
- test-util: 正确暂停/恢复时间 ([#2253])。
- time: `DelayQueue` 在 `insert` 后正确唤醒 ([#2285])。

### 已添加

- io: 为 std io 类型实现 `RawFd`、`AsRawHandle` ([#2335])。
- rt: 自动协作任务让步 ([#2160], [#2343], [#2349])。
- sync: `RwLock::into_inner` ([#2321])。

### 已更改

- sync: 重写了信号量、互斥锁的内部实现以避免分配 ([#2325])。

[#2160]: https://github.com/tokio-rs/tokio/pull/2160
[#2253]: https://github.com/tokio-rs/tokio/pull/2253
[#2273]: https://github.com/tokio-rs/tokio/pull/2273
[#2285]: https://github.com/tokio-rs/tokio/pull/2285
[#2321]: https://github.com/tokio-rs/tokio/pull/2321
[#2325]: https://github.com/tokio-rs/tokio/pull/2325
[#2333]: https://github.com/tokio-rs/tokio/pull/2333
[#2335]: https://github.com/tokio-rs/tokio/pull/2335
[#2343]: https://github.com/tokio-rs/tokio/pull/2343
[#2349]: https://github.com/tokio-rs/tokio/pull/2349

## 0.2.13 (2020年2月28日)

### 修复

- macros: `pin!` 中未解析的导入 ([#2281])。

[#2281]: https://github.com/tokio-rs/tokio/pull/2281

## 0.2.12 (2020年2月27日)

### 修复

- net: `UnixStream::poll_shutdown` 应调用 `shutdown(Write)` ([#2245])。
- process: 在 `EPOLLERR` 上唤醒读和写 ([#2218])。
- rt: 使用 `block_in_place` 并关闭运行时可能出现的死锁 ([#2119])。
- rt: 仅在未指定 `core_threads` 时检测 CPU 数量 ([#2238])。
- sync: 减少 `watch::Receiver` 结构体大小 ([#2191])。
- time: 在设置 `$MAX-1` 的延迟时成功 ([#2184])。
- time: 避免在插入新延迟后需要轮询 `DelayQueue` ([#2217])。

### 已添加

- macros: `pin!` 的变体，可分配给标识符并固定 ([#2274])。
- net: 为 `Listener` 类型实现 `Stream` ([#2275])。
- rt: `Runtime::shutdown_timeout` 等待运行时关闭指定的持续时间 ([#2186])。
- stream: `StreamMap` 合并流，并可在运行时插入/移除流 ([#2185])。
- stream: `StreamExt::skip()` 跳过固定数量的项 ([#2204])。
- stream: `StreamExt::skip_while()` 根据谓词跳过项 ([#2205])。
- sync: `Notify` 提供基本的 `async` / `await` 任务通知 ([#2210])。
- sync: `Mutex::into_inner` 检索受保护的数据 ([#2250])。
- sync: `mpsc::Sender::send_timeout` 发送，等待最多指定的持续时间以获得通道容量 ([#2227])。
- time: 为 `Instant` 实现 `Ord` 和 `Hash` ([#2239])。

[#2119]: https://github.com/tokio-rs/tokio/pull/2119
[#2184]: https://github.com/tokio-rs/tokio/pull/2184
[#2185]: https://github.com/tokio-rs/tokio/pull/2185
[#2186]: https://github.com/tokio-rs/tokio/pull/2186
[#2191]: https://github.com/tokio-rs/tokio/pull/2191
[#2204]: https://github.com/tokio-rs/tokio/pull/2204
[#2205]: https://github.com/tokio-rs/tokio/pull/2205
[#2210]: https://github.com/tokio-rs/tokio/pull/2210
[#2217]: https://github.com/tokio-rs/tokio/pull/2217
[#2218]: https://github.com/tokio-rs/tokio/pull/2218
[#2227]: https://github.com/tokio-rs/tokio/pull/2227
[#2238]: https://github.com/tokio-rs/tokio/pull/2238
[#2239]: https://github.com/tokio-rs/tokio/pull/2239
[#2245]: https://github.com/tokio-rs/tokio/pull/2245
[#2250]: https://github.com/tokio-rs/tokio/pull/2250
[#2274]: https://github.com/tokio-rs/tokio/pull/2274
[#2275]: https://github.com/tokio-rs/tokio/pull/2275

## 0.2.11 (2020年1月27日)

### 修复

- docs: 杂项修复和调整 ([#2155], [#2103], [#2027], [#2167], [#2175])。
- macros: 处理 `#[tokio::main]` 方法中的泛型 ([#2177])。
- sync: `broadcast` 可能丢失通知 ([#2135])。
- rt: 改进“无运行时”的 panic 消息 ([#2145])。

### 已添加

- 可选支持内部使用 `parking_lot` ([#2164])。
- fs: `fs::copy`，`std::fs::copy` 的异步版本 ([#2079])。
- macros: `select!` 等待第一个分支完成 ([#2152])。
- macros: `join!` 等待所有分支完成 ([#2158])。
- macros: `try_join!` 等待所有分支完成或第一个错误 ([#2169])。
- macros: `pin!` 将值固定到堆栈上 ([#2163])。
- net: `ReadHalf::poll()` 和 `ReadHalf::poll_peak` ([#2151])
- stream: `StreamExt::timeout()` 设置每项的最大持续时间 ([#2149])。
- stream: `StreamExt::fold()` 应用一个函数，产生单个值。 ([#2122])。
- sync: 为 `oneshot::RecvError` 实现 `Eq`、`PartialEq` ([#2168])。
- task: 用于检查 `JoinError` 原因的方法 ([#2051])。

[#2027]: https://github.com/tokio-rs/tokio/pull/2027
[#2051]: https://github.com/tokio-rs/tokio/pull/2051
[#2079]: https://github.com/tokio-rs/tokio/pull/2079
[#2103]: https://github.com/tokio-rs/tokio/pull/2103
[#2122]: https://github.com/tokio-rs/tokio/pull/2122
[#2135]: https://github.com/tokio-rs/tokio/pull/2135
[#2145]: https://github.com/tokio-rs/tokio/pull/2145
[#2149]: https://github.com/tokio-rs/tokio/pull/2149
[#2151]: https://github.com/tokio-rs/tokio/pull/2151
[#2152]: https://github.com/tokio-rs/tokio/pull/2152
[#2155]: https://github.com/tokio-rs/tokio/pull/2155
[#2158]: https://github.com/tokio-rs/tokio/pull/2158
[#2163]: https://github.com/tokio-rs/tokio/pull/2163
[#2164]: https://github.com/tokio-rs/tokio/pull/2164
[#2167]: https://github.com/tokio-rs/tokio