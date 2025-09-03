# 更新日志

每个版本中对 Tokio 库所做的所有显著更改的详细记录。

## 1.47.1 (2025年8月1日)

### 已修复

- process：修复了由虚假 pidfd 唤醒引起的 panic ([#7494](https://github.com/tokio-rs/tokio/pull/7494))
- sync：修复了 `SetOnce` 文档中 Python `asyncio.Event` 的损坏链接 ([#7485](https://github.com/tokio-rs/tokio/pull/7485))

## 1.47.0 (2025年7月25日)

此版本将用于协作调度的 `poll_proceed` 和 `cooperative` 添加到 `coop` 模块，将 `SetOnce` 添加到 `sync` 模块（其功能类似于 `std::sync::OnceLock`），并添加了一个新方法 `sync::Notify::notified_owned()`，该方法返回一个不带生命周期参数的 `OwnedNotified`。

### 已添加

- coop：添加 `cooperative` 和 `poll_proceed` ([#7405](https://github.com/tokio-rs/tokio/pull/7405))
- sync：添加 `SetOnce` ([#7418](https://github.com/tokio-rs/tokio/pull/7418))
- sync：添加 `sync::Notify::notified_owned()` ([#7465](https://github.com/tokio-rs/tokio/pull/7465))

### 已更改

- deps：将 windows-sys 从 0.52 升级到 0.59 ([#7117](https://github.com/tokio-rs/tokio/pull/7117))
- deps：更新到 socket2 v0.6 ([#7443](https://github.com/tokio-rs/tokio/pull/7443))
- sync：提升 `AtomicWaker::wake` 的性能 ([#7450](https://github.com/tokio-rs/tokio/pull/7450))

### 已记录

- metrics：修复了某些指标列出的功能要求 ([#7449](https://github.com/tokio-rs/tokio/pull/7449))
- runtime：改进了 `Readiness<'_>` 的安全注释 ([#7415](https://github.com/tokio-rs/tokio/pull/7415))

## 1.46.1 (2025年7月4日)

此版本修复了使用 `tokio::spawn` 而非 `Runtime::spawn` 生成的任务在运行时任务钩子中的生成位置不正确的问题。此问题仅影响 `TaskMeta::spawned_at` 中的生成位置，不影响 Tracing 事件中的任务位置。

### 不稳定

- runtime：添加 `TaskMeta::spawn_location` 以跟踪任务的生成位置 ([#7440](https://github.com/tokio-rs/tokio/pull/7440))

## 1.46.0 (2025年7月2日)

### 已修复

- net：修复了在 macOS 上 `TcpStream::shutdown` 错误地返回一个 error 的问题 ([#7290](https://github.com/tokio-rs/tokio/pull/7290))

### 已添加

- sync：`mpsc::OwnedPermit::{same_channel, same_channel_as_sender}` 方法 ([#7389](https://github.com/tokio-rs/tokio/pull/7389))
- macros：为 `join!` 和 `try_join!` 添加 `biased` 选项，类似于 `select!` ([#7307](https://github.com/tokio-rs/tokio/pull/7307))
- net：支持 cygwin ([#7393](https://github.com/tokio-rs/tokio/pull/7393))
- net：在 Android 上支持 `pipe::OpenOptions::read_write` ([#7426](https://github.com/tokio-rs/tokio/pull/7426))
- net：为 `net::unix::SocketAddr` 添加 `Clone` 实现 ([#7422](https://github.com/tokio-rs/tokio/pull/7422))

### 已更改

- runtime：在操作 `queue::Local<T>` 时消除了不必要的 lfence ([#7340](https://github.com/tokio-rs/tokio/pull/7340))
- task：禁止在 `LocalSet::{poll,drop}` 中阻塞 ([#7372](https://github.com/tokio-rs/tokio/pull/7372))

### 不稳定

- runtime：添加 `TaskMeta::spawn_location` 以跟踪任务的生成位置 ([#7417](https://github.com/tokio-rs/tokio/pull/7417))
- runtime：从 `runtime::Builder::build_local` 的 `LocalOptions` 参数中移除了借用 ([#7346](https://github.com/tokio-rs/tokio/pull/7346))

### 已记录

- io：阐明了在未使用 `start_seek` 时寻址的行为 ([#7366](https://github.com/tokio-rs/tokio/pull/7366))
- io：记录了 `AsyncWriteExt::flush` 的取消安全性 ([#7364](https://github.com/tokio-rs/tokio/pull/7364))
- net：修复了 `recv_buffer_size` 方法的文档 ([#7336](https://github.com/tokio-rs/tokio/pull/7336))
- net：修复了 `TcpSocket` 文档中 `RawFd` 的损坏链接 ([#7416](https://github.com/tokio-rs/tokio/pull/7416))
- net：将 `AsRawFd` 文档链接更新到当前的 Rust stdlib 位置 ([#7429](https://github.com/tokio-rs/tokio/pull/7429))
- readme：修复了反应器描述中的双句点问题 ([#7363](https://github.com/tokio-rs/tokio/pull/7363))
- runtime：添加文档说明 `on_*_task_poll` 是不稳定的 ([#7311](https://github.com/tokio-rs/tokio/pull/7311))
- sync：更新了关于分配失败的广播文档 ([#7352](https://github.com/tokio-rs/tokio/pull/7352))
- time：为 `time::advance` 添加了一个缺失的 panic 场景 ([#7394](https://github.com/tokio-rs/tokio/pull/7394))

## 1.45.1 (2025年5月24日)

此版本修复了 wasm32-unknown-unknown 目标上的一个回归问题，即之前因调用 `Instant::now()` 而未发生 panic 的代码开始失败。这是由于第一个基于时间的指标已稳定所致。

### 已修复

- 在 wasm32-unknown-unknown 上禁用基于时间的指标 ([#7322](https://github.com/tokio-rs/tokio/pull/7322))

## 1.45.0 (2025年5月5日)

### 已添加

- metrics：稳定 `worker_total_busy_duration`、`worker_park_count` 和 `worker_unpark_count` ([#6899](https://github.com/tokio-rs/tokio/pull/6899), [#7276](https://github.com/tokio-rs/tokio/pull/7276))
- process：添加 `Command::spawn_with` ([#7249](https://github.com/tokio-rs/tokio/pull/7249))

### 已更改

- io：某些 trait 实现不再需要 `Unpin` ([#7204](https://github.com/tokio-rs/tokio/pull/7204))
- rt：将 `runtime::Handle` 标记为 unwind safe ([#7230](https://github.com/tokio-rs/tokio/pull/7230))
- time：恢复内部 sharding 实现 ([#7226](https://github.com/tokio-rs/tokio/pull/7226))

### 不稳定

- rt：移除备用的多线程运行时 ([#7275](https://github.com/tokio-rs/tokio/pull/7275))

## 1.44.2 (2025年4月5日)

此版本修复了广播通道中的一个健全性问题。该通道接受 `Send` 但 `!Sync` 的值。之前，该通道在未同步的情况下对这些值调用 `clone()`。此版本通过同步对 `.clone()` 的调用来修复此问题（感谢 Austin Bonander 发现并报告此问题）。

### 已修复

- sync：在广播通道中同步 `clone()` 调用 ([#7232](https://github.com/tokio-rs/tokio/pull/7232))

## 1.44.1 (2025年3月13日)

### 已修复

- rt：在 `block_in_place` 上下文中跳过延迟队列 ([#7216](https://github.com/tokio-rs/tokio/pull/7216))

## 1.44.0 (2025年3月7日)

此版本更改了套接字上的 `from_std` 方法，以便在提供阻塞套接字时会发生 panic。我们确定此更改不是一个破坏性更改，因为 Tokio 不应使用阻塞套接字进行操作。这样做会导致运行时挂起，应被视为一个 bug。意外地将阻塞套接字传递给 Tokio 是最常见的用户错误之一。如果此更改给您带来问题，请在 [#7172](https://github.com/tokio-rs/tokio/pull/7172) 中评论。

### 已添加

- coop：添加 `task::coop` 模块 ([#7116](https://github.com/tokio-rs/tokio/pull/7116))
- process：添加 `Command::get_kill_on_drop()` ([#7086](https://github.com/tokio-rs/tokio/pull/7086))
- sync：添加 `broadcast::Sender::closed` ([#6685](https://github.com/tokio-rs/tokio/pull/6685), [#7090](https://github.com/tokio-rs/tokio/pull/7090))
- sync：添加 `broadcast::WeakSender` ([#7100](https://github.com/tokio-rs/tokio/pull/7100))
- sync：添加 `oneshot::Receiver::is_empty()` ([#7153](https://github.com/tokio-rs/tokio/pull/7153))
- sync：添加 `oneshot::Receiver::is_terminated()` ([#7152](https://github.com/tokio-rs/tokio/pull/7152))

### 已修复

- fs：`File` 上的空读取不应启动后台读取 ([#7139](https://github.com/tokio-rs/tokio/pull/7139))
- process：在已退出的子进程上调用 `start_kill` 不应失败 ([#7160](https://github.com/tokio-rs/tokio/pull/7160))
- signal：修复了 Windows 上的 `CTRL_CLOSE`、`CTRL_LOGOFF` 和 `CTRL_SHUTDOWN` ([#7122](https://github.com/tokio-rs/tokio/pull/7122))
- sync：在 mpsc drop 期间正确处理 panic ([#7094](https://github.com/tokio-rs/tokio/pull/7094))

### 更改

- runtime：清理注册集中的幻数 ([#7112](https://github.com/tokio-rs/tokio/pull/7112))
- coop：使 coop 使用 waker 延迟策略进行让步 ([#7185](https://github.com/tokio-rs/tokio/pull/7185))
- macros：使 `select!` 具有预算感知能力 ([#7164](https://github.com/tokio-rs/tokio/pull/7164))
- net：在将阻塞套接字传递给 `from_std` 时 panic ([#7166](https://github.com/tokio-rs/tokio/pull/7166))
- io：清理缓冲区转换 ([#7142](https://github.com/tokio-rs/tokio/pull/7142))

### 对不稳定 API 的更改

- rt：添加任务轮询前后的回调 ([#7120](https://github.com/tokio-rs/tokio/pull/7120))
- tracing：使任务 tracing API 成为不稳定的公共 API ([#6972](https://github.com/tokio-rs/tokio/pull/6972))

### 已记录

- docs：修复顶级文档中章节的嵌套问题 ([#7159](https://github.com/tokio-rs/tokio/pull/7159))
- fs：重命名符号链接和硬链接的参数名称 ([#7143](https://github.com/tokio-rs/tokio/pull/7143))
- io：在 simplex 文档测试中交换 reader/writer ([#7176](https://github.com/tokio-rs/tokio/pull/7176))
- macros：关于 `select!` 替代方案的文档 ([#7110](https://github.com/tokio-rs/tokio/pull/7110))
- net：重命名 `send_to` 的参数 ([#7146](https://github.com/tokio-rs/tokio/pull/7146))
- process：添加读取 `Child` stdout 的示例 ([#7141](https://github.com/tokio-rs/tokio/pull/7141))
- process：阐明 `Child::kill` 的行为 ([#7162](https://github.com/tokio-rs/tokio/pull/7162))
- process：修复 `ChildStdin` 结构文档注释的语法错误 ([#7192](https://github.com/tokio-rs/tokio/pull/7192))
- runtime：统一使用 `worker_threads` 而非 `core_threads` ([#7186](https://github.com/tokio-rs/tokio/pull/7186))

## 1.43.2 (2025年8月1日)

### 已修复

- process：修复了由虚假 pidfd 唤醒引起的 panic ([#7494](https://github.com/tokio-rs/tokio/pull/7494))

## 1.43.1 (2025年4月5日)

此版本修复了广播通道中的一个健全性问题。该通道接受 `Send` 但 `!Sync` 的值。之前，该通道在未同步的情况下对这些值调用 `clone()`。此版本通过同步对 `.clone()` 的调用来修复此问题（感谢 Austin Bonander 发现并报告此问题）。

### 已修复

- sync：在广播通道中同步 `clone()` 调用 ([#7232](https://github.com/tokio-rs/tokio/pull/7232))

## 1.43.0 (2025年1月8日)

### 已添加

- net：添加 `UdpSocket::peek` 方法 ([#7068](https://github.com/tokio-rs/tokio/pull/7068))
- net：添加对 Haiku OS 的支持 ([#7042](https://github.com/tokio-rs/tokio/pull/7042))
- process：添加 `Command::into_std()` ([#7014](https://github.com/tokio-rs/tokio/pull/7014))
- signal：在 illumos 上添加 `SignalKind::info` ([#6995](https://github.com/tokio-rs/tokio/pull/6995))
- signal：在 illumos 上添加对实时信号的支持 ([#7029](https://github.com/tokio-rs/tokio/pull/7029))

### 已修复

- io：在 `Blocking` 中初始化向量之前不要调用 `set_len` ([#7054](https://github.com/tokio-rs/tokio/pull/7054))
- macros：在 `#[tokio::main]` 中抑制 `clippy::needless_return` ([#6874](https://github.com/tokio-rs/tokio/pull/6874))
- runtime：修复 WebAssembly 上的线程停放问题 ([#7041](https://github.com/tokio-rs/tokio/pull/7041))

### 更改

- chore：为 `unsync_load` 使用非同步加载 ([#7073](https://github.com/tokio-rs/tokio/pull/7073))
- io：在 `Repeat` 读取实现中使用 `Buf::put_bytes` ([#7055](https://github.com/tokio-rs/tokio/pull/7055))
- task：尽早丢弃任务的 join waker ([#6986](https://github.com/tokio-rs/tokio/pull/6986))

### 对不稳定 API 的更改

- metrics：提高 H2Histogram 配置的灵活性 ([#6963](https://github.com/tokio-rs/tokio/pull/6963))
- taskdump：为回溯添加访问器方法 ([#6975](https://github.com/tokio-rs/tokio/pull/6975))

### 已记录

- io：阐明 `ReadBuf::uninit` 也允许初始化缓冲区 ([#7053](https://github.com/tokio-rs/tokio/pull/7053))
- net：修复 `TcpStream::try_write_vectored` 文档中的歧义 ([#7067](https://github.com/tokio-rs/tokio/pull/7067))
- runtime：修复 `LocalRuntime` 文档链接 ([#7074](https://github.com/tokio-rs/tokio/pull/7074))
- sync：扩展 `watch::Receiver::wait_for` 的文档 ([#7038](https://github.com/tokio-rs/tokio/pull/7038))
- sync：修复 `OnceCell` 文档中的拼写错误 ([#7047](https://github.com/tokio-rs/tokio/pull/7047))

## 1.42.1 (2025年4月8日)

此版本修复了广播通道中的一个健全性问题。该通道接受 `Send` 但 `!Sync` 的值。之前，该通道在未同步的情况下对这些值调用 `clone()`。此版本通过同步对 `.clone()` 的调用来修复此问题（感谢 Austin Bonander 发现并报告此问题）。

### 已修复

- sync：在广播通道中同步 `clone()` 调用 ([#7232](https://github.com/tokio-rs/tokio/pull/7232))

## 1.42.0 (2024年12月3日)

### 已添加

- io：添加 `AsyncFd::{try_io, try_io_mut}` ([#6967](https://github.com/tokio-rs/tokio/pull/6967))

### 已修复

- io：避免在 RegistrationSet 中出现 `ptr->ref->ptr` 往返 ([#6929](https://github.com/tokio-rs/tokio/pull/6929))
- runtime：不要在 `block_in_place` 内部延迟 `yield_now` ([#6999](https://github.com/tokio-rs/tokio/pull/6999))

### 更改

- io：简化 io 就绪逻辑 ([#6966](https://github.com/tokio-rs/tokio/pull/6966))

### 已记录

- net：修复 `tokio::net::unix::{pid_t, gid_t, uid_t}` 的文档 ([#6791](https://github.com/tokio-rs/tokio/pull/6791))
- time：修复 `Instant` 文档中的拼写错误 ([#6982](https://github.com/tokio-rs/tokio/pull/6982))

## 1.41.1 (2024年11月7日)

### 已修复

- metrics：修复直方图桶数不正确的 bug ([#6957](https://github.com/tokio-rs/tokio/pull/6957))
- net：在文档中显示 `net::UdpSocket` 的 `net` 要求 ([#6938](https://github.com/tokio-rs/tokio/pull/6938))
- net：修复 `TcpStream` 内部注释中的拼写错误 ([#6944](https://github.com/tokio-rs/tokio/pull/6944))

## 1.41.0 (2024年10月22日)

### 已添加

- metrics：稳定 `global_queue_depth` ([#6854](https://github.com/tokio-rs/tokio/pull/6854), [#6918](https://github.com/tokio-rs/tokio/pull/6918))
- net：为 unix `SocketAddr` 添加转换 ([#6868](https://github.com/tokio-rs/tokio/pull/6868))
- sync：添加 `watch::Sender::sender_count` ([#6836](https://github.com/tokio-rs/tokio/pull/6836))
- sync：添加 `mpsc::Receiver::blocking_recv_many` ([#6867](https://github.com/tokio-rs/tokio/pull/6867))
- task：稳定 `Id` API ([#6793](https://github.com/tokio-rs/tokio/pull/6793), [#6891](https://github.com/tokio-rs/tokio/pull/6891))

### 已添加 (不稳定)

- metrics：添加 H2 Histogram 选项以提高直方图粒度 ([#6897](https://github.com/tokio-rs/tokio/pull/6897))
- metrics：重命名一些直方图 API ([#6924](https://github.com/tokio-rs/tokio/pull/6924))
- runtime：添加 `LocalRuntime` ([#6808](https://github.com/tokio-rs/tokio/pull/6808))

### 已更改

- runtime：在发布模式下装箱大于 16k 的 future ([#6826](https://github.com/tokio-rs/tokio/pull/6826))
- sync：为 `Notified` 添加 `#[must_use]` ([#6828](https://github.com/tokio-rs/tokio/pull/6828))
- sync：使 `watch` 协作 ([#6846](https://github.com/tokio-rs/tokio/pull/6846))
- sync：使 `broadcast::Receiver` 协作 ([#6870](https://github.com/tokio-rs/tokio/pull/6870))
- task：在 tracing 检测中添加任务大小 ([#6881](https://github.com/tokio-rs/tokio/pull/6881))
- wasm：为 `wasi` 目标启用 `cfg_fs` ([#6822](https://github.com/tokio-rs/tokio/pull/6822))

### 已修复

- net：修复 unix 套接字中抽象套接字路径的回归问题 ([#6838](https://github.com/tokio-rs/tokio/pull/6838))

### 已记录

- io：推荐将 `OwnedFd` 与 `AsyncFd` 一起使用 ([#6821](https://github.com/tokio-rs/tokio/pull/6821))
- io：记录 `AsyncFd` 方法的取消安全性 ([#6890](https://github.com/tokio-rs/tokio/pull/6890))
- macros：为 `join` 和 `try_join` 渲染更易于理解的文档 ([#6814](https://github.com/tokio-rs/tokio/pull/6814), [#6841](https://github.com/tokio-rs/tokio/pull/6841))
- net：修复 `TcpSocket::set_nodelay` 和 `TcpSocket::nodelay` 交换的示例 ([#6840](https://github.com/tokio-rs/tokio/pull/6840))
- sync：记录运行时兼容性 ([#6833](https://github.com/tokio-rs/tokio/pull/6833))

## 1.40.0 (2024年8月30日)

### 已添加

- io：添加 `util::SimplexStream` ([#6589](https://github.com/tokio-rs/tokio/pull/6589))
- process：稳定 `Command::process_group` ([#6731](https://github.com/tokio-rs/tokio/pull/6731))
- sync：添加 `{TrySendError,SendTimeoutError}::into_inner` ([#6755](https://github.com/tokio-rs/tokio/pull/6755))
- task：添加 `JoinSet::join_all` ([#6784](https://github.com/tokio-rs/tokio/pull/6784))

### 已添加 (不稳定)

- runtime：添加 `Builder::{on_task_spawn, on_task_terminate}` ([#6742](https://github.com/tokio-rs/tokio/pull/6742))

### 已更改

- io：在可能的情况下为 `write_all_buf` 使用向量化 io ([#6724](https://github.com/tokio-rs/tokio/pull/6724))
- runtime：防止 niche-optimization 以避免触发 miri ([#6744](https://github.com/tokio-rs/tokio/pull/6744))
- sync：将 mpsc 类型标记为 `UnwindSafe` ([#6783](https://github.com/tokio-rs/tokio/pull/6783))
- sync,time：使 `Sleep` 和 `BatchSemaphore` 检测成为显式根 ([#6727](https://github.com/tokio-rs/tokio/pull/6727))
- task：为 `task::Id` 使用 `NonZeroU64` ([#6733](https://github.com/tokio-rs/tokio/pull/6733))
- task：在打印 `JoinError` 时包含 panic 消息 ([#6753](https://github.com/tokio-rs/tokio/pull/6753))
- task：为 `JoinHandle::abort_handle` 添加 `#[must_use]` ([#6762](https://github.com/tokio-rs/tokio/pull/6762))
- time：消除计时器轮分配 ([#6779](https://github.com/tokio-rs/tokio/pull/6779))

### 已记录

- docs：阐明 `[build]` 部分不在 Cargo.toml 中 ([#6728](https://github.com/tokio-rs/tokio/pull/6728))
- io：阐明剩余容量为零的情况 ([#6790](https://github.com/tokio-rs/tokio/pull/6790))
- macros：改进 `select!` 的文档 ([#6774](https://github.com/tokio-rs/tokio/pull/6774))
- sync：记录 mpsc 通道分配行为 ([#6773](https://github.com/tokio-rs/tokio/pull/6773))

## 1.39.3 (2024年8月17日)

此版本修复了一个回归问题，即 unix 套接字 API 停止接受抽象套接字命名空间。([#6772](https://github.com/tokio-rs/tokio/pull/6772))

## 1.39.2 (2024年7月27日)

此版本修复了一个回归问题，即 `select!` 宏停止接受使用临时生命周期扩展的表达式。([#6722](https://github.com/tokio-rs/tokio/pull/6722))

## 1.39.1 (2024年7月23日)

此版本恢复了“time: avoid traversing entries in the time wheel twice”，因为它包含一个 bug。([#6715](https://github.com/tokio-rs/tokio/pull/6715))

## 1.39.0 (2024年7月23日)

已撤销。请改用 1.39.1。

- 此版本将 MSRV 提升至 1.70。([#6645](https://github.com/tokio-rs/tokio/pull/6645))
- 此版本升级到 mio v1。([#6635](https://github.com/tokio-rs/tokio/pull/6635))
- 此版本升级到 windows-sys v0.52 ([#6154](https://github.com/tokio-rs/tokio/pull/6154))

### 已添加

- io：为 `Empty` 实现 `AsyncSeek` ([#6663](https://github.com/tokio-rs/tokio/pull/6663))
- metrics：稳定 `num_alive_tasks` ([#6619](https://github.com/tokio-rs/tokio/pull/6619), [#6667](https://github.com/tokio-rs/tokio/pull/6667))
- process：添加 `Command::as_std_mut` ([#6608](https://github.com/tokio-rs/tokio/pull/6608))
- sync：添加 `watch::Sender::same_channel` ([#6637](https://github.com/tokio-rs/tokio/pull/6637))
- sync：添加 `{Receiver,UnboundedReceiver}::{sender_strong_count,sender_weak_count}` ([#6661](https://github.com/tokio-rs/tokio/pull/6661))
- sync：为 `watch::Sender` 实现 `Default` ([#6626](https://github.com/tokio-rs/tokio/pull/6626))
- task：为 `AbortHandle` 实现 `Clone` ([#6621](https://github.com/tokio-rs/tokio/pull/6621))
- task：稳定 `consume_budget` ([#6622](https://github.com/tokio-rs/tokio/pull/6622))

### 已更改

- io：改进 `ReadBuf::put_slice()` 的 panic 消息 ([#6629](https://github.com/tokio-rs/tokio/pull/6629))
- io：在 `copy_bidirectional` 和 `copy` 中进行写时读 ([#6532](https://github.com/tokio-rs/tokio/pull/6532))
- runtime：将 `num_cpus` 替换为 `available_parallelism` ([#6709](https://github.com/tokio-rs/tokio/pull/6709))
- task：避免在将大 future 传递给 `block_on` 时发生堆栈溢出 ([#6692](https://github.com/tokio-rs/tokio/pull/6692))
- time：避免在时间轮中两次遍历条目 ([#6584](https://github.com/tokio-rs/tokio/pull/6584))
- time：支持 `IntoFuture` 与 `timeout` ([#6666](https://github.com/tokio-rs/tokio/pull/6666))
- macros：支持 `IntoFuture` 与 `join!` 和 `select!` ([#6710](https://github.com/tokio-rs/tokio/pull/6710))

### 已修复

- docs：修复启用 fs 功能的 docsrs 构建 ([#6585](https://github.com/tokio-rs/tokio/pull/6585))
- io：仅在已知兼容的平台上使用短读优化 ([#6668](https://github.com/tokio-rs/tokio/pull/6668))
- time：修复使用大持续时间与 `Interval` 时的溢出 panic ([#6612](https://github.com/tokio-rs/tokio/pull/6612))

### 已添加 (不稳定)

- macros：允许 `#[tokio::main]` 和 `#[tokio::test]` 的 `unhandled_panic` 行为 ([#6593](https://github.com/tokio-rs/tokio/pull/6593))
- metrics：添加 `spawned_tasks_count` ([#6114](https://github.com/tokio-rs/tokio/pull/6114))
- metrics：添加 `worker_park_unpark_count` ([#6696](https://github.com/tokio-rs/tokio/pull/6696))
- metrics：添加工作线程 ID ([#6695](https://github.com/tokio-rs/tokio/pull/6695))

### 已记录

- io：更新 `tokio::io::stdout` 文档 ([#6674](https://github.com/tokio-rs/tokio/pull/6674))
- macros：修复 `join.rs` 和 `try_join.rs` 中的拼写错误 ([#6641](https://github.com/tokio-rs/tokio/pull/6641))
- runtime：修复 `unhandled_panic` 中的拼写错误 ([#6660](https://github.com/tokio-rs/tokio/pull/6660))
- task：记录所有任务运行时 `JoinSet::try_join_next` 的行为 ([#6671](https://github.com/tokio-rs/tokio/pull/6671))

## 1.38.2 (2025年4月2日)

此版本修复了广播通道中的一个健全性问题。该通道接受 `Send` 但 `!Sync` 的值。之前，该通道在未同步的情况下对这些值调用 `clone()`。此版本通过同步对 `.clone()` 的调用来修复此问题（感谢 Austin Bonander 发现并报告此问题）。

### 已修复

- sync：在广播通道中同步 `clone()` 调用 ([#7232](https://github.com/tokio-rs/tokio/pull/7232))

## 1.38.1 (2024年7月16日)

此版本修复了被识别为 ([#6682](https://github.com/tokio-rs/tokio/pull/6682)) 的 bug，该 bug 导致计时器在应触发时未触发。

### 已修复

- time：在持有分片时间轮的所有锁时更新 `wake_up` ([#6683](https://github.com/tokio-rs/tokio/pull/6683))

## 1.38.0 (2024年5月30日)

此版本标志着运行时指标稳定化的开始。它稳定了 `RuntimeMetrics::worker_count`。未来的版本将继续稳定更多指标。

### 已添加

- fs：添加 `File::create_new` ([#6573](https://github.com/tokio-rs/tokio/pull/6573))
- io：添加 `copy_bidirectional_with_sizes` ([#6500](https://github.com/tokio-rs/tokio/pull/6500))
- io：为 `Join` 实现 `AsyncBufRead` ([#6449](https://github.com/tokio-rs/tokio/pull/6449))
- net：添加 Apple visionOS 支持 ([#6465](https://github.com/tokio-rs/tokio/pull/6465))
- net：为 `NamedPipeInfo` 实现 `Clone` ([#6586](https://github.com/tokio-rs/tokio/pull/6586))
- net：支持 QNX OS ([#6421](https://github.com/tokio-rs/tokio/pull/6421))
- sync：添加 `Notify::notify_last` ([#6520](https://github.com/tokio-rs/tokio/pull/6520))
- sync：添加 `mpsc::Receiver::{capacity,max_capacity}` ([#6511](https://github.com/tokio-rs/tokio/pull/6511))
- sync：为信号量许可添加 `split` 方法 ([#6472](https://github.com/tokio-rs/tokio/pull/6472), [#6478](https://github.com/tokio-rs/tokio/pull/6478))
- task：添加 `tokio::task::join_set::Builder::spawn_blocking` ([#6578](https://github.com/tokio-rs/tokio/pull/6578))
- wasm：通过 wasm32-wasi-preview1-threads 支持 rt-multi-thread ([#6510](https://github.com/tokio-rs/tokio/pull/6510))

### 已更改

- macros：使 `#[tokio::test]` 在属性列表末尾追加 `#[test]` ([#6497](https://github.com/tokio-rs/tokio/pull/6497))
- metrics：修复 `blocking_threads` 计数 ([#6551](https://github.com/tokio-rs/tokio/pull/6551))
- metrics：稳定 `RuntimeMetrics::worker_count` ([#6556](https://github.com/tokio-rs/tokio/pull/6556))
- runtime：在 `block_in_place` 中将任务移出 `lifo_slot` ([#6596](https://github.com/tokio-rs/tokio/pull/6596))
- runtime：如果 `global_queue_interval` 为零则 panic ([#6445](https://github.com/tokio-rs/tokio/pull/6445))
- sync：在 oneshot receiver 的析构函数中始终丢弃消息 ([#6558](https://github.com/tokio-rs/tokio/pull/6558))
- sync：为任务转储检测 `Semaphore` ([#6499](https://github.com/tokio-rs/tokio/pull/6499))
- sync：唤醒批量 waker 时使用 FIFO 顺序 ([#6521](https://github.com/tokio-rs/tokio/pull/6521))
- task：使 `LocalKey::get` 与 Clone 类型一起工作 ([#6433](https://github.com/tokio-rs/tokio/pull/6433))
- tests：更新 nix 和 mio-aio 开发依赖项 ([#6552](https://github.com/tokio-rs/tokio/pull/6552))
- time：清理实现 ([#6517](https://github.com/tokio-rs/tokio/pull/6517))
- time：在首次轮询时延迟初始化计时器 ([#6512](https://github.com/tokio-rs/tokio/pull/6512))
- time：移除 `TimerShared` 中的 `true_when` 字段 ([#6563](https://github.com/tokio-rs/tokio/pull/6563))
- time：使用分片实现计时器 ([#6534](https://github.com/tokio-rs/tokio/pull/6534))

### 已修复

- taskdump：允许在非 unix 机器上构建 taskdump 文档 ([#6564](https://github.com/tokio-rs/tokio/pull/6564))
- time：检查 `Interval::poll_tick` 中的溢出 ([#6487](https://github.com/tokio-rs/tokio/pull/6487))
- sync：修复 mpsc 块边界上不正确的 `is_empty` ([#6603](https://github.com/tokio-rs/tokio/pull/6603))

### 已记录

- fs：重写文件系统文档 ([#6467](https://github.com/tokio-rs/tokio/pull/6467))
- io：修复 `stdin` 文档 ([#6581](https://github.com/tokio-rs/tokio/pull/6581))
- io：修复 `ReadHalf::unsplit()` 文档中的过时引用 ([#6498](https://github.com/tokio-rs/tokio/pull/6498))
- macros：为 `select!` 渲染更易于理解的文档 ([#6468](https://github.com/tokio-rs/tokio/pull/6468))
- net：为模块文档添加缺失的类型 ([#6482](https://github.com/tokio-rs/tokio/pull/6482))
- net：修复误导性的 `NamedPipeServer` 示例 ([#6590](https://github.com/tokio-rs/tokio/pull/6590))
- sync：为 `SemaphorePermit`、`OwnedSemaphorePermit` 添加示例 ([#6477](https://github.com/tokio-rs/tokio/pull/6477))
- sync：记录 `Barrier::wait` 不是取消安全的 ([#6494](https://github.com/tokio-rs/tokio/pull/6494))
- sync：解释 `watch::Sender::{subscribe,closed}` 之间的关系 ([#6490](https://github.com/tokio-rs/tokio/pull/6490))
- task：阐明不能中止 `spawn_blocking` 任务 ([#6571](https://github.com/tokio-rs/tokio/pull/6571))
- task：修复 `LocalSet::run_until` 文档中的拼写错误 ([#6599](https://github.com/tokio-rs/tokio/pull/6599))
- time：修复文档中暂停和恢复的 test-util 要求 ([#6503](https://github.com/tokio-rs/tokio/pull/6503))

## 1.37.0 (2024年3月28日)

### 已添加

- fs：为 `tokio::fs::File` 添加 `set_max_buf_size` ([#6411](https://github.com/tokio-rs/tokio/pull/6411))
- io：为 `AsyncFd` 添加 `try_new` 和 `try_with_interest` ([#6345](https://github.com/tokio-rs/tokio/pull/6345))
- sync：为信号量添加 `forget_permits` 方法 ([#6331](https://github.com/tokio-rs/tokio/pull/6331))
- sync：为 mpsc 接收器添加 `is_closed`、`is_empty` 和 `len` ([#6348](https://github.com/tokio-rs/tokio/pull/6348))
- sync：为拥有的 `RwLock` 守卫添加 `rwlock()` 方法 ([#6418](https://github.com/tokio-rs/tokio/pull/6418))
- sync：公开 mpsc 发送器句柄的强弱计数 ([#6405](https://github.com/tokio-rs/tokio/pull/6405))
- sync：为 `watch::Sender` 实现 `Clone` ([#6388](https://github.com/tokio-rs/tokio/pull/6388))
- task：添加 `TaskLocalFuture::take_value` ([#6340](https://github.com/tokio-rs/tokio/pull/6340))
- task：为 `JoinSet` 实现 `FromIterator` ([#6300](https://github.com/tokio-rs/tokio/pull/6300))

### 已更改

- io：使 `io::split` 使用互斥锁而不是自旋锁 ([#6403](https://github.com/tokio-rs/tokio/pull/6403))

### 已修复

- docs：修复没有 net 功能的 docsrs 构建 ([#6360](https://github.com/tokio-rs/tokio/pull/6360))
- macros：允许只有 else 分支的 select ([#6339](https://github.com/tokio-rs/tokio/pull/6339))
- runtime：修复 os 注册失败时泄漏注册条目的问题 ([#6329](https://github.com/tokio-rs/tokio/pull/6329))

### 已记录

- io：记录 `AsyncBufReadExt::fill_buf` 的取消安全性 ([#6431](https://github.com/tokio-rs/tokio/pull/6431))
- io：记录 `AsyncReadExt` 原始读取函数的取消安全性 ([#6337](https://github.com/tokio-rs/tokio/pull/6337))
- runtime：添加从 `Runtime` 到 `#[tokio::main]` 的文档链接 ([#6366](https://github.com/tokio-rs/tokio/pull/6366))
- runtime：使 `enter` 示例具有确定性 ([#6351](https://github.com/tokio-rs/tokio/pull/6351))
- sync：为 Semaphore 添加限制传出请求数量的示例 ([#6419](https://github.com/tokio-rs/tokio/pull/6419))
- sync：修复广播文档中缺失的句点 ([#6377](https://github.com/tokio-rs/tokio/pull/6377))
- sync：使用 `#[must_use]` 标记 `mpsc::Sender::downgrade` ([#6326](https://github.com/tokio-rs/tokio/pull/6326))
- sync：在 `new_with` 之前重新排序 `const_new` ([#6392](https://github.com/tokio-rs/tokio/pull/6392))
- sync：更新 watch 通道文档 ([#6395](https://github.com/tokio-rs/tokio/pull/6395))
- task：修复文档链接 ([#6336](https://github.com/tokio-rs/tokio/pull/6336))

### 已更改 (不稳定)

- runtime：在 taskdumps 中包含任务 `Id` ([#6328](https://github.com/tokio-rs/tokio/pull/6328))
- runtime：如果启用 `unhandled_panic` 但不支持则 panic ([#6410](https://github.com/tokio-rs/tokio/pull/6410))

## 1.36.0 (2024年2月2日)

### 已添加

- io：添加 `tokio::io::Join` ([#6220](https://github.com/tokio-rs/tokio/pull/6220))
- io：为 `Empty` 实现 `AsyncWrite` ([#6235](https://github.com/tokio-rs/tokio/pull/6235))
- net：添加对匿名 unix 管道的支持 ([#6127](https://github.com/tokio-rs/tokio/pull/6127))
- net：添加 `UnixSocket` ([#6290](https://github.com/tokio-rs/tokio/pull/6290))
- net：在 `TcpSocket` 上公开 keepalive 选项 ([#6311](https://github.com/tokio-rs/tokio/pull/6311))
- sync：添加 `{Receiver,UnboundedReceiver}::poll_recv_many` ([#6236](https://github.com/tokio-rs/tokio/pull/6236))
- sync：添加 `Sender::{try_,}reserve_many` ([#6205](https://github.com/tokio-rs/tokio/pull/6205))
- sync：添加 `watch::Receiver::mark_unchanged` ([#6252](https://github.com/tokio-rs/tokio/pull/6252))
- task：添加 `JoinSet::try_join_next` ([#6280](https://github.com/tokio-rs/tokio/pull/6280))

### 已更改

- io：使 `copy` 协作 ([#6265](https://github.com/tokio-rs/tokio/pull/6265))
- io：使 `repeat` 和 `sink` 协作 ([#6254](https://github.com/tokio-rs/tokio/pull/6254))
- io：简化空切片的检查 ([#6293](https://github.com/tokio-rs/tokio/pull/6293))
- process：在 Linux 上可用时使用 pidfd ([#6152](https://github.com/tokio-rs/tokio/pull/6152))
- sync：在广播通道 future 中使用 AtomicBool ([#6298](https://github.com/tokio-rs/tokio/pull/6298))

### 已记录

- io：阐明 `clear_ready` 文档 ([#6304](https://github.com/tokio-rs/tokio/pull/6304))
- net：记录 `TcpSocket` 上的 `*Fd` trait 仅限 unix ([#6294](https://github.com/tokio-rs/tokio/pull/6294))
- sync：记录 `tokio::sync::Mutex` 的 FIFO 行为 ([#6279](https://github.com/tokio-rs/tokio/pull/6279))
- chore：排版改进 ([#6262](https://github.com/tokio-rs/tokio/pull/6262))
- runtime：移除过时的注释 ([#6303](https://github.com/tokio-rs/tokio/pull/6303))
- task：修复拼写错误 ([#6261](https://github.com/tokio-rs/tokio/pull/6261))

## 1.35.1 (2023年12月19日)

这是对已向后移植到 1.25.3 的更改的向前移植。

### 已修复

- io：为 `tokio::runtime::io::registration::async_io` 添加预算 ([#6221](https://github.com/tokio-rs/tokio/pull/6221))

## 1.35.0 (2023年12月8日)

### 已添加

- net：添加 Apple watchOS 支持 ([#6176](https://github.com/tokio-rs/tokio/pull/6176))

### 已更改

- io：从 `AsyncReadExt.read_buf` 中移除 `Sized` 要求 ([#6169](https://github.com/tokio-rs/tokio/pull/6169))
- runtime：使 `Runtime` unwind safe ([#6189](https://github.com/tokio-rs/tokio/pull/6189))
- runtime：减少任务生成中的锁竞争 ([#6001](https://github.com/tokio-rs/tokio/pull/6001))
- tokio：将 nix 依赖更新到 0.27.1 ([#6190](https://github.com/tokio-rs/tokio/pull/6190))

### 已修复

- chore：使 `--cfg docsrs` 在没有 net 功能的情况下工作 ([#6166](https://github.com/tokio-rs/tokio/pull/6166))
- chore：在 miri 上为 `unsync_load` 使用宽松加载 ([#6179](https://github.com/tokio-rs/tokio/pull/6179))
- runtime：处理唤醒时缺少上下文的问题 ([#6148](https://github.com/tokio-rs/tokio/pull/6148))
- taskdump：修复 taskdump cargo 配置示例 ([#6150](https://github.com/tokio-rs/tokio/pull/6150))
- taskdump：在任务转储期间跳过已通知的任务 ([#6194](https://github.com/tokio-rs/tokio/pull/6194))
- tracing：避免使用当前父级创建资源跨度，而是使用 None 父级 ([#6107](https://github.com/tokio-rs/tokio/pull/6107))
- tracing：使任务跨度成为显式根 ([#6158](https://github.com/tokio-rs/tokio/pull/6158))

### 已记录

- io：在 `AsyncWriteExt` 示例中刷新 ([#6149](https://github.com/tokio-rs/tokio/pull/6149))
- runtime：记录公平性保证和当前行为 ([#6145](https://github.com/tokio-rs/tokio/pull/6145))
- task：记录 `LocalSet::run_until` 的取消安全性 ([#6147](https://github.com/tokio-rs/tokio/pull/6147))

## 1.34.0 (2023年11月19日)

### 已修复

- io：允许在 io 驱动程序关闭后 `clear_readiness` ([#6067](https://github.com/tokio-rs/tokio/pull/6067))
- io：修复 `take` 中的整数溢出 ([#6080](https://github.com/tokio-rs/tokio/pull/6080))
- io：修复 I/O 资源挂起问题 ([#6134](https://github.com/tokio-rs/tokio/pull/6134))
- sync：修复 `broadcast::channel` 链接 ([#6100](https://github.com/tokio-rs/tokio/pull/6100))

### 已更改

- macros：在 `tokio::test` 宏中使用 `::core` 限定的导入代替 `::std` ([#5973](https://github.com/tokio-rs/tokio/pull/5973))

### 已添加

- fs：在 `fs::read_dir` 中更新 cfg attr 以包含 `aix` ([#6075](https://github.com/tokio-rs/tokio/pull/6075))
- sync：添加 `mpsc::Receiver::recv_many` ([#6010](https://github.com/tokio-rs/tokio/pull/6010))
- tokio：添加了 vita 目标支持 ([#6094](https://github.com/tokio-rs/tokio/pull/6094))

## 1.33.0 (2023年10月9日)

### 已修复

- io：使用 `#[must_use]` 标记 `Interest::add` ([#6037](https://github.com/tokio-rs/tokio/pull/6037))
- runtime：修复 RISC-V 的缓存行大小 ([#5994](https://github.com/tokio-rs/tokio/pull/5994))
- sync：防止 `watch::Receiver::wait_for` 中的锁中毒 ([#6021](https://github.com/tokio-rs/tokio/pull/6021))
- task：修复 `spawn_local` 源位置 ([#5984](https://github.com/tokio-rs/tokio/pull/5984))

### 已更改

- sync：在 `watch` 中使用 Acquire/Release 顺序代替 SeqCst ([#6018](https://github.com/tokio-rs/tokio/pull/6018))

### 已添加

- fs：为 `tokio::fs::File` 添加向量化写入 ([#5958](https://github.com/tokio-rs/tokio/pull/5958))
- io：添加 `Interest::remove` 方法 ([#5906](https://github.com/tokio-rs/tokio/pull/5906))
- io：为 `DuplexStream` 添加向量化写入 ([#5985](https://github.com/tokio-rs/tokio/pull/5985))
- net：添加 Apple tvOS 支持 ([#6045](https://github.com/tokio-rs/tokio/pull/6045))
- sync：为 `{MutexGuard,OwnedMutexGuard}::map` 添加 `?Sized` 绑定 ([#5997](https://github.com/tokio-rs/tokio/pull/5997))
- sync：添加 `watch::Receiver::mark_unseen` ([#5962](https://github.com/tokio-rs/tokio/pull/5962), [#6014](https://github.com/tokio-rs/tokio/pull/6014), [#6017](https://github.com/tokio-rs/tokio/pull/6017))
- sync：添加 `watch::Sender::new` ([#5998](https://github.com/tokio-rs/tokio/pull/5998))
- sync：添加 const fn `OnceCell::from_value` ([#5903](https://github.com/tokio-rs/tokio/pull/5903))

### 已移除

- 移除未使用的 `stats` 功能 ([#5952](https://github.com/tokio-rs/tokio/pull/5952))

### 已记录

- 在代码示例中添加缺失的反引号 ([#5938](https://github.com/tokio-rs/tokio/pull/5938), [#6056](https://github.com/tokio-rs/tokio/pull/6056))
- 修复拼写错误 ([#5988](https://github.com/tokio-rs/tokio/pull/5988), [#6030](https://github.com/tokio-rs/tokio/pull/6030))
- process：记录 `Child::wait` 是取消安全的 ([#5977](https://github.com/tokio-rs/tokio/pull/5977))
- sync：为 `Semaphore` 添加示例 ([#5939](https://github.com/tokio-rs/tokio/pull/5939), [#5956](https://github.com/tokio-rs/tokio/pull/5956), [#5978](https://github.com/tokio-rs/tokio/pull/5978), [#6031](https://github.com/tokio-rs/tokio/pull/6031), [#6032](https://github.com/tokio-rs/tokio/pull/6032), [#6050](https://github.com/tokio-rs/tokio/pull/6050))
- sync：记录 `broadcast` 容量是下限 ([#6042](https://github.com/tokio-rs/tokio/pull/6042))
- sync：记录 `const_new` 未被检测 ([#6002](https://github.com/tokio-rs/tokio/pull/6002))
- sync：改进 `mpsc::Sender::send` 的取消安全文档 ([#5947](https://github.com/tokio-rs/tokio/pull/5947))
- sync：改进 `watch` 通道的文档 ([#5954](https://github.com/tokio-rs/tokio/pull/5954))
- taskdump：在 docs.rs 上渲染 taskdump 文档 ([#5972](https://github.com/tokio-rs/tokio/pull/5972))

### 不稳定

- taskdump：修复潜在的死锁 ([#6036](https://github.com/tokio-rs/tokio/pull/6036))

## 1.32.1 (2023年12月19日)

这是对已向后移植到 1.25.3 的更改的向前移植。

### 已修复

- io：为 `tokio::runtime::io::registration::async_io` 添加预算 ([#6221](https://github.com/tokio-rs/tokio/pull/6221))

## 1.32.0 (2023年8月16日)

### 已修复

- sync：修复 `broadcast::Receiver` 中潜在的二次行为 ([#5925](https://github.com/tokio-rs/tokio/pull/5925))

### 已添加

- process：稳定 `Command::raw_arg` ([#5930](https://github.com/tokio-rs/tokio/pull/5930))
- io：允许等待错误就绪 ([#5781](https://github.com/tokio-rs/tokio/pull/5781))

### 不稳定

- rt(alt)：随着核心数量的增长，提高备用运行时的可扩展性 ([#5935](https://github.com/tokio-rs/tokio/pull/5935))

## 1.31.0 (2023年8月10日)

### 已修复

- io：委托 `WriteHalf::poll_write_vectored` ([#5914](https://github.com/tokio-rs/tokio/pull/5914))

### 不稳定

- rt(alt)：修复不稳定的下一代调度器原型中的内存泄漏 ([#5911](https://github.com/tokio-rs/tokio/pull/5911))
- rt：公开平均任务轮询时间指标 ([#5927](https://github.com/tokio-rs/tokio/pull/5927))

## 1.30.0 (2023年8月9日)

此版本将 Tokio 的 MSRV 提升至 1.63。([#5887](https://github.com/tokio-rs/tokio/pull/5887))

### 已更改

- tokio：减少 LLVM 代码生成 ([#5859](https://github.com/tokio-rs/tokio/pull/5859))
- io：支持 `--cfg mio_unsupported_force_poll_poll` 标志 ([#5881](https://github.com/tokio-rs/tokio/pull/5881))
- sync：使 `const_new` 方法始终可用 ([#5885](https://github.com/tokio-rs/tokio/pull/5885))
- sync：避免 mpsc 通道中的伪共享 ([#5829](https://github.com/tokio-rs/tokio/pull/5829))
- rt：从注入队列中至少弹出一个任务 ([#5908](https://github.com/tokio-rs/tokio/pull/5908))

### 已添加

- sync：添加 `broadcast::Sender::new` ([#5824](https://github.com/tokio-rs/tokio/pull/5824))
- net：为 espidf 实现 `UCred` ([#5868](https://github.com/tokio-rs/tokio/pull/5868))
- fs：添加 `File::options()` ([#5869](https://github.com/tokio-rs/tokio/pull/5869))
- time：为 `Interval` 实现额外的重置变体 ([#5878](https://github.com/tokio-rs/tokio/pull/5878))
- process：添加 `{ChildStd*}::into_owned_{fd, handle}` ([#5899](https://github.com/tokio-rs/tokio/pull/5899))

### 已移除

- tokio：移除未使用的 `tokio_*` cfgs ([#5890](https://github.com/tokio-rs/tokio/pull/5890))
- 移除构建脚本以加快编译速度 ([#5887](https://github.com/tokio-rs/tokio/pull/5887))

### 已记录

- sync：在 `broadcast::send` 的文档中提及滞后 ([#5820](https://github.com/tokio-rs/tokio/pull/5820))
- runtime：扩展共享运行时文档 ([#5858](https://github.com/tokio-rs/tokio/pull/5858))
- io：在 `AsyncReadExt::read_exact` 的示例中使用 vec ([#5863](https://github.com/tokio-rs/tokio/pull/5863))
- time：在文档中将 `Sleep` 标记为 `!Unpin` ([#5916](https://github.com/tokio-rs/tokio/pull/5916))
- process：修复 `raw_arg` 未在文档中显示的问题 ([#5865](https://github.com/tokio-rs/tokio/pull/5865))

### 不稳定

- rt：添加运行时 ID ([#5864](https://github.com/tokio-rs/tokio/pull/5864))
- rt：新线程运行时的初始实现 ([#5823](https://github.com/tokio-rs/tokio/pull/5823))

## 1.29.1 (2023年6月29日)

### 已修复

- rt：修复在两个 `block_in_place` 之间嵌套一个 `block_on` 的问题 ([#5837](https://github.com/tokio-rs/tokio/pull/5837))

## 1.29.0 (2023年6月27日)

技术上是一个破坏性更改，从 `runtime::EnterGuard` 中移除了 `Send` 实现。此更改修复了一个 bug，不应影响大多数用户。

### 破坏性更改

- rt：`EnterGuard` 不应是 `Send` ([#5766](https://github.com/tokio-rs/tokio/pull/5766))

### 已修复

- fs：减少 `fs::read_dir` 中的阻塞操作 ([#5653](https://github.com/tokio-rs/tokio/pull/5653))
- rt：修复可能的饥饿问题 ([#5686](https://github.com/tokio-rs/tokio/pull/5686), [#5712](https://github.com/tokio-rs/tokio/pull/5712))
- rt：修复 `JoinSet` 中的堆栈借用问题 ([#5693](https://github.com/tokio-rs/tokio/pull/5693))
- rt：如果 `EnterGuard` 以不正确的顺序被丢弃则 panic ([#5772](https://github.com/tokio-rs/tokio/pull/5772))
- time：不要溢出到信号值 ([#5710](https://github.com/tokio-rs/tokio/pull/5710))
- fs：在克隆 `File` 之前等待进行中的操作 ([#5803](https://github.com/tokio-rs/tokio/pull/5803))

### 已更改

- rt：减少轮询从运行时外部调度的任务的时间 ([#5705](https://github.com/tokio-rs/tokio/pull/5705), [#5720](https://github.com/tokio-rs/tokio/pull/5720))

### 已添加

- net：为 unix 套接字添加 uds 文档别名 ([#5659](https://github.com/tokio-rs/tokio/pull/5659))
- rt：添加任务数量的指标 ([#5628](https://github.com/tokio-rs/tokio/pull/5628))
- sync：为通道错误实现更多 trait ([#5666](https://github.com/tokio-rs/tokio/pull/5666))
- net：在 TcpSocket 上添加 nodelay 方法 ([#5672](https://github.com/tokio-rs/tokio/pull/5672))
- sync：添加 `broadcast::Receiver::blocking_recv` ([#5690](https://github.com/tokio-rs/tokio/pull/5690))
- process：为 `Command` 添加 `raw_arg` 方法 ([#5704](https://github.com/tokio-rs/tokio/pull/5704))
- io：支持 PRIORITY epoll 事件 ([#5566](https://github.com/tokio-rs/tokio/pull/5566))
- task：添加 `JoinSet::poll_join_next` ([#5721](https://github.com/tokio-rs/tokio/pull/5721))
- net：添加对 Redox OS 的支持 ([#5790](https://github.com/tokio-rs/tokio/pull/5790))

### 不稳定

- rt：添加转储任务回溯的功能 ([#5608](https://github.com/tokio-rs/tokio/pull/5608), [#5676](https://github.com/tokio-rs/tokio/pull/5676), [#5708](https://github.com/tokio-rs/tokio/pull/5708), [#5717](https://github.com/tokio-rs/tokio/pull/5717))
- rt：使用直方图检测任务轮询时间 ([#5685](https://github.com/tokio-rs/tokio/pull/5685))

## 1.28.2 (2023年5月28日)

向前移植 1.18.6 的更改。

### 已修复

- deps：禁用 mio 的默认功能 ([#5728](https://github.com/tokio-rs/tokio/pull/5728))

## 1.28.1 (2023年5月10日)

此版本修复了构建脚本中的一个错误，该错误导致 `AsFd` 实现在 Rust 1.63 上不可用。([#5677](https://github.com/tokio-rs/tokio/pull/5677))

## 1.28.0 (2023年4月25日)

### 已添加

- io：添加 `AsyncFd::async_io` ([#5542](https://github.com/tokio-rs/tokio/pull/5542))
- io：为 ReadBuf 实现 BufMut ([#5590](https://github.com/tokio-rs/tokio/pull/5590))
- net：为 `UdpSocket` 和 `UnixDatagram` 添加 `recv_buf` ([#5583](https://github.com/tokio-rs/tokio/pull/5583))
- sync：添加 `OwnedSemaphorePermit::semaphore` ([#5618](https://github.com/tokio-rs/tokio/pull/5618))
- sync：为广播通道添加 `same_channel` ([#5607](https://github.com/tokio-rs/tokio/pull/5607))
- sync：添加 `watch::Receiver::wait_for` ([#5611](https://github.com/tokio-rs/tokio/pull/5611))
- task：添加 `JoinSet::spawn_blocking` 和 `JoinSet::spawn_blocking_on` ([#5612](https://github.com/tokio-rs/tokio/pull/5612))

### 已更改

- deps：将 windows-sys 更新到 0.48 ([#5591](https://github.com/tokio-rs/tokio/pull/5591))
- io：使 `read_to_end` 不会不必要地增长 ([#5610](https://github.com/tokio-rs/tokio/pull/5610))
- macros：使入口点更高效 ([#5621](https://github.com/tokio-rs/tokio/pull/5621))
- sync：改进 `RwLock` 的 Debug 实现 ([#5647](https://github.com/tokio-rs/tokio/pull/5647))
- sync：减少 `Notify` 中的竞争 ([#5503](https://github.com/tokio-rs/tokio/pull/5503))

### 已修复

- net：支持在 AIX 上 `get_peer_cred` ([#5065](https://github.com/tokio-rs/tokio/pull/5065))
- sync：避免 `broadcast` 中使用自定义 waker 时的死锁 ([#5578](https://github.com/tokio-rs/tokio/pull/5578))

### 已记录

- sync：修复 `Semaphore::MAX_PERMITS` 中的拼写错误 ([#5645](https://github.com/tokio-rs/tokio/pull/5645))
- sync：修复 `tokio::sync::watch::Sender` 文档中的拼写错误 ([#5587](https://github.com/tokio-rs/tokio/pull/5587))

## 1.27.0 (2023年3月27日)

此版本将 Tokio 的 MSRV 提升至 1.56。([#5559](https://github.com/tokio-rs/tokio/pull/5559))

### 已添加

- io：为套接字添加 `async_io` 辅助方法 ([#5512](https://github.com/tokio-rs/tokio/pull/5512))
- io：添加 `AsFd`/`AsHandle`/`AsSocket` 的实现 ([#5514](https://github.com/tokio-rs/tokio/pull/5514), [#5540](https://github.com/tokio-rs/tokio/pull/5540))
- net：添加 `UdpSocket::peek_sender()` ([#5520](https://github.com/tokio-rs/tokio/pull/5520))
- sync：添加 `RwLockWriteGuard::{downgrade_map, try_downgrade_map}` ([#5527](https://github.com/tokio-rs/tokio/pull/5527))
- task：添加 `JoinHandle::abort_handle` ([#5543](https://github.com/tokio-rs/tokio/pull/5543))

### 已更改

- io：使用 `libc` 中的 `memchr` ([#5558](https://github.com/tokio-rs/tokio/pull/5558))
- macros：在 `#[tokio::main]` 中接受路径作为 crate 重命名 ([#5557](https://github.com/tokio-rs/tokio/pull/5557))
- macros：更新到 syn 2.0.0 ([#5572](https://github.com/tokio-rs/tokio/pull/5572))
- time：当 `Interval` 返回 `Ready` 时不注册唤醒 ([#5553](https://github.com/tokio-rs/tokio/pull/5553))

### 已修复

- fs：在 `ReadDir` 中融合 std 迭代器 ([#5555](https://github.com/tokio-rs/tokio/pull/5555))
- tracing：修复 `spawn_blocking` 位置字段 ([#5573](https://github.com/tokio-rs/tokio/pull/5573))
- time：清理 `Wheel::poll()` 中的冗余检查 ([#5574](https://github.com/tokio-rs/tokio/pull/5574))

### 已记录

- macros：定义取消安全性 ([#5525](https://github.com/tokio-rs/tokio/pull/5525))
- io：在 `tokio::io::copy[_buf]` 的文档中添加细节 ([#5575](https://github.com/tokio-rs/tokio/pull/5575))
- io：在模块文档中引用 `ReaderStream` 和 `StreamReader` ([#5576](https://github.com/tokio-rs/tokio/pull/5576))

## 1.26.0 (2023年3月1日)

### 已修复

- macros：修复空的 `join!` 和 `try_join!` ([#5504](https://github.com/tokio-rs/tokio/pull/5504))
- sync：不要在互斥锁守卫中泄漏 tracing spans ([#5469](https://github.com/tokio-rs/tokio/pull/5469))
- sync：在 Notify 中解锁互斥锁后丢弃 wakers ([#5471](https://github.com/tokio-rs/tokio/pull/5471))
- sync：在 semaphore 中在锁外丢弃 wakers ([#5475](https://github.com/tokio-rs/tokio/pull/5475))

### 已添加

- fs：添加 `fs::try_exists` ([#4299](https://github.com/tokio-rs/tokio/pull/4299))
- net：为命名的 unix 管道添加类型 ([#5351](https://github.com/tokio-rs/tokio/pull/5351))
- sync：添加 `MappedOwnedMutexGuard` ([#5474](https://github.com/tokio-rs/tokio/pull/5474))

### 已更改

- chore：将 windows-sys 更新到 0.45 ([#5386](https://github.com/tokio-rs/tokio/pull/5386))
- net：为命名管道使用消息读取模式 ([#5350](https://github.com/tokio-rs/tokio/pull/5350))
- sync：使用 `#[clippy::has_significant_drop]` 标记锁守卫 ([#5422](https://github.com/tokio-rs/tokio/pull/5422))
- sync：减少 watch 通道中的竞争 ([#5464](https://github.com/tokio-rs/tokio/pull/5464))
- time：移除计时器条目中的缓存填充 ([#5468](https://github.com/tokio-rs/tokio/pull/5468))
- time：通过 test-util 提高 `Instant::now()` 的性能 ([#5513](https://github.com/tokio-rs/tokio/pull/5513))

### 不稳定

- metrics：为预算耗尽的让步添加新指标 ([#5517](https://github.com/tokio-rs/tokio/pull/5517))

### 已记录

- io：改进 AsyncFd 示例 ([#5481](https://github.com/tokio-rs/tokio/pull/5481))
- runtime：记录 main future 的性质 ([#5494](https://github.com/tokio-rs/tokio/pull/5494))
- runtime：移除文档中多余的句点 ([#5511](https://github.com/tokio-rs/tokio/pull/5511))
- signal：更新信号的文档 ([#5459](https://github.com/tokio-rs/tokio/pull/5459))
- sync：为 `blocking_*` 方法添加文档别名 ([#5448](https://github.com/tokio-rs/tokio/pull/5448))
- sync：修复广播中 Send/Sync 绑定的文档 ([#5480](https://github.com/tokio-rs/tokio/pull/5480))
- sync：记录通道的丢弃行为 ([#5497](https://github.com/tokio-rs/tokio/pull/5497))
- task：阐明运行时关闭期间生成的任务会发生什么 ([#5394](https://github.com/tokio-rs/tokio/pull/5394))
- task：阐明 `process::Command` 文档 ([#5413](https://github.com/tokio-rs/tokio/pull/5413))
- task：修复 'unsend' 的措辞 ([#5452](https://github.com/tokio-rs/tokio/pull/5452))
- time：记录超时的立即完成保证 ([#5509](https://github.com/tokio-rs/tokio/pull/5509))
- tokio：记录支持的平台 ([#5483](https://github.com/tokio-rs/tokio/pull/5483))

## 1.25.3 (2023年12月17日)

### 已修复
- io：为 `tokio::runtime::io::registration::async_io` 添加预算 ([#6221](https://github.com/tokio-rs/tokio/pull/6221))

## 1.25.2 (2023年9月22日)

向前移植 1.20.6 的更改。

### 已更改

- io：使用 `libc` 中的 `memchr` ([#5960](https://github.com/tokio-rs/tokio/pull/5960))

## 1.25.1 (2023年5月28日)

向前移植 1.18.6 的更改。

### 已修复

- deps：禁用 mio 的默认功能 ([#5728](https://github.com/tokio-rs/tokio/pull/5728))

## 1.25.0 (2023年1月28日)

### 已修复

- rt：修复运行时指标报告 ([#5330](https://github.com/tokio-rs/tokio/pull/5330))

### 已添加

- sync：添加 `broadcast::Sender::len` ([#5343](https://github.com/tokio-rs/tokio/pull/5343))

### 已更改

- fs：将最大读取缓冲区大小增加到 2MiB ([#5397](https://github.com/tokio-rs/tokio/pull/5397))