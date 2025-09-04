# Changelog

A detailed record of all notable changes made to the Tokio library in each version.

## 1.47.1 (August 1st, 2025)

### Fixed

- process: fix panic from spurious pidfd wakeup [[#7494]](https://github.com/tokio-rs/tokio/pull/7494)
- sync: fix broken link of Python `asyncio.Event` in `SetOnce` docs [[#7485]](https://github.com/tokio-rs/tokio/pull/7485)

## 1.47.0 (July 25th, 2025)

This release adds `poll_proceed` and `cooperative` to the `coop` module for cooperative scheduling, adds `SetOnce` to the `sync` module which provides similar functionality to `std::sync::OnceLock`, and adds a new method `sync::Notify::notified_owned()` which returns an `OwnedNotified` without a lifetime parameter.

### Added

- coop: add `cooperative` and `poll_proceed` [[#7405]](https://github.com/tokio-rs/tokio/pull/7405)
- sync: add `SetOnce` [[#7418]](https://github.com/tokio-rs/tokio/pull/7418)
- sync: add `sync::Notify::notified_owned()` [[#7465]](https://github.com/tokio-rs/tokio/pull/7465)

### Changed

- deps: upgrade windows-sys 0.52 → 0.59 [[#7117]](https://github.com/tokio-rs/tokio/pull/7117)
- deps: update to socket2 v0.6 [[#7443]](https://github.com/tokio-rs/tokio/pull/7443)
- sync: improve `AtomicWaker::wake` performance [[#7450]](https://github.com/tokio-rs/tokio/pull/7450)

### Documented

- metrics: fix listed feature requirements for some metrics [[#7449]](https://github.com/tokio-rs/tokio/pull/7449)
- runtime: improve safety comments of `Readiness<'_>` [[#7415]](https://github.com/tokio-rs/tokio/pull/7415)

## 1.46.1 (July 4th, 2025)

This release fixes incorrect spawn locations in runtime task hooks for tasks spawned using `tokio::spawn` rather than `Runtime::spawn`. This issue only effected the spawn location in `TaskMeta::spawned_at`, and did not effect task locations in Tracing events.

### Unstable

- runtime: add `TaskMeta::spawn_location` tracking where a task was spawned [[#7440]](https://github.com/tokio-rs/tokio/pull/7440)

## 1.46.0 (July 2nd, 2025)

### Fixed

- net: fixed `TcpStream::shutdown` incorrectly returning an error on macOS [[#7290]](https://github.com/tokio-rs/tokio/pull/7290)

### Added

- sync: `mpsc::OwnedPermit::{same_channel, same_channel_as_sender}` methods [[#7389]](https://github.com/tokio-rs/tokio/pull/7389)
- macros: `biased` option for `join!` and `try_join!`, similar to `select!` [[#7307]](https://github.com/tokio-rs/tokio/pull/7307)
- net: support for cygwin [[#7393]](https://github.com/tokio-rs/tokio/pull/7393)
- net: support `pipe::OpenOptions::read_write` on Android [[#7426]](https://github.com/tokio-rs/tokio/pull/7426)
- net: add `Clone` implementation for `net::unix::SocketAddr` [[#7422]](https://github.com/tokio-rs/tokio/pull/7422)

### Changed

- runtime: eliminate unnecessary lfence while operating on `queue::Local<T>` [[#7340]](https://github.com/tokio-rs/tokio/pull/7340)
- task: disallow blocking in `LocalSet::{poll,drop}` [[#7372]](https://github.com/tokio-rs/tokio/pull/7372)

### Unstable

- runtime: add `TaskMeta::spawn_location` tracking where a task was spawned [[#7417]](https://github.com/tokio-rs/tokio/pull/7417)
- runtime: removed borrow from `LocalOptions` parameter to `runtime::Builder::build_local` [[#7346]](https://github.com/tokio-rs/tokio/pull/7346)

### Documented

- io: clarify behavior of seeking when `start_seek` is not used [[#7366]](https://github.com/tokio-rs/tokio/pull/7366)
- io: document cancellation safety of `AsyncWriteExt::flush` [[#7364]](https://github.com/tokio-rs/tokio/pull/7364)
- net: fix docs for `recv_buffer_size` method [[#7336]](https://github.com/tokio-rs/tokio/pull/7336)
- net: fix broken link of `RawFd` in `TcpSocket` docs [[#7416]](https://github.com/tokio-rs/tokio/pull/7416)
- net: update `AsRawFd` doc link to current Rust stdlib location [[#7429]](https://github.com/tokio-rs/tokio/pull/7429)
- readme: fix double period in reactor description [[#7363]](https://github.com/tokio-rs/tokio/pull/7363)
- runtime: add doc note that `on_*_task_poll` is unstable [[#7311]](https://github.com/tokio-rs/tokio/pull/7311)
- sync: update broadcast docs on allocation failure [[#7352]](https://github.com/tokio-rs/tokio/pull/7352)
- time: add a missing panic scenario of `time::advance` [[#7394]](https://github.com/tokio-rs/tokio/pull/7394)

## 1.45.1 (May 24th, 2025)

This fixes a regression on the wasm32-unknown-unknown target, where code that previously did not panic due to calls to `Instant::now()` started failing. This is due to the stabilization of the first time-based metric.

### Fixed

- Disable time-based metrics on wasm32-unknown-unknown [[#7322]](https://github.com/tokio-rs/tokio/pull/7322)

## 1.45.0 (May 5th, 2025)

### Added

- metrics: stabilize `worker_total_busy_duration`, `worker_park_count`, and `worker_unpark_count` [[#6899]](https://github.com/tokio-rs/tokio/pull/6899), [[#7276]](https://github.com/tokio-rs/tokio/pull/7276)
- process: add `Command::spawn_with` [[#7249]](https://github.com/tokio-rs/tokio/pull/7249)

### Changed

- io: do not require `Unpin` for some trait impls [[#7204]](https://github.com/tokio-rs/tokio/pull/7204)
- rt: mark `runtime::Handle` as unwind safe [[#7230]](https://github.com/tokio-rs/tokio/pull/7230)
- time: revert internal sharding implementation [[#7226]](https://github.com/tokio-rs/tokio/pull/7226)

### Unstable

- rt: remove alt multi-threaded runtime [[#7275]](https://github.com/tokio-rs/tokio/pull/7275)

## 1.44.2 (April 5th, 2025)

This release fixes a soundness issue in the broadcast channel. The channel accepts values that are `Send` but `!Sync`. Previously, the channel called `clone()` on these values without synchronizing. This release fixes the channel by synchronizing calls to `.clone()` (Thanks Austin Bonander for finding and reporting the issue).

### Fixed

- sync: synchronize `clone()` call in broadcast channel [[#7232]](https://github.com/tokio-rs/tokio/pull/7232)

## 1.44.1 (March 13th, 2025)

### Fixed

- rt: skip defer queue in `block_in_place` context [[#7216]](https://github.com/tokio-rs/tokio/pull/7216)

## 1.44.0 (March 7th, 2025)

This release changes the `from_std` method on sockets to panic if a blocking socket is provided. We determined this change is not a breaking change as Tokio is not intended to operate using blocking sockets. Doing so results in runtime hangs and should be considered a bug. Accidentally passing a blocking socket to Tokio is one of the most common user mistakes. If this change causes an issue for you, please comment on [[#7172]](https://github.com/tokio-rs/tokio/pull/7172).

### Added

- coop: add `task::coop` module [[#7116]](https://github.com/tokio-rs/tokio/pull/7116)
- process: add `Command::get_kill_on_drop()` [[#7086]](https://github.com/tokio-rs/tokio/pull/7086)
- sync: add `broadcast::Sender::closed` [[#6685]](https://github.com/tokio-rs/tokio/pull/6685), [[#7090]](https://github.com/tokio-rs/tokio/pull/7090)
- sync: add `broadcast::WeakSender` [[#7100]](https://github.com/tokio-rs/tokio/pull/7100)
- sync: add `oneshot::Receiver::is_empty()` [[#7153]](https://github.com/tokio-rs/tokio/pull/7153)
- sync: add `oneshot::Receiver::is_terminated()` [[#7152]](https://github.com/tokio-rs/tokio/pull/7152)

### Fixed

- fs: empty reads on `File` should not start a background read [[#7139]](https://github.com/tokio-rs/tokio/pull/7139)
- process: calling `start_kill` on exited child should not fail [[#7160]](https://github.com/tokio-rs/tokio/pull/7160)
- signal: fix `CTRL_CLOSE`, `CTRL_LOGOFF`, `CTRL_SHUTDOWN` on windows [[#7122]](https://github.com/tokio-rs/tokio/pull/7122)
- sync: properly handle panic during mpsc drop [[#7094]](https://github.com/tokio-rs/tokio/pull/7094)

### Changes

- runtime: clean up magic number in registration set [[#7112]](https://github.com/tokio-rs/tokio/pull/7112)
- coop: make coop yield using waker defer strategy [[#7185]](https://github.com/tokio-rs/tokio/pull/7185)
- macros: make `select!` budget-aware [[#7164]](https://github.com/tokio-rs/tokio/pull/7164)
- net: panic when passing a blocking socket to `from_std` [[#7166]](https://github.com/tokio-rs/tokio/pull/7166)
- io: clean up buffer casts [[#7142]](https://github.com/tokio-rs/tokio/pull/7142)

### Changes to unstable APIs

- rt: add before and after task poll callbacks [[#7120]](https://github.com/tokio-rs/tokio/pull/7120)
- tracing: make the task tracing API unstable public [[#6972]](https://github.com/tokio-rs/tokio/pull/6972)

### Documented

- docs: fix nesting of sections in top-level docs [[#7159]](https://github.com/tokio-rs/tokio/pull/7159)
- fs: rename symlink and hardlink parameter names [[#7143]](https://github.com/tokio-rs/tokio/pull/7143)
- io: swap reader/writer in simplex doc test [[#7176]](https://github.com/tokio-rs/tokio/pull/7176)
- macros: docs about `select!` alternatives [[#7110]](https://github.com/tokio-rs/tokio/pull/7110)
- net: rename the argument for `send_to` [[#7146]](https://github.com/tokio-rs/tokio/pull/7146)
- process: add example for reading `Child` stdout [[#7141]](https://github.com/tokio-rs/tokio/pull/7141)
- process: clarify `Child::kill` behavior [[#7162]](https://github.com/tokio-rs/tokio/pull/7162)
- process: fix grammar of the `ChildStdin` struct doc comment [[#7192]](https://github.com/tokio-rs/tokio/pull/7192)
- runtime: consistently use `worker_threads` instead of `core_threads` [[#7186]](https://github.com/tokio-rs/tokio/pull/7186)

## 1.43.2 (August 1st, 2025)

### Fixed

- process: fix panic from spurious pidfd wakeup [[#7494]](https://github.com/tokio-rs/tokio/pull/7494)

## 1.43.1 (April 5th, 2025)

This release fixes a soundness issue in the broadcast channel. The channel accepts values that are `Send` but `!Sync`. Previously, the channel called `clone()` on these values without synchronizing. This release fixes the channel by synchronizing calls to `.clone()` (Thanks Austin Bonander for finding and reporting the issue).

### Fixed

- sync: synchronize `clone()` call in broadcast channel [[#7232]](https://github.com/tokio-rs/tokio/pull/7232)

## 1.43.0 (Jan 8th, 2025)

### Added

- net: add `UdpSocket::peek` methods [[#7068]](https://github.com/tokio-rs/tokio/pull/7068)
- net: add support for Haiku OS [[#7042]](https://github.com/tokio-rs/tokio/pull/7042)
- process: add `Command::into_std()` [[#7014]](https://github.com/tokio-rs/tokio/pull/7014)
- signal: add `SignalKind::info` on illumos [[#6995]](https://github.com/tokio-rs/tokio/pull/6995)
- signal: add support for realtime signals on illumos [[#7029]](https://github.com/tokio-rs/tokio/pull/7029)

### Fixed

- io: don't call `set_len` before initializing vector in `Blocking` [[#7054]](https://github.com/tokio-rs/tokio/pull/7054)
- macros: suppress `clippy::needless_return` in `#[tokio::main]` [[#6874]](https://github.com/tokio-rs/tokio/pull/6874)
- runtime: fix thread parking on WebAssembly [[#7041]](https://github.com/tokio-rs/tokio/pull/7041)

### Changes

- chore: use unsync loads for `unsync_load` [[#7073]](https://github.com/tokio-rs/tokio/pull/7073)
- io: use `Buf::put_bytes` in `Repeat` read impl [[#7055]](https://github.com/tokio-rs/tokio/pull/7055)
- task: drop the join waker of a task eagerly [[#6986]](https://github.com/tokio-rs/tokio/pull/6986)

### Changes to unstable APIs

- metrics: improve flexibility of H2Histogram Configuration [[#6963]](https://github.com/tokio-rs/tokio/pull/6963)
- taskdump: add accessor methods for backtrace [[#6975]](https://github.com/tokio-rs/tokio/pull/6975)

### Documented

- io: clarify `ReadBuf::uninit` allows initialized buffers as well [[#7053]](https://github.com/tokio-rs/tokio/pull/7053)
- net: fix ambiguity in `TcpStream::try_write_vectored` docs [[#7067]](https://github.com/tokio-rs/tokio/pull/7067)
- runtime: fix `LocalRuntime` doc links [[#7074]](https://github.com/tokio-rs/tokio/pull/7074)
- sync: extend documentation for `watch::Receiver::wait_for` [[#7038]](https://github.com/tokio-rs/tokio/pull/7038)
- sync: fix typos in `OnceCell` docs [[#7047]](https://github.com/tokio-rs/tokio/pull/7047)

## 1.42.1 (April 8th, 2025)

This release fixes a soundness issue in the broadcast channel. The channel accepts values that are `Send` but `!Sync`. Previously, the channel called `clone()` on these values without synchronizing. This release fixes the channel by synchronizing calls to `.clone()` (Thanks Austin Bonander for finding and reporting the issue).

### Fixed

- sync: synchronize `clone()` call in broadcast channel [[#7232]](https://github.com/tokio-rs/tokio/pull/7232)

## 1.42.0 (Dec 3rd, 2024)

### Added

- io: add `AsyncFd::{try_io, try_io_mut}` [[#6967]](https://github.com/tokio-rs/tokio/pull/6967)

### Fixed

- io: avoid `ptr->ref->ptr` roundtrip in RegistrationSet [[#6929]](https://github.com/tokio-rs/tokio/pull/6929)
- runtime: do not defer `yield_now` inside `block_in_place` [[#6999]](https://github.com/tokio-rs/tokio/pull/6999)

### Changes

- io: simplify io readiness logic [[#6966]](https://github.com/tokio-rs/tokio/pull/6966)

### Documented

- net: fix docs for `tokio::net::unix::{pid_t, gid_t, uid_t}` [[#6791]](https://github.com/tokio-rs/tokio/pull/6791)
- time: fix a typo in `Instant` docs [[#6982]](https://github.com/tokio-rs/tokio/pull/6982)

## 1.41.1 (Nov 7th, 2024)

### Fixed

- metrics: fix bug with wrong number of buckets for the histogram [[#6957]](https://github.com/tokio-rs/tokio/pull/6957)
- net: display `net` requirement for `net::UdpSocket` in docs [[#6938]](https://github.com/tokio-rs/tokio/pull/6938)
- net: fix typo in `TcpStream` internal comment [[#6944]](https://github.com/tokio-rs/tokio/pull/6944)

## 1.41.0 (Oct 22nd, 2024)

### Added

- metrics: stabilize `global_queue_depth` [[#6854]](https://github.com/tokio-rs/tokio/pull/6854), [[#6918]](https://github.com/tokio-rs/tokio/pull/6918)
- net: add conversions for unix `SocketAddr` [[#6868]](https://github.com/tokio-rs/tokio/pull/6868)
- sync: add `watch::Sender::sender_count` [[#6836]](https://github.com/tokio-rs/tokio/pull/6836)
- sync: add `mpsc::Receiver::blocking_recv_many` [[#6867]](https://github.com/tokio-rs/tokio/pull/6867)
- task: stabilize `Id` apis [[#6793]](https://github.com/tokio-rs/tokio/pull/6793), [[#6891]](https://github.com/tokio-rs/tokio/pull/6891)

### Added (unstable)

- metrics: add H2 Histogram option to improve histogram granularity [[#6897]](https://github.com/tokio-rs/tokio/pull/6897)
- metrics: rename some histogram apis [[#6924]](https://github.com/tokio-rs/tokio/pull/6924)
- runtime: add `LocalRuntime` [[#6808]](https://github.com/tokio-rs/tokio/pull/6808)

### Changed

- runtime: box futures larger than 16k on release mode [[#6826]](https://github.com/tokio-rs/tokio/pull/6826)
- sync: add `#[must_use]` to `Notified` [[#6828]](https://github.com/tokio-rs/tokio/pull/6828)
- sync: make `watch` cooperative [[#6846]](https://github.com/tokio-rs/tokio/pull/6846)
- sync: make `broadcast::Receiver` cooperative [[#6870]](https://github.com/tokio-rs/tokio/pull/6870)
- task: add task size to tracing instrumentation [[#6881]](https://github.com/tokio-rs/tokio/pull/6881)
- wasm: enable `cfg_fs` for `wasi` target [[#6822]](https://github.com/tokio-rs/tokio/pull/6822)

### Fixed

- net: fix regression of abstract socket path in unix socket [[#6838]](https://github.com/tokio-rs/tokio/pull/6838)

### Documented

- io: recommend `OwnedFd` with `AsyncFd` [[#6821]](https://github.com/tokio-rs/tokio/pull/6821)
- io: document cancel safety of `AsyncFd` methods [[#6890]](https://github.com/tokio-rs/tokio/pull/6890)
- macros: render more comprehensible documentation for `join` and `try_join` [[#6814]](https://github.com/tokio-rs/tokio/pull/6814), [[#6841]](https://github.com/tokio-rs/tokio/pull/6841)
- net: fix swapped examples for `TcpSocket::set_nodelay` and `TcpSocket::nodelay` [[#6840]](https://github.com/tokio-rs/tokio/pull/6840)
- sync: document runtime compatibility [[#6833]](https://github.com/tokio-rs/tokio/pull/6833)

## 1.40.0 (August 30th, 2024)

### Added

- io: add `util::SimplexStream` [[#6589]](https://github.com/tokio-rs/tokio/pull/6589)
- process: stabilize `Command::process_group` [[#6731]](https://github.com/tokio-rs/tokio/pull/6731)
- sync: add `{TrySendError,SendTimeoutError}::into_inner` [[#6755]](https://github.com/tokio-rs/tokio/pull/6755)
- task: add `JoinSet::join_all` [[#6784]](https://github.com/tokio-rs/tokio/pull/6784)

### Added (unstable)

- runtime: add `Builder::{on_task_spawn, on_task_terminate}` [[#6742]](https://github.com/tokio-rs/tokio/pull/6742)

### Changed

- io: use vectored io for `write_all_buf` when possible [[#6724]](https://github.com/tokio-rs/tokio/pull/6724)
- runtime: prevent niche-optimization to avoid triggering miri [[#6744]](https://github.com/tokio-rs/tokio/pull/6744)
- sync: mark mpsc types as `UnwindSafe` [[#6783]](https://github.com/tokio-rs/tokio/pull/6783)
- sync,time: make `Sleep` and `BatchSemaphore` instrumentation explicit roots [[#6727]](https://github.com/tokio-rs/tokio/pull/6727)
- task: use `NonZeroU64` for `task::Id` [[#6733]](https://github.com/tokio-rs/tokio/pull/6733)
- task: include panic message when printing `JoinError` [[#6753]](https://github.com/tokio-rs/tokio/pull/6753)
- task: add `#[must_use]` to `JoinHandle::abort_handle` [[#6762]](https://github.com/tokio-rs/tokio/pull/6762)
- time: eliminate timer wheel allocations [[#6779]](https://github.com/tokio-rs/tokio/pull/6779)

### Documented

- docs: clarify that `[build]` section doesn't go in Cargo.toml [[#6728]](https://github.com/tokio-rs/tokio/pull/6728)
- io: clarify zero remaining capacity case [[#6790]](https://github.com/tokio-rs/tokio/pull/6790)
- macros: improve documentation for `select!` [[#6774]](https://github.com/tokio-rs/tokio/pull/6774)
- sync: document mpsc channel allocation behavior [[#6773]](https://github.com/tokio-rs/tokio/pull/6773)

## 1.39.3 (August 17th, 2024)

This release fixes a regression where the unix socket api stopped accepting the abstract socket namespace. [[#6772]](https://github.com/tokio-rs/tokio/pull/6772)

## 1.39.2 (July 27th, 2024)

This release fixes a regression where the `select!` macro stopped accepting expressions that make use of temporary lifetime extension. [[#6722]](https://github.com/tokio-rs/tokio/pull/6722)

## 1.39.1 (July 23rd, 2024)

This release reverts "time: avoid traversing entries in the time wheel twice" because it contains a bug. [[#6715]](https://github.com/tokio-rs/tokio/pull/6715)

## 1.39.0 (July 23rd, 2024)

Yanked. Please use 1.39.1 instead.

- This release bumps the MSRV to 1.70. [[#6645]](https://github.com/tokio-rs/tokio/pull/6645)
- This release upgrades to mio v1. [[#6635]](https://github.com/tokio-rs/tokio/pull/6635)
- This release upgrades to windows-sys v0.52 [[#6154]](https://github.com/tokio-rs/tokio/pull/6154)

### Added

- io: implement `AsyncSeek` for `Empty` [[#6663]](https://github.com/tokio-rs/tokio/pull/6663)
- metrics: stabilize `num_alive_tasks` [[#6619]](https://github.com/tokio-rs/tokio/pull/6619), [[#6667]](https://github.com/tokio-rs/tokio/pull/6667)
- process: add `Command::as_std_mut` [[#6608]](https://github.com/tokio-rs/tokio/pull/6608)
- sync: add `watch::Sender::same_channel` [[#6637]](https://github.com/tokio-rs/tokio/pull/6637)
- sync: add `{Receiver,UnboundedReceiver}::{sender_strong_count,sender_weak_count}` [[#6661]](https://github.com/tokio-rs/tokio/pull/6661)
- sync: implement `Default` for `watch::Sender` [[#6626]](https://github.com/tokio-rs/tokio/pull/6626)
- task: implement `Clone` for `AbortHandle` [[#6621]](https://github.com/tokio-rs/tokio/pull/6621)
- task: stabilize `consume_budget` [[#6622]](https://github.com/tokio-rs/tokio/pull/6622)

### Changed

- io: improve panic message of `ReadBuf::put_slice()` [[#6629]](https://github.com/tokio-rs/tokio/pull/6629)
- io: read during write in `copy_bidirectional` and `copy` [[#6532]](https://github.com/tokio-rs/tokio/pull/6532)
- runtime: replace `num_cpus` with `available_parallelism` [[#6709]](https://github.com/tokio-rs/tokio/pull/6709)
- task: avoid stack overflow when passing large future to `block_on` [[#6692]](https://github.com/tokio-rs/tokio/pull/6692)
- time: avoid traversing entries in the time wheel twice [[#6584]](https://github.com/tokio-rs/tokio/pull/6584)
- time: support `IntoFuture` with `timeout` [[#6666]](https://github.com/tokio-rs/tokio/pull/6666)
- macros: support `IntoFuture` with `join!` and `select!` [[#6710]](https://github.com/tokio-rs/tokio/pull/6710)

### Fixed

- docs: fix docsrs builds with the fs feature enabled [[#6585]](https://github.com/tokio-rs/tokio/pull/6585)
- io: only use short-read optimization on known-to-be-compatible platforms [[#6668]](https://github.com/tokio-rs/tokio/pull/6668)
- time: fix overflow panic when using large durations with `Interval` [[#6612]](https://github.com/tokio-rs/tokio/pull/6612)

### Added (unstable)

- macros: allow `unhandled_panic` behavior for `#[tokio::main]` and `#[tokio::test]` [[#6593]](https://github.com/tokio-rs/tokio/pull/6593)
- metrics: add `spawned_tasks_count` [[#6114]](https://github.com/tokio-rs/tokio/pull/6114)
- metrics: add `worker_park_unpark_count` [[#6696]](https://github.com/tokio-rs/tokio/pull/6696)
- metrics: add worker thread id [[#6695]](https://github.com/tokio-rs/tokio/pull/6695)

### Documented

- io: update `tokio::io::stdout` documentation [[#6674]](https://github.com/tokio-rs/tokio/pull/6674)
- macros: typo fix in `join.rs` and `try_join.rs` [[#6641]](https://github.com/tokio-rs/tokio/pull/6641)
- runtime: fix typo in `unhandled_panic` [[#6660]](https://github.com/tokio-rs/tokio/pull/6660)
- task: document behavior of `JoinSet::try_join_next` when all tasks are running [[#6671]](https://github.com/tokio-rs/tokio/pull/6671)

## 1.38.2 (April 2nd, 2025)

This release fixes a soundness issue in the broadcast channel. The channel accepts values that are `Send` but `!Sync`. Previously, the channel called `clone()` on these values without synchronizing. This release fixes the channel by synchronizing calls to `.clone()` (Thanks Austin Bonander for finding and reporting the issue).

### Fixed

- sync: synchronize `clone()` call in broadcast channel [[#7232]](https://github.com/tokio-rs/tokio/pull/7232)

## 1.38.1 (July 16th, 2024)

This release fixes the bug identified as [[#6682]](https://github.com/tokio-rs/tokio/pull/6682), which caused timers not to fire when they should.

### Fixed

- time: update `wake_up` while holding all the locks of sharded time wheels [[#6683]](https://github.com/tokio-rs/tokio/pull/6683)

## 1.38.0 (May 30th, 2024)

This release marks the beginning of stabilization for runtime metrics. It stabilizes `RuntimeMetrics::worker_count`. Future releases will continue to stabilize more metrics.

### Added

- fs: add `File::create_new` [[#6573]](https://github.com/tokio-rs/tokio/pull/6573)
- io: add `copy_bidirectional_with_sizes` [[#6500]](https://github.com/tokio-rs/tokio/pull/6500)
- io: implement `AsyncBufRead` for `Join` [[#6449]](https://github.com/tokio-rs/tokio/pull/6449)
- net: add Apple visionOS support [[#6465]](https://github.com/tokio-rs/tokio/pull/6465)
- net: implement `Clone` for `NamedPipeInfo` [[#6586]](https://github.com/tokio-rs/tokio/pull/6586)
- net: support QNX OS [[#6421]](https://github.com/tokio-rs/tokio/pull/6421)
- sync: add `Notify::notify_last` [[#6520]](https://github.com/tokio-rs/tokio/pull/6520)
- sync: add `mpsc::Receiver::{capacity,max_capacity}` [[#6511]](https://github.com/tokio-rs/tokio/pull/6511)
- sync: add `split` method to the semaphore permit [[#6472]](https://github.com/tokio-rs/tokio/pull/6472), [[#6478]](https://github.com/tokio-rs/tokio/pull/6478)
- task: add `tokio::task::join_set::Builder::spawn_blocking` [[#6578]](https://github.com/tokio-rs/tokio/pull/6578)
- wasm: support rt-multi-thread with wasm32-wasi-preview1-threads [[#6510]](https://github.com/tokio-rs/tokio/pull/6510)

### Changed

- macros: make `#[tokio::test]` append `#[test]` at the end of the attribute list [[#6497]](https://github.com/tokio-rs/tokio/pull/6497)
- metrics: fix `blocking_threads` count [[#6551]](https://github.com/tokio-rs/tokio/pull/6551)
- metrics: stabilize `RuntimeMetrics::worker_count` [[#6556]](https://github.com/tokio-rs/tokio/pull/6556)
- runtime: move task out of the `lifo_slot` in `block_in_place` [[#6596]](https://github.com/tokio-rs/tokio/pull/6596)
- runtime: panic if `global_queue_interval` is zero [[#6445]](https://github.com/tokio-rs/tokio/pull/6445)
- sync: always drop message in destructor for oneshot receiver [[#6558]](https://github.com/tokio-rs/tokio/pull/6558)
- sync: instrument `Semaphore` for task dumps [[#6499]](https://github.com/tokio-rs/tokio/pull/6499)
- sync: use FIFO ordering when waking batches of wakers [[#6521]](https://github.com/tokio-rs/tokio/pull/6521)
- task: make `LocalKey::get` work with Clone types [[#6433]](https://github.com/tokio-rs/tokio/pull/6433)
- tests: update nix and mio-aio dev-dependencies [[#6552]](https://github.com/tokio-rs/tokio/pull/6552)
- time: clean up implementation [[#6517]](https://github.com/tokio-rs/tokio/pull/6517)
- time: lazily init timers on first poll [[#6512]](https://github.com/tokio-rs/tokio/pull/6512)
- time: remove the `true_when` field in `TimerShared` [[#6563]](https://github.com/tokio-rs/tokio/pull/6563)
- time: use sharding for timer implementation [[#6534]](https://github.com/tokio-rs/tokio/pull/6534)

### Fixed

- taskdump: allow building taskdump docs on non-unix machines [[#6564]](https://github.com/tokio-rs/tokio/pull/6564)
- time: check for overflow in `Interval::poll_tick` [[#6487]](https://github.com/tokio-rs/tokio/pull/6487)
- sync: fix incorrect `is_empty` on mpsc block boundaries [[#6603]](https://github.com/tokio-rs/tokio/pull/6603)

### Documented

- fs: rewrite file system docs [[#6467]](https://github.com/tokio-rs/tokio/pull/6467)
- io: fix `stdin` documentation [[#6581]](https://github.com/tokio-rs/tokio/pull/6581)
- io: fix obsolete reference in `ReadHalf::unsplit()` documentation [[#6498]](https://github.com/tokio-rs/tokio/pull/6498)
- macros: render more comprehensible documentation for `select!` [[#6468]](https://github.com/tokio-rs/tokio/pull/6468)
- net: add missing types to module docs [[#6482]](https://github.com/tokio-rs/tokio/pull/6482)
- net: fix misleading `NamedPipeServer` example [[#6590]](https://github.com/tokio-rs/tokio/pull/6590)
- sync: add examples for `SemaphorePermit`, `OwnedSemaphorePermit` [[#6477]](https://github.com/tokio-rs/tokio/pull/6477)
- sync: document that `Barrier::wait` is not cancel safe [[#6494]](https://github.com/tokio-rs/tokio/pull/6494)
- sync: explain relation between `watch::Sender::{subscribe,closed}` [[#6490]](https://github.com/tokio-rs/tokio/pull/6490)
- task: clarify that you can't abort `spawn_blocking` tasks [[#6571]](https://github.com/tokio-rs/tokio/pull/6571)
- task: fix a typo in doc of `LocalSet::run_until` [[#6599]](https://github.com/tokio-rs/tokio/pull/6599)
- time: fix test-util requirement for pause and resume in docs [[#6503]](https://github.com/tokio-rs/tokio/pull/6503)

## 1.37.0 (March 28th, 2024)

### Added

- fs: add `set_max_buf_size` to `tokio::fs::File` [[#6411]](https://github.com/tokio-rs/tokio/pull/6411)
- io: add `try_new` and `try_with_interest` to `AsyncFd` [[#6345]](https://github.com/tokio-rs/tokio/pull/6345)
- sync: add `forget_permits` method to semaphore [[#6331]](https://github.com/tokio-rs/tokio/pull/6331)
- sync: add `is_closed`, `is_empty`, and `len` to mpsc receivers [[#6348]](https://github.com/tokio-rs/tokio/pull/6348)
- sync: add a `rwlock()` method to owned `RwLock` guards [[#6418]](https://github.com/tokio-rs/tokio/pull/6418)
- sync: expose strong and weak counts of mpsc sender handles [[#6405]](https://github.com/tokio-rs/tokio/pull/6405)
- sync: implement `Clone` for `watch::Sender` [[#6388]](https://github.com/tokio-rs/tokio/pull/6388)
- task: add `TaskLocalFuture::take_value` [[#6340]](https://github.com/tokio-rs/tokio/pull/6340)
- task: implement `FromIterator` for `JoinSet` [[#6300]](https://github.com/tokio-rs/tokio/pull/6300)

### Changed

- io: make `io::split` use a mutex instead of a spinlock [[#6403]](https://github.com/tokio-rs/tokio/pull/6403)

### Fixed

- docs: fix docsrs build without net feature [[#6360]](https://github.com/tokio-rs/tokio/pull/6360)
- macros: allow select with only else branch [[#6339]](https://github.com/tokio-rs/tokio/pull/6339)
- runtime: fix leaking registration entries when os registration fails [[#6329]](https://github.com/tokio-rs/tokio/pull/6329)

### Documented

- io: document cancel safety of `AsyncBufReadExt::fill_buf` [[#6431]](https://github.com/tokio-rs/tokio/pull/6431)
- io: document cancel safety of `AsyncReadExt`'s primitive read functions [[#6337]](https://github.com/tokio-rs/tokio/pull/6337)
- runtime: add doc link from `Runtime` to `#[tokio::main]` [[#6366]](https://github.com/tokio-rs/tokio/pull/6366)
- runtime: make the `enter` example deterministic [[#6351]](https://github.com/tokio-rs/tokio/pull/6351)
- sync: add Semaphore example for limiting the number of outgoing requests [[#6419]](https://github.com/tokio-rs/tokio/pull/6419)
- sync: fix missing period in broadcast docs [[#6377]](https://github.com/tokio-rs/tokio/pull/6377)
- sync: mark `mpsc::Sender::downgrade` with `#[must_use]` [[#6326]](https://github.com/tokio-rs/tokio/pull/6326)
- sync: reorder `const_new` before `new_with` [[#6392]](https://github.com/tokio-rs/tokio/pull/6392)
- sync: update watch channel docs [[#6395]](https://github.com/tokio-rs/tokio/pull/6395)
- task: fix documentation links [[#6336]](https://github.com/tokio-rs/tokio/pull/6336)

### Changed (unstable)

- runtime: include task `Id` in taskdumps [[#6328]](https://github.com/tokio-rs/tokio/pull/6328)
- runtime: panic if `unhandled_panic` is enabled when not supported [[#6410]](https://github.com/tokio-rs/tokio/pull/6410)

## 1.36.0 (February 2nd, 2024)

### Added

- io: add `tokio::io::Join` [[#6220]](https://github.com/tokio-rs/tokio/pull/6220)
- io: implement `AsyncWrite` for `Empty` [[#6235]](https://github.com/tokio-rs/tokio/pull/6235)
- net: add support for anonymous unix pipes [[#6127]](https://github.com/tokio-rs/tokio/pull/6127)
- net: add `UnixSocket` [[#6290]](https://github.com/tokio-rs/tokio/pull/6290)
- net: expose keepalive option on `TcpSocket` [[#6311]](https://github.com/tokio-rs/tokio/pull/6311)
- sync: add `{Receiver,UnboundedReceiver}::poll_recv_many` [[#6236]](https://github.com/tokio-rs/tokio/pull/6236)
- sync: add `Sender::{try_,}reserve_many` [[#6205]](https://github.com/tokio-rs/tokio/pull/6205)
- sync: add `watch::Receiver::mark_unchanged` [[#6252]](https://github.com/tokio-rs/tokio/pull/6252)
- task: add `JoinSet::try_join_next` [[#6280]](https://github.com/tokio-rs/tokio/pull/6280)

### Changed

- io: make `copy` cooperative [[#6265]](https://github.com/tokio-rs/tokio/pull/6265)
- io: make `repeat` and `sink` cooperative [[#6254]](https://github.com/tokio-rs/tokio/pull/6254)
- io: simplify check for empty slice [[#6293]](https://github.com/tokio-rs/tokio/pull/6293)
- process: use pidfd on Linux when available [[#6152]](https://github.com/tokio-rs/tokio/pull/6152)
- sync: use AtomicBool in broadcast channel future [[#6298]](https://github.com/tokio-rs/tokio/pull/6298)

### Documented

- io: clarify `clear_ready` docs [[#6304]](https://github.com/tokio-rs/tokio/pull/6304)
- net: document that `*Fd` traits on `TcpSocket` are unix-only [[#6294]](https://github.com/tokio-rs/tokio/pull/6294)
- sync: document FIFO behavior of `tokio::sync::Mutex` [[#6279]](https://github.com/tokio-rs/tokio/pull/6279)
- chore: typographic improvements [[#6262]](https://github.com/tokio-rs/tokio/pull/6262)
- runtime: remove obsolete comment [[#6303]](https://github.com/tokio-rs/tokio/pull/6303)
- task: fix typo [[#6261]](https://github.com/tokio-rs/tokio/pull/6261)

## 1.35.1 (December 19, 2023)

This is a forward part of a change that was backported to 1.25.3.

### Fixed

- io: add budgeting to `tokio::runtime::io::registration::async_io` [[#6221]](https://github.com/tokio-rs/tokio/pull/6221)

## 1.35.0 (December 8th, 2023)

### Added

- net: add Apple watchOS support [[#6176]](https://github.com/tokio-rs/tokio/pull/6176)

### Changed

- io: drop the `Sized` requirements from `AsyncReadExt.read_buf` [[#6169]](https://github.com/tokio-rs/tokio/pull/6169)
- runtime: make `Runtime` unwind safe [[#6189]](https://github.com/tokio-rs/tokio/pull/6189)
- runtime: reduce the lock contention in task spawn [[#6001]](https://github.com/tokio-rs/tokio/pull/6001)
- tokio: update nix dependency to 0.27.1 [[#6190]](https://github.com/tokio-rs/tokio/pull/6190)

### Fixed

- chore: make `--cfg docsrs` work without net feature [[#6166]](https://github.com/tokio-rs/tokio/pull/6166)
- chore: use relaxed load for `unsync_load` on miri [[#6179]](https://github.com/tokio-rs/tokio/pull/6179)
- runtime: handle missing context on wake [[#6148]](https://github.com/tokio-rs/tokio/pull/6148)
- taskdump: fix taskdump cargo config example [[#6150]](https://github.com/tokio-rs/tokio/pull/6150)
- taskdump: skip notified tasks during taskdumps [[#6194]](https://github.com/tokio-rs/tokio/pull/6194)
- tracing: avoid creating resource spans with current parent, use a None parent instead [[#6107]](https://github.com/tokio-rs/tokio/pull/6107)
- tracing: make task span explicit root [[#6158]](https://github.com/tokio-rs/tokio/pull/6158)

### Documented

- io: flush in `AsyncWriteExt` examples [[#6149]](https://github.com/tokio-rs/tokio/pull/6149)
- runtime: document fairness guarantees and current behavior [[#6145]](https://github.com/tokio-rs/tokio/pull/6145)
- task: document cancel safety of `LocalSet::run_until` [[#6147]](https://github.com/tokio-rs/tokio/pull/6147)

## 1.34.0 (November 19th, 2023)

### Fixed

- io: allow `clear_readiness` after io driver shutdown [[#6067]](https://github.com/tokio-rs/tokio/pull/6067)
- io: fix integer overflow in `take` [[#6080]](https://github.com/tokio-rs/tokio/pull/6080)
- io: fix I/O resource hang [[#6134]](https://github.com/tokio-rs/tokio/pull/6134)
- sync: fix `broadcast::channel` link [[#6100]](https://github.com/tokio-rs/tokio/pull/6100)

### Changed

- macros: use `::core` qualified imports instead of `::std` inside `tokio::test` macro [[#5973]](https://github.com/tokio-rs/tokio/pull/5973)

### Added

- fs: update cfg attr in `fs::read_dir` to include `aix` [[#6075]](https://github.com/tokio-rs/tokio/pull/6075)
- sync: add `mpsc::Receiver::recv_many` [[#6010]](https://github.com/tokio-rs/tokio/pull/6010)
- tokio: added vita target support [[#6094]](https://github.com/tokio-rs/tokio/pull/6094)

## 1.33.0 (October 9, 2023)

### Fixed

- io: mark `Interest::add` with `#[must_use]` [[#6037]](https://github.com/tokio-rs/tokio/pull/6037)
- runtime: fix cache line size for RISC-V [[#5994]](https://github.com/tokio-rs/tokio/pull/5994)
- sync: prevent lock poisoning in `watch::Receiver::wait_for` [[#6021]](https://github.com/tokio-rs/tokio/pull/6021)
- task: fix `spawn_local` source location [[#5984]](https://github.com/tokio-rs/tokio/pull/5984)

### Changed

- sync: use Acquire/Release orderings instead of SeqCst in `watch` [[#6018]](https://github.com/tokio-rs/tokio/pull/6018)

### Added

- fs: add vectored writes to `tokio::fs::File` [[#5958]](https://github.com/tokio-rs/tokio/pull/5958)
- io: add `Interest::remove` method [[#5906]](https://github.com/tokio-rs/tokio/pull/5906)
- io: add vectored writes to `DuplexStream` [[#5985]](https://github.com/tokio-rs/tokio/pull/5985)
- net: add Apple tvOS support [[#6045]](https://github.com/tokio-rs/tokio/pull/6045)
- sync: add `?Sized` bound to `{MutexGuard,OwnedMutexGuard}::map` [[#5997]](https://github.com/tokio-rs/tokio/pull/5997)
- sync: add `watch::Receiver::mark_unseen` [[#5962]](https://github.com/tokio-rs/tokio/pull/5962), [[#6014]](https://github.com/tokio-rs/tokio/pull/6014), [[#6017]](https://github.com/tokio-rs/tokio/pull/6017)
- sync: add `watch::Sender::new` [[#5998]](https://github.com/tokio-rs/tokio/pull/5998)
- sync: add const fn `OnceCell::from_value` [[#5903]](https://github.com/tokio-rs/tokio/pull/5903)

### Removed

- remove unused `stats` feature [[#5952]](https://github.com/tokio-rs/tokio/pull/5952)

### Documented

- add missing backticks in code examples [[#5938]](https://github.com/tokio-rs/tokio/pull/5938), [[#6056]](https://github.com/tokio-rs/tokio/pull/6056)
- fix typos [[#5988]](https://github.com/tokio-rs/tokio/pull/5988), [[#6030]](https://github.com/tokio-rs/tokio/pull/6030)
- process: document that `Child::wait` is cancel safe [[#5977]](https://github.com/tokio-rs/tokio/pull/5977)
- sync: add examples for `Semaphore` [[#5939]](https://github.com/tokio-rs/tokio/pull/5939), [[#5956]](https://github.com/tokio-rs/tokio/pull/5956), [[#5978]](https://github.com/tokio-rs/tokio/pull/5978), [[#6031]](https://github.com/tokio-rs/tokio/pull/6031), [[#6032]](https://github.com/tokio-rs/tokio/pull/6032), [[#6050]](https://github.com/tokio-rs/tokio/pull/6050)
- sync: document that `broadcast` capacity is a lower bound [[#6042]](https://github.com/tokio-rs/tokio/pull/6042)
- sync: document that `const_new` is not instrumented [[#6002]](https://github.com/tokio-rs/tokio/pull/6002)
- sync: improve cancel-safety documentation for `mpsc::Sender::send` [[#5947]](https://github.com/tokio-rs/tokio/pull/5947)
- sync: improve docs for `watch` channel [[#5954]](https://github.com/tokio-rs/tokio/pull/5954)
- taskdump: render taskdump documentation on docs.rs [[#5972]](https://github.com/tokio-rs/tokio/pull/5972)

### Unstable

- taskdump: fix potential deadlock [[#6036]](https://github.com/tokio-rs/tokio/pull/6036)

## 1.32.1 (December 19, 2023)

This is a forward part of a change that was backported to 1.25.3.

### Fixed

- io: add budgeting to `tokio::runtime::io::registration::async_io` [[#6221]](https://github.com/tokio-rs/tokio/pull/6221)

## 1.32.0 (August 16, 2023)

### Fixed

- sync: fix potential quadratic behavior in `broadcast::Receiver` [[#5925]](https://github.com/tokio-rs/tokio/pull/5925)

### Added

- process: stabilize `Command::raw_arg` [[#5930]](https://github.com/tokio-rs/tokio/pull/5930)
- io: enable awaiting error readiness [[#5781]](https://github.com/tokio-rs/tokio/pull/5781)

### Unstable

- rt(alt): improve scalability of alt runtime as the number of cores grows [[#5935]](https://github.com/tokio-rs/tokio/pull/5935)

## 1.31.0 (August 10, 2023)

### Fixed

- io: delegate `WriteHalf::poll_write_vectored` [[#5914]](https://github.com/tokio-rs/tokio/pull/5914)

### Unstable

- rt(alt): fix memory leak in unstable next-gen scheduler prototype [[#5911]](https://github.com/tokio-rs/tokio/pull/5911)
- rt: expose mean task poll time metric [[#5927]](https://github.com/tokio-rs/tokio/pull/5927)

## 1.30.0 (August 9, 2023)

This release bumps the MSRV of Tokio to 1.63. [[#5887]](https://github.com/tokio-rs/tokio/pull/5887)

### Changed

- tokio: reduce LLVM code generation [[#5859]](https://github.com/tokio-rs/tokio/pull/5859)
- io: support `--cfg mio_unsupported_force_poll_poll` flag [[#5881]](https://github.com/tokio-rs/tokio/pull/5881)
- sync: make `const_new` methods always available [[#5885]](https://github.com/tokio-rs/tokio/pull/5885)
- sync: avoid false sharing in mpsc channel [[#5829]](https://github.com/tokio-rs/tokio/pull/5829)
- rt: pop at least one task from inject queue [[#5908]](https://github.com/tokio-rs/tokio/pull/5908)

### Added

- sync: add `broadcast::Sender::new` [[#5824]](https://github.com/tokio-rs/tokio/pull/5824)
- net: implement `UCred` for espidf [[#5868]](https://github.com/tokio-rs/tokio/pull/5868)
- fs: add `File::options()` [[#5869]](https://github.com/tokio-rs/tokio/pull/5869)
- time: implement extra reset variants for `Interval` [[#5878]](https://github.com/tokio-rs/tokio/pull/5878)
- process: add `{ChildStd*}::into_owned_{fd, handle}` [[#5899]](https://github.com/tokio-rs/tokio/pull/5899)

### Removed

- tokio: removed unused `tokio_*` cfgs [[#5890]](https://github.com/tokio-rs/tokio/pull/5890)
- remove build script to speed up compilation [[#5887]](https://github.com/tokio-rs/tokio/pull/5887)

### Documented

- sync: mention lagging in docs for `broadcast::send` [[#5820]](https://github.com/tokio-rs/tokio/pull/5820)
- runtime: expand on sharing runtime docs [[#5858]](https://github.com/tokio-rs/tokio/pull/5858)
- io: use vec in example for `AsyncReadExt::read_exact` [[#5863]](https://github.com/tokio-rs/tokio/pull/5863)
- time: mark `Sleep` as `!Unpin` in docs [[#5916]](https://github.com/tokio-rs/tokio/pull/5916)
- process: fix `raw_arg` not showing up in docs [[#5865]](https://github.com/tokio-rs/tokio/pull/5865)

### Unstable

- rt: add runtime ID [[#5864]](https://github.com/tokio-rs/tokio/pull/5864)
- rt: initial implementation of new threaded runtime [[#5823]](https://github.com/tokio-rs/tokio/pull/5823)

## 1.29.1 (June 29, 2023)

### Fixed

- rt: fix nesting two `block_in_place` with a `block_on` between [[#5837]](https://github.com/tokio-rs/tokio/pull/5837)

## 1.29.0 (June 27, 2023)

Technically a breaking change, the `Send` implementation is removed from `runtime::EnterGuard`. This change fixes a bug and should not impact most users.

### Breaking

- rt: `EnterGuard` should not be `Send` [[#5766]](https://github.com/tokio-rs/tokio/pull/5766)

### Fixed

- fs: reduce blocking ops in `fs::read_dir` [[#5653]](https://github.com/tokio-rs/tokio/pull/5653)
- rt: fix possible starvation [[#5686]](https://github.com/tokio-rs/tokio/pull/5686), [[#5712]](https://github.com/tokio-rs/tokio/pull/5712)
- rt: fix stacked borrows issue in `JoinSet` [[#5693]](https://github.com/tokio-rs/tokio/pull/5693)
- rt: panic if `EnterGuard` dropped incorrect order [[#5772]](https://github.com/tokio-rs/tokio/pull/5772)
- time: do not overflow to signal value [[#5710]](https://github.com/tokio-rs/tokio/pull/5710)
- fs: wait for in-flight ops before cloning `File` [[#5803]](https://github.com/tokio-rs/tokio/pull/5803)

### Changed

- rt: reduce time to poll tasks scheduled from outside the runtime [[#5705]](https://github.com/tokio-rs/tokio/pull/5705), [[#5720]](https://github.com/tokio-rs/tokio/pull/5720)

### Added

- net: add uds doc alias for unix sockets [[#5659]](https://github.com/tokio-rs/tokio/pull/5659)
- rt: add metric for number of tasks [[#5628]](https://github.com/tokio-rs/tokio/pull/5628)
- sync: implement more traits for channel errors [[#5666]](https://github.com/tokio-rs/tokio/pull/5666)
- net: add nodelay methods on TcpSocket [[#5672]](https://github.com/tokio-rs/tokio/pull/5672)
- sync: add `broadcast::Receiver::blocking_recv` [[#5690]](https://github.com/tokio-rs/tokio/pull/5690)
- process: add `raw_arg` method to `Command` [[#5704]](https://github.com/tokio-rs/tokio/pull/5704)
- io: support PRIORITY epoll events [[#5566]](https://github.com/tokio-rs/tokio/pull/5566)
- task: add `JoinSet::poll_join_next` [[#5721]](https://github.com/tokio-rs/tokio/pull/5721)
- net: add support for Redox OS [[#5790]](https://github.com/tokio-rs/tokio/pull/5790)

### Unstable

- rt: add the ability to dump task backtraces [[#5608]](https://github.com/tokio-rs/tokio/pull/5608), [[#5676]](https://github.com/tokio-rs/tokio/pull/5676), [[#5708]](https://github.com/tokio-rs/tokio/pull/5708), [[#5717]](https://github.com/tokio-rs/tokio/pull/5717)
- rt: instrument task poll times with a histogram [[#5685]](https://github.com/tokio-rs/tokio/pull/5685)

## 1.28.2 (May 28, 2023)

Forward ports 1.18.6 changes.

### Fixed

- deps: disable default features for mio [[#5728]](https://github.com/tokio-rs/tokio/pull/5728)

## 1.28.1 (May 10th, 2023)

This release fixes a mistake in the build script that makes `AsFd` implementations unavailable on Rust 1.63. [[#5677]](https://github.com/tokio-rs/tokio/pull/5677)

## 1.28.0 (April 25th, 2023)

### Added

- io: add `AsyncFd::async_io` [[#5542]](https://github.com/tokio-rs/tokio/pull/5542)
- io: impl BufMut for ReadBuf [[#5590]](https://github.com/tokio-rs/tokio/pull/5590)
- net: add `recv_buf` for `UdpSocket` and `UnixDatagram` [[#5583]](https://github.com/tokio-rs/tokio/pull/5583)
- sync: add `OwnedSemaphorePermit::semaphore` [[#5618]](https://github.com/tokio-rs/tokio/pull/5618)
- sync: add `same_channel` to broadcast channel [[#5607]](https://github.com/tokio-rs/tokio/pull/5607)
- sync: add `watch::Receiver::wait_for` [[#5611]](https://github.com/tokio-rs/tokio/pull/5611)
- task: add `JoinSet::spawn_blocking` and `JoinSet::spawn_blocking_on` [[#5612]](https://github.com/tokio-rs/tokio/pull/5612)

### Changed

- deps: update windows-sys to 0.48 [[#5591]](https://github.com/tokio-rs/tokio/pull/5591)
- io: make `read_to_end` not grow unnecessarily [[#5610]](https://github.com/tokio-rs/tokio/pull/5610)
- macros: make entrypoints more efficient [[#5621]](https://github.com/tokio-rs/tokio/pull/5621)
- sync: improve Debug impl for `RwLock` [[#5647]](https://github.com/tokio-rs/tokio/pull/5647)
- sync: reduce contention in `Notify` [[#5503]](https://github.com/tokio-rs/tokio/pull/5503)

### Fixed

- net: support `get_peer_cred` on AIX [[#5065]](https://github.com/tokio-rs/tokio/pull/5065)
- sync: avoid deadlocks in `broadcast` with custom wakers [[#5578]](https://github.com/tokio-rs/tokio/pull/5578)

### Documented

- sync: fix typo in `Semaphore::MAX_PERMITS` [[#5645]](https://github.com/tokio-rs/tokio/pull/5645)
- sync: fix typo in `tokio::sync::watch::Sender` docs [[#5587]](https://github.com/tokio-rs/tokio/pull/5587)

## 1.27.0 (March 27th, 2023)

This release bumps the MSRV of Tokio to 1.56. [[#5559]](https://github.com/tokio-rs/tokio/pull/5559)

### Added

- io: add `async_io` helper method to sockets [[#5512]](https://github.com/tokio-rs/tokio/pull/5512)
- io: add implementations of `AsFd`/`AsHandle`/`AsSocket` [[#5514]](https://github.com/tokio-rs/tokio/pull/5514), [[#5540]](https://github.com/tokio-rs/tokio/pull/5540)
- net: add `UdpSocket::peek_sender()` [[#5520]](https://github.com/tokio-rs/tokio/pull/5520)
- sync: add `RwLockWriteGuard::{downgrade_map, try_downgrade_map}` [[#5527]](https://github.com/tokio-rs/tokio/pull/5527)
- task: add `JoinHandle::abort_handle` [[#5543]](https://github.com/tokio-rs/tokio/pull/5543)

### Changed

- io: use `memchr` from `libc` [[#5558]](https://github.com/tokio-rs/tokio/pull/5558)
- macros: accept path as crate rename in `#[tokio::main]` [[#5557]](https://github.com/tokio-rs/tokio/pull/5557)
- macros: update to syn 2.0.0 [[#5572]](https://github.com/tokio-rs/tokio/pull/5572)
- time: don't register for a wakeup when `Interval` returns `Ready` [[#5553]](https://github.com/tokio-rs/tokio/pull/5553)

### Fixed

- fs: fuse std iterator in `ReadDir` [[#5555]](https://github.com/tokio-rs/tokio/pull/5555)
- tracing: fix `spawn_blocking` location fields [[#5573]](https://github.com/tokio-rs/tokio/pull/5573)
- time: clean up redundant check in `Wheel::poll()` [[#5574]](https://github.com/tokio-rs/tokio/pull/5574)

### Documented

- macros: define cancellation safety [[#5525]](https://github.com/tokio-rs/tokio/pull/5525)
- io: add details to docs of `tokio::io::copy[_buf]` [[#5575]](https://github.com/tokio-rs/tokio/pull/5575)
- io: refer to `ReaderStream` and `StreamReader` in module docs [[#5576]](https://github.com/tokio-rs/tokio/pull/5576)

## 1.26.0 (March 1st, 2023)

### Fixed

- macros: fix empty `join!` and `try_join!` [[#5504]](https://github.com/tokio-rs/tokio/pull/5504)
- sync: don't leak tracing spans in mutex guards [[#5469]](https://github.com/tokio-rs/tokio/pull/5469)
- sync: drop wakers after unlocking the mutex in Notify [[#5471]](https://github.com/tokio-rs/tokio/pull/5471)
- sync: drop wakers outside lock in semaphore [[#5475]](https://github.com/tokio-rs/tokio/pull/5475)

### Added

- fs: add `fs::try_exists` [[#4299]](https://github.com/tokio-rs/tokio/pull/4299)
- net: add types for named unix pipes [[#5351]](https://github.com/tokio-rs/tokio/pull/5351)
- sync: add `MappedOwnedMutexGuard` [[#5474]](https://github.com/tokio-rs/tokio/pull/5474)

### Changed

- chore: update windows-sys to 0.45 [[#5386]](https://github.com/tokio-rs/tokio/pull/5386)
- net: use Message Read Mode for named pipes [[#5350]](https://github.com/tokio-rs/tokio/pull/5350)
- sync: mark lock guards with `#[clippy::has_significant_drop]` [[#5422]](https://github.com/tokio-rs/tokio/pull/5422)
- sync: reduce contention in watch channel [[#5464]](https://github.com/tokio-rs/tokio/pull/5464)
- time: remove cache padding in timer entries [[#5468]](https://github.com/tokio-rs/tokio/pull/5468)
- time: Improve `Instant::now()` perf with test-util [[#5513]](https://github.com/tokio-rs/tokio/pull/5513)

### Internal Changes

- io: use `poll_fn` in `copy_bidirectional` [[#5486]](https://github.com/tokio-rs/tokio/pull/5486)
- net: refactor named pipe builders to not use bitfields [[#5477]](https://github.com/tokio-rs/tokio/pull/5477)
- rt: remove Arc from Clock [[#5434]](https://github.com/tokio-rs/tokio/pull/5434)
- sync: make `notify_waiters` calls atomic [[#5458]](https://github.com/tokio-rs/tokio/pull/5458)
- time: don't store deadline twice in sleep entries [[#5410]](https://github.com/tokio-rs/tokio/pull/5410)

### Unstable

- metrics: add a new metric for budget exhaustion yields [[#5517]](https://github.com/tokio-rs/tokio/pull/5517)

### Documented

- io: improve AsyncFd example [[#5481]](https://github.com/tokio-rs/tokio/pull/5481)
- runtime: document the nature of the main future [[#5494]](https://github.com/tokio-rs/tokio/pull/5494)
- runtime: remove extra period in docs [[#5511]](https://github.com/tokio-rs/tokio/pull/5511)
- signal: updated Documentation for Signals [[#5459]](https://github.com/tokio-rs/tokio/pull/5459)
- sync: add doc aliases for `blocking_*` methods [[#5448]](https://github.com/tokio-rs/tokio/pull/5448)
- sync: fix docs for Send/Sync bounds in broadcast [[#5480]](https://github.com/tokio-rs/tokio/pull/5480)
- sync: document drop behavior for channels [[#5497]](https://github.com/tokio-rs/tokio/pull/5497)
- task: clarify what happens to spawned work during runtime shutdown [[#5394]](https://github.com/tokio-rs/tokio/pull/5394)
- task: clarify `process::Command` docs [[#5413]](https://github.com/tokio-rs/tokio/pull/5413)
- task: fix wording with 'unsend' [[#5452]](https://github.com/tokio-rs/tokio/pull/5452)
- time: document immediate completion guarantee for timeouts [[#5509]](https://github.com/tokio-rs/tokio/pull/5509)
- tokio: document supported platforms [[#5483]](https://github.com/tokio-rs/tokio/pull/5483)

## 1.25.3 (December 17th, 2023)

### Fixed

- io: add budgeting to `tokio::runtime::io::registration::async_io` [[#6221]](https://github.com/tokio-rs/tokio/pull/6221)

## 1.25.2 (September 22, 2023)

Forward ports 1.20.6 changes.

### Changed

- io: use `memchr` from `libc` [[#5960]](https://github.com/tokio-rs/tokio/pull/5960)

## 1.25.1 (May 28, 2023)

Forward ports 1.18.6 changes.

### Fixed

- deps: disable default features for mio [[#5728]](https://github.com/tokio-rs/tokio/pull/5728)

## 1.25.0 (January 28, 2023)

### Fixed

- rt: fix runtime metrics reporting [[#5330]](https://github.com/tokio-rs/tokio/pull/5330)

### Added

- sync: add `broadcast::Sender::len` [[#5343]](https://github.com/tokio-rs/tokio/pull/5343)

### Changed

- fs: increase maximum read buffer size to 2MiB [[#5397]](https://github.com/tokio-rs/tokio/pull/5397)

## 1.24.2 (January 17, 2023)

Forward ports 1.18.5 changes.

### Fixed

- io: fix unsoundness in `ReadHalf::unsplit` [[#5375]](https://github.com/tokio-rs/tokio/pull/5375)

## 1.24.1 (January 6, 2022)

This release fixes a compilation failure on targets without `AtomicU64` when using rustc older than 1.63. [[#5356]](https://github.com/tokio-rs/tokio/pull/5356)

## 1.24.0 (January 5, 2022)

### Fixed

- rt: improve native `AtomicU64` support detection [[#5284]](https://github.com/tokio-rs/tokio/pull/5284)

### Added

- rt: add configuration option for max number of I/O events polled from the OS per tick [[#5186]](https://github.com/tokio-rs/tokio/pull/5186)
- rt: add an environment variable for configuring the default number of worker threads per runtime instance [[#4250]](https://github.com/tokio-rs/tokio/pull/4250)

### Changed

- sync: reduce MPSC channel stack usage [[#5294]](https://github.com/tokio-rs/tokio/pull/5294)
- io: reduce lock contention in I/O operations [[#5300]](https://github.com/tokio-rs/tokio/pull/5300)
- fs: speed up `read_dir()` by chunking operations [[#5309]](https://github.com/tokio-rs/tokio/pull/5309)
- rt: use internal `ThreadId` implementation [[#5329]](https://github.com/tokio-rs/tokio/pull/5329)
- test: don't auto-advance time when a `spawn_blocking` task is running [[#5115]](https://github.com/tokio-rs/tokio/pull/5115)

## 1.23.1 (January 4, 2022)

This release forward ports changes from 1.18.4.

### Fixed

- net: fix Windows named pipe server builder to maintain option when toggling pipe mode [[#5336]](https://github.com/tokio-rs/tokio/pull/5336).

## 1.23.0 (December 5, 2022)

### Fixed

- net: fix Windows named pipe connect [[#5208]](https://github.com/tokio-rs/tokio/pull/5208)
- io: support vectored writes for `ChildStdin` [[#5216]](https://github.com/tokio-rs/tokio/pull/5216)
- io: fix `async fn ready()` false positive for OS-specific events [[#5231]](https://github.com/tokio-rs/tokio/pull/5231)

### Changed

- runtime: `yield_now` defers task until after driver poll [[#5223]](https://github.com/tokio-rs/tokio/pull/5223)
- runtime: reduce amount of codegen needed per spawned task [[#5213]](https://github.com/tokio-rs/tokio/pull/5213)
- windows: replace `winapi` dependency with `windows-sys` [[#5204]](https://github.com/tokio-rs/tokio/pull/5204)

## 1.22.0 (November 17, 2022)

### Added

- runtime: add `Handle::runtime_flavor` [[#5138]](https://github.com/tokio-rs/tokio/pull/5138)
- sync: add `Mutex::blocking_lock_owned` [[#5130]](https://github.com/tokio-rs/tokio/pull/5130)
- sync: add `Semaphore::MAX_PERMITS` [[#5144]](https://github.com/tokio-rs/tokio/pull/5144)
- sync: add `merge()` to semaphore permits [[#4948]](https://github.com/tokio-rs/tokio/pull/4948)
- sync: add `mpsc::WeakUnboundedSender` [[#5189]](https://github.com/tokio-rs/tokio/pull/5189)

### Added (unstable)

- process: add `Command::process_group` [[#5114]](https://github.com/tokio-rs/tokio/pull/5114)
- runtime: export metrics about the blocking thread pool [[#5161]](https://github.com/tokio-rs/tokio/pull/5161)
- task: add `task::id()` and `task::try_id()` [[#5171]](https://github.com/tokio-rs/tokio/pull/5171)

### Fixed

- macros: don't take ownership of futures in macros [[#5087]](https://github.com/tokio-rs/tokio/pull/5087)
- runtime: fix Stacked Borrows violation in `LocalOwnedTasks` [[#5099]](https://github.com/tokio-rs/tokio/pull/5099)
- runtime: mitigate ABA with 32-bit queue indices when possible [[#5042]](https://github.com/tokio-rs/tokio/pull/5042)
- task: wake local tasks to the local queue when woken by the same thread [[#5095]](https://github.com/tokio-rs/tokio/pull/5095)
- time: panic in release mode when `mark_pending` called illegally [[#5093]](https://github.com/tokio-rs/tokio/pull/5093)
- runtime: fix typo in expect message [[#5169]](https://github.com/tokio-rs/tokio/pull/5169)
- runtime: fix `unsync_load` on atomic types [[#5175]](https://github.com/tokio-rs/tokio/pull/5175)
- task: elaborate safety comments in task deallocation [[#5172]](https://github.com/tokio-rs/tokio/pull/5172)
- runtime: fix `LocalSet` drop in thread local [[#5179]](https://github.com/tokio-rs/tokio/pull/5179)
- net: remove libc type leakage in a public API [[#5191]](https://github.com/tokio-rs/tokio/pull/5191)
- runtime: update the alignment of `CachePadded` [[#5106]](https://github.com/tokio-rs/tokio/pull/5106)

### Changed

- io: make `tokio::io::copy` continue filling the buffer when writer stalls [[#5066]](https://github.com/tokio-rs/tokio/pull/5066)
- runtime: remove `coop::budget` from `LocalSet::run_until` [[#5155]](https://github.com/tokio-rs/tokio/pull/5155)
- sync: make `Notify` panic safe [[#5154]](https://github.com/tokio-rs/tokio/pull/5154)

### Documented

- io: fix doc for `write_i8` to use signed integers [[#5040]](https://github.com/tokio-rs/tokio/pull/5040)
- net: fix doc typos for TCP and UDP `set_tos` methods [[#5073]](https://github.com/tokio-rs/tokio/pull/5073)
- net: fix function name in `UdpSocket::recv` documentation [[#5150]](https://github.com/tokio-rs/tokio/pull/5150)
- sync: typo in `TryLockError` for `RwLock::try_write` [[#5160]](https://github.com/tokio-rs/tokio/pull/5160)
- task: document that spawned tasks execute immediately [[#5117]](https://github.com/tokio-rs/tokio/pull/5117)
- time: document return type of `timeout` [[#5118]](https://github.com/tokio-rs/tokio/pull/5118)
- time: document that `timeout` checks only before poll [[#5126]](https://github.com/tokio-rs/tokio/pull/5126)
- sync: specify return type of `oneshot::Receiver` in docs [[#5198]](https://github.com/tokio-rs/tokio/pull/5198)

## 1.21.2 (September 27, 2022)

This release removes the dependency on the `once_cell` crate to restore the MSRV of 1.21.x, which is the latest minor version at the time of release. [[#5048]](https://github.com/tokio-rs/tokio/pull/5048)

## 1.21.1 (September 13, 2022)

### Fixed

- net: fix dependency resolution for socket2 [[#5000]](https://github.com/tokio-rs/tokio/pull/5000)
- task: ignore failure to set TLS in `LocalSet` Drop [[#4976]](https://github.com/tokio-rs/tokio/pull/4976)

## 1.21.0 (September 2, 2022)

This release is the first release of Tokio to intentionally support WASM. The `sync,macros,io-util,rt,time` features are stabilized on WASM. Additionally the wasm32-wasi target is given unstable support for the `net` feature.

### Added

- net: add `device` and `bind_device` methods to TCP/UDP sockets [[#4882]](https://github.com/tokio-rs/tokio/pull/4882)
- net: add `tos` and `set_tos` methods to TCP and UDP sockets [[#4877]](https://github.com/tokio-rs/tokio/pull/4877)
- net: add security flags to named pipe `ServerOptions` [[#4845]](https://github.com/tokio-rs/tokio/pull/4845)
- signal: add more windows signal handlers [[#4924]](https://github.com/tokio-rs/tokio/pull/4924)
- sync: add `mpsc::Sender::max_capacity` method [[#4904]](https://github.com/tokio-rs/tokio/pull/4904)
- sync: implement Weak version of `mpsc::Sender` [[#4595]](https://github.com/tokio-rs/tokio/pull/4595)
- task: add `LocalSet::enter` [[#4765]](https://github.com/tokio-rs/tokio/pull/4765)
- task: stabilize `JoinSet` and `AbortHandle` [[#4920]](https://github.com/tokio-rs/tokio/pull/4920)
- tokio: add `track_caller` to public APIs [[#4805]](https://github.com/tokio-rs/tokio/pull/4805), [[#4848]](https://github.com/tokio-rs/tokio/pull/4848), [[#4852]](https://github.com/tokio-rs/tokio/pull/4852)
- wasm: initial support for `wasm32-wasi` target [[#4716]](https://github.com/tokio-rs/tokio/pull/4716)

### Fixed

- miri: improve miri compatibility by avoiding temporary references in `linked_list::Link` impls [[#4841]](https://github.com/tokio-rs/tokio/pull/4841)
- signal: don't register write interest on signal pipe [[#4898]](https://github.com/tokio-rs/tokio/pull/4898)
- sync: add `#[must_use]` to lock guards [[#4886]](https://github.com/tokio-rs/tokio/pull/4886)
- sync: fix hang when calling `recv` on closed and reopened broadcast channel [[#4867]](https://github.com/tokio-rs/tokio/pull/4867)
- task: propagate attributes on task-locals [[#4837]](https://github.com/tokio-rs/tokio/pull/4837)

### Changed

- fs: change panic to error in `File::start_seek` [[#4897]](https://github.com/tokio-rs/tokio/pull/4897)
- io: reduce syscalls in `poll_read` [[#4840]](https://github.com/tokio-rs/tokio/pull/4840)
- process: use blocking threadpool for child stdio I/O [[#4824]](https://github.com/tokio-rs/tokio/pull/4824)
- signal: make `SignalKind` methods const [[#4956]](https://github.com/tokio-rs/tokio/pull/4956)

### Documented

- chore: fix typos and grammar [[#4858]](https://github.com/tokio-rs/tokio/pull/4858), [[#4894]](https://github.com/tokio-rs/tokio/pull/4894), [[#4928]](https://github.com/tokio-rs/tokio/pull/4928)
- io: fix typo in `AsyncSeekExt::rewind` docs [[#4893]](https://github.com/tokio-rs/tokio/pull/4893)
- net: add documentation to `try_read()` for zero-length buffers [[#4937]](https://github.com/tokio-rs/tokio/pull/4937)
- runtime: remove incorrect panic section for `Builder::worker_threads` [[#4849]](https://github.com/tokio-rs/tokio/pull/4849)
- sync: doc of `watch::Sender::send` improved [[#4959]](https://github.com/tokio-rs/tokio/pull/4959)
- task: add cancel safety docs to `JoinHandle` [[#4901]](https://github.com/tokio-rs/tokio/pull/4901)
- task: expand on cancellation of `spawn_blocking` [[#4811]](https://github.com/tokio-rs/tokio/pull/4811)
- time: clarify that the first tick of `Interval::tick` happens immediately [[#4951]](https://github.com/tokio-rs/tokio/pull/4951)

### Unstable

- rt: add unstable option to disable the LIFO slot [[#4936]](https://github.com/tokio-rs/tokio/pull/4936)
- task: fix incorrect signature in `Builder::spawn_on` [[#4953]](https://github.com/tokio-rs/tokio/pull/4953)
- task: make `task::Builder::spawn*` methods fallible [[#4823]](https://github.com/tokio-rs/tokio/pull/4823)

## 1.20.6 (September 22, 2023)

This is a backport of a change from 1.27.0.

### Changed

- io: use `memchr` from `libc` [[#5960]](https://github.com/tokio-rs/tokio/pull/5960)

## 1.20.5 (May 28, 2023)

Forward ports 1.18.6 changes.

### Fixed

- deps: disable default features for mio [[#5728]](https://github.com/tokio-rs/tokio/pull/5728)

## 1.20.4 (January 17, 2023)

Forward ports 1.18.5 changes.

### Fixed

- io: fix unsoundness in `ReadHalf::unsplit` [[#5375]](https://github.com/tokio-rs/tokio/pull/5375)

## 1.20.3 (January 3, 2022)

This release forward ports changes from 1.18.4.

### Fixed

- net: fix Windows named pipe server builder to maintain option when toggling pipe mode [[#5336]](https://github.com/tokio-rs/tokio/pull/5336).

## 1.20.2 (September 27, 2022)

This release removes the dependency on the `once_cell` crate to restore the MSRV of the 1.20.x LTS release. [[#5048]](https://github.com/tokio-rs/tokio/pull/5048)

## 1.20.1 (July 25, 2022)

### Fixed

- chore: fix version detection in build script [[#4860]](https://github.com/tokio-rs/tokio/pull/4860)

## 1.20.0 (July 12, 2022)

### Added

- tokio: add `track_caller` to public APIs [[#4772]](https://github.com/tokio-rs/tokio/pull/4772), [[#4791]](https://github.com/tokio-rs/tokio/pull/4791), [[#4793]](https://github.com/tokio-rs/tokio/pull/4793), [[#4806]](https://github.com/tokio-rs/tokio/pull/4806), [[#4808]](https://github.com/tokio-rs/tokio/pull/4808)
- sync: Add `has_changed` method to `watch::Ref` [[#4758]](https://github.com/tokio-rs/tokio/pull/4758)

### Changed

- time: remove `src/time/driver/wheel/stack.rs` [[#4766]](https://github.com/tokio-rs/tokio/pull/4766)
- rt: clean up arguments passed to basic scheduler [[#4767]](https://github.com/tokio-rs/tokio/pull/4767)
- net: be more specific about winapi features [[#4764]](https://github.com/tokio-rs/tokio/pull/4764)
- tokio: use const initialized thread locals where possible [[#4677]](https://github.com/tokio-rs/tokio/pull/4677)
- task: various small improvements to LocalKey [[#4795]](https://github.com/tokio-rs/tokio/pull/4795)

### Documented

- fs: warn about performance pitfall [[#4762]](https://github.com/tokio-rs/tokio/pull/4762)
- chore: fix spelling [[#4769]](https://github.com/tokio-rs/tokio/pull/4769)
- sync: document spurious failures in oneshot [[#4777]](https://github.com/tokio-rs/tokio/pull/4777)
- sync: add warning for watch in non-Send futures [[#4741]](https://github.com/tokio-rs/tokio/pull/4741)
- chore: fix typo [[#4798]](https://github.com/tokio-rs/tokio/pull/4798)

### Unstable

- joinset: rename `join_one` to `join_next` [[#4755]](https://github.com/tokio-rs/tokio/pull/4755)
- rt: unhandled panic config for current thread rt [[#4770]](https://github.com/tokio-rs/tokio/pull/4770)

## 1.19.2 (June 6, 2022)

This release fixes another bug in `Notified::enable`. [[#4751]](https://github.com/tokio-rs/tokio/pull/4751)

## 1.19.1 (June 5, 2022)

This release fixes a bug in `Notified::enable`. [[#4747]](https://github.com/tokio-rs/tokio/pull/4747)

## 1.19.0 (June 3, 2022)

### Added

- runtime: add `is_finished` method for `JoinHandle` and `AbortHandle` [[#4709]](https://github.com/tokio-rs/tokio/pull/4709)
- runtime: make global queue and event polling intervals configurable [[#4671]](https://github.com/tokio-rs/tokio/pull/4671)
- sync: add `Notified::enable` [[#4705]](https://github.com/tokio-rs/tokio/pull/4705)
- sync: add `watch::Sender::send_if_modified` [[#4591]](https://github.com/tokio-rs/tokio/pull/4591)
- sync: add resubscribe method to broadcast::Receiver [[#4607]](https://github.com/tokio-rs/tokio/pull/4607)
- net: add `take_error` to `TcpSocket` and `TcpStream` [[#4739]](https://github.com/tokio-rs/tokio/pull/4739)

### Changed

- io: refactor out usage of Weak in the io handle [[#4656]](https://github.com/tokio-rs/tokio/pull/4656)

### Fixed

- macros: avoid starvation in `join!` and `try_join!` [[#4624]](https://github.com/tokio-rs/tokio/pull/4624)

### Documented

- runtime: clarify semantics of tasks outliving `block_on` [[#4729]](https://github.com/tokio-rs/tokio/pull/4729)
- time: fix example for `MissedTickBehavior::Burst` [[#4713]](https://github.com/tokio-rs/tokio/pull/4713)

### Unstable

- metrics: correctly update atomics in `IoDriverMetrics` [[#4725]](https://github.com/tokio-rs/tokio/pull/4725)
- metrics: fix compilation with unstable, process, and rt, but without net [[#4682]](https://github.com/tokio-rs/tokio/pull/4682)
- task: add `#[track_caller]` to `JoinSet`/`JoinMap` [[#4697]](https://github.com/tokio-rs/tokio/pull/4697)
- task: add `Builder::{spawn_on, spawn_local_on, spawn_blocking_on}` [[#4683]](https://github.com/tokio-rs/tokio/pull/4683)
- task: add `consume_budget` for cooperative scheduling [[#4498]](https://github.com/tokio-rs/tokio/pull/4498)
- task: add `join_set::Builder` for configuring `JoinSet` tasks [[#4687]](https://github.com/tokio-rs/tokio/pull/4687)
- task: update return value of `JoinSet::join_one` [[#4726]](https://github.com/tokio-rs/tokio/pull/4726)

## 1.18.6 (May 28, 2023)

### Fixed

- deps: disable default features for mio [[#5728]](https://github.com/tokio-rs/tokio/pull/5728)

## 1.18.5 (January 17, 2023)

### Fixed

- io: fix unsoundness in `ReadHalf::unsplit` [[#5375]](https://github.com/tokio-rs/tokio/pull/5375)

## 1.18.4 (January 3, 2022)

### Fixed

- net: fix Windows named pipe server builder to maintain option when toggling pipe mode [[#5336]](https://github.com/tokio-rs/tokio/pull/5336).

## 1.18.3 (September 27, 2022)

This release removes the dependency on the `once_cell` crate to restore the MSRV of the 1.18.x LTS release. [[#5048]](https://github.com/tokio-rs/tokio/pull/5048)

## 1.18.2 (May 5, 2022)

Add missing features for the `winapi` dependency. [[#4663]](https://github.com/tokio-rs/tokio/pull/4663)

## 1.18.1 (May 2, 2022)

The 1.18.0 release broke the build for targets without 64-bit atomics when building with `tokio_unstable`. This release fixes that. [[#4649]](https://github.com/tokio-rs/tokio/pull/4649)

## 1.18.0 (April 27, 2022)

This release adds a number of new APIs in `tokio::net`, `tokio::signal`, and `tokio::sync`. In addition, it adds new unstable APIs to `tokio::task` (`Id`s for uniquely identifying a task, and `AbortHandle` for remotely cancelling a task), as well as a number of bugfixes.

### Fixed

- blocking: add missing `#[track_caller]` for `spawn_blocking` [[#4616]](https://github.com/tokio-rs/tokio/pull/4616)
- macros: fix `select` macro to process 64 branches [[#4519]](https://github.com/tokio-rs/tokio/pull/4519)
- net: fix `try_io` methods not calling Mio's `try_io` internally [[#4582]](https://github.com/tokio-rs/tokio/pull/4582)
- runtime: recover when OS fails to spawn a new thread [[#4485]](https://github.com/tokio-rs/tokio/pull/4485)

### Added

- net: add `UdpSocket::peer_addr` [[#4611]](https://github.com/tokio-rs/tokio/pull/4611)
- net: add `try_read_buf` method for named pipes [[#4626]](https://github.com/tokio-rs/tokio/pull/4626)
- signal: add `SignalKind` `Hash`/`Eq` impls and `c_int` conversion [[#4540]](https://github.com/tokio-rs/tokio/pull/4540)
- signal: add support for signals up to `SIGRTMAX` [[#4555]](https://github.com/tokio-rs/tokio/pull/4555)
- sync: add `watch::Sender::send_modify` method [[#4310]](https://github.com/tokio-rs/tokio/pull/4310)
- sync: add `broadcast::Receiver::len` method [[#4542]](https://github.com/tokio-rs/tokio/pull/4542)
- sync: add `watch::Receiver::same_channel` method [[#4581]](https://github.com/tokio-rs/tokio/pull/4581)
- sync: implement `Clone` for `RecvError` types [[#4560]](https://github.com/tokio-rs/tokio/pull/4560)

### Changed

- update `mio` to 0.8.1 [[#4582]](https://github.com/tokio-rs/tokio/pull/4582)
- macros: rename `tokio::select!`'s internal `util` module [[#4543]](https://github.com/tokio-rs/tokio/pull/4543)
- runtime: use `Vec::with_capacity` when building runtime [[#4553]](https://github.com/tokio-rs/tokio/pull/4553)

### Documented

- improve docs for `tokio_unstable` [[#4524]](https://github.com/tokio-rs/tokio/pull/4524)
- runtime: include more documentation for thread_pool/worker [[#4511]](https://github.com/tokio-rs/tokio/pull/4511)
- runtime: update `Handle::current`'s docs to mention `EnterGuard` [[#4567]](https://github.com/tokio-rs/tokio/pull/4567)
- time: clarify platform specific timer resolution [[#4474]](https://github.com/tokio-rs/tokio/pull/4474)
- signal: document that `Signal::recv` is cancel-safe [[#4634]](https://github.com/tokio-rs/tokio/pull/4634)
- sync: `UnboundedReceiver` close docs [[#4548]](https://github.com/tokio-rs/tokio/pull/4548)

### Unstable

The following changes only apply when building with `--cfg tokio_unstable`:

- task: add `task::Id` type [[#4630]](https://github.com/tokio-rs/tokio/pull/4630)
- task: add `AbortHandle` type for cancelling tasks in a `JoinSet` [[#4530]](https://github.com/tokio-rs/tokio/pull/4530), [[#4640]](https://github.com/tokio-rs/tokio/pull/4640)
- task: fix missing `doc(cfg(...))` attributes for `JoinSet` [[#4531]](https://github.com/tokio-rs/tokio/pull/4531)
- task: fix broken link in `AbortHandle` RustDoc [[#4545]](https://github.com/tokio-rs/tokio/pull/4545)
- metrics: add initial IO driver metrics [[#4507]](https://github.com/tokio-rs/tokio/pull/4507)

## 1.17.0 (February 16, 2022)

This release updates the minimum supported Rust version (MSRV) to 1.49, the `mio` dependency to v0.8, and the (optional) `parking_lot` dependency to v0.12. Additionally, it contains several bug fixes, as well as internal refactoring and performance improvements.

### Fixed

- time: prevent panicking in `sleep` with large durations [[#4495]](https://github.com/tokio-rs/tokio/pull/4495)
- time: eliminate potential panics in `Instant` arithmetic on platforms where `Instant::now` is not monotonic [[#4461]](https://github.com/tokio-rs/tokio/pull/4461)
- io: fix `DuplexStream` not participating in cooperative yielding [[#4478]](https://github.com/tokio-rs/tokio/pull/4478)
- rt: fix potential double panic when dropping a `JoinHandle` [[#4430]](https://github.com/tokio-rs/tokio/pull/4430)

### Changed

- update minimum supported Rust version to 1.49 [[#4457]](https://github.com/tokio-rs/tokio/pull/4457)
- update `parking_lot` dependency to v0.12.0 [[#4459]](https://github.com/tokio-rs/tokio/pull/4459)
- update `mio` dependency to v0.8 [[#4449]](https://github.com/tokio-rs/tokio/pull/4449)
- rt: remove an unnecessary lock in the blocking pool [[#4436]](https://github.com/tokio-rs/tokio/pull/4436)
- rt: remove an unnecessary enum in the basic scheduler [[#4462]](https://github.com/tokio-rs/tokio/pull/4462)
- time: use bit manipulation instead of modulo to improve performance [[#4480]](https://github.com/tokio-rs/tokio/pull/4480)
- net: use `std::future::Ready` instead of our own `Ready` future [[#4271]](https://github.com/tokio-rs/tokio/pull/4271)
- replace deprecated `atomic::spin_loop_hint` with `hint::spin_loop` [[#4491]](https://github.com/tokio-rs/tokio/pull/4491)
- fix miri failures in intrusive linked lists [[#4397]](https://github.com/tokio-rs/tokio/pull/4397)

### Documented

- io: add an example for `tokio::process::ChildStdin` [[#4479]](https://github.com/tokio-rs/tokio/pull/4479)

### Unstable

The following changes only apply when building with `--cfg tokio_unstable`:

- task: fix missing location information in `tracing` spans generated by `spawn_local` [[#4483]](https://github.com/tokio-rs/tokio/pull/4483)
- task: add `JoinSet` for managing sets of tasks [[#4335]](https://github.com/tokio-rs/tokio/pull/4335)
- metrics: fix compilation error on MIPS [[#4475]](https://github.com/tokio-rs/tokio/pull/4475)
- metrics: fix compilation error on arm32v7 [[#4453]](https://github.com/tokio-rs/tokio/pull/4453)

## 1.16.1 (January 28, 2022)

This release fixes a bug in [[#4428]](https://github.com/tokio-rs/tokio/pull/4428) with the change [[#4437]](https://github.com/tokio-rs/tokio/pull/4437).

## 1.16.0 (January 27, 2022)

Fixes a soundness bug in `io::Take` ([[#4428]](https://github.com/tokio-rs/tokio/pull/4428)). The unsoundness is exposed when leaking memory in the given `AsyncRead` implementation and then overwriting the supplied buffer:

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

Also, this release includes improvements to the multi-threaded scheduler that can increase throughput by up to 20% in some cases ([[#4383]](https://github.com/tokio-rs/tokio/pull/4383)).

### Fixed

- io: **soundness** don't expose uninitialized memory when using `io::Take` in edge case [[#4428]](https://github.com/tokio-rs/tokio/pull/4428)
- fs: ensure `File::write` results in a `write` syscall when the runtime shuts down [[#4316]](https://github.com/tokio-rs/tokio/pull/4316)
- process: drop pipe after child exits in `wait_with_output` [[#4315]](https://github.com/tokio-rs/tokio/pull/4315)
- rt: improve error message when spawning a thread fails [[#4398]](https://github.com/tokio-rs/tokio/pull/4398)
- rt: reduce false-positive thread wakups in the multi-threaded scheduler [[#4383]](https://github.com/tokio-rs/tokio/pull/4383)
- sync: don't inherit `Send` from `parking_lot::*Guard` [[#4359]](https://github.com/tokio-rs/tokio/pull/4359)

### Added

- net: `TcpSocket::linger()` and `set_linger()` [[#4324]](https://github.com/tokio-rs/tokio/pull/4324)
- net: impl `UnwindSafe` for socket types [[#4384]](https://github.com/tokio-rs/tokio/pull/4384)
- rt: impl `UnwindSafe` for `JoinHandle` [[#4418]](https://github.com/tokio-rs/tokio/pull/4418)
- sync: `watch::Receiver::has_changed()` [[#4342]](https://github.com/tokio-rs/tokio/pull/4342)
- sync: `oneshot::Receiver::blocking_recv()` [[#4334]](https://github.com/tokio-rs/tokio/pull/4334)
- sync: `RwLock` blocking operations [[#4425]](https://github.com/tokio-rs/tokio/pull/4425)

### Unstable

The following changes only apply when building with `--cfg tokio_unstable`

- rt: **breaking change** overhaul runtime metrics API [[#4373]](https://github.com/tokio-rs/tokio/pull/4373)

## 1.15.0 (December 15, 2021)

### Fixed

- io: add cooperative yielding support to `io::empty()` [[#4300]](https://github.com/tokio-rs/tokio/pull/4300)
- time: make timeout robust against budget-depleting tasks [[#4314]](https://github.com/tokio-rs/tokio/pull/4314)

### Changed

- update minimum supported Rust version to 1.46.

### Added

- time: add `Interval::reset()` [[#4248]](https://github.com/tokio-rs/tokio/pull/4248)
- io: add explicit lifetimes to `AsyncFdReadyGuard` [[#4267]](https://github.com/tokio-rs/tokio/pull/4267)
- process: add `Command::as_std()` [[#4295]](https://github.com/tokio-rs/tokio/pull/4295)

### Added (unstable)

- tracing: instrument `tokio::sync` types [[#4302]](https://github.com/tokio-rs/tokio/pull/4302)

## 1.14.0 (November 15, 2021)

### Fixed

- macros: fix compiler errors when using `mut` patterns in `select!` [[#4211]](https://github.com/tokio-rs/tokio/pull/4211)
- sync: fix a data race between `oneshot::Sender::send` and awaiting a `oneshot::Receiver` when the oneshot has been closed [[#4226]](https://github.com/tokio-rs/tokio/pull/4226)
- sync: make `AtomicWaker` panic safe [[#3689]](https://github.com/tokio-rs/tokio/pull/3689)
- runtime: fix basic scheduler dropping tasks outside a runtime context [[#4213]](https://github.com/tokio-rs/tokio/pull/4213)

### Added

- stats: add `RuntimeStats::busy_duration_total` [[#4179]](https://github.com/tokio-rs/tokio/pull/4179), [[#4223]](https://github.com/tokio-rs/tokio/pull/4223)

### Changed

- io: updated `copy` buffer size to match `std::io::copy` [[#4209]](https://github.com/tokio-rs/tokio/pull/4209)

### Documented

- io: rename buffer to file in doc-test [[#4230]](https://github.com/tokio-rs/tokio/pull/4230)
- sync: fix Notify example [[#4212]](https://github.com/tokio-rs/tokio/pull/4212)

## 1.13.1 (November 15, 2021)

### Fixed

- sync: fix a data race between `oneshot::Sender::send` and awaiting a `oneshot::Receiver` when the oneshot has been closed [[#4226]](https://github.com/tokio-rs/tokio/pull/4226)

## 1.13.0 (October 29, 2021)

### Fixed

- sync: fix `Notify` to clone the waker before locking its waiter list [[#4129]](https://github.com/tokio-rs/tokio/pull/4129)
- tokio: add riscv32 to non atomic64 architectures [[#4185]](https://github.com/tokio-rs/tokio/pull/4185)

### Added

- net: add `poll_{recv,send}_ready` methods to `udp` and `uds_datagram` [[#4131]](https://github.com/tokio-rs/tokio/pull/4131)
- net: add `try_*`, `readable`, `writable`, `ready`, and `peer_addr` methods to split halves [[#4120]](https://github.com/tokio-rs/tokio/pull/4120)
- sync: add `blocking_lock` to `Mutex` [[#4130]](https://github.com/tokio-rs/tokio/pull/4130)
- sync: add `watch::Sender::send_replace` [[#3962]](https://github.com/tokio-rs/tokio/pull/3962), [[#4195]](https://github.com/tokio-rs/tokio/pull/4195)
- sync: expand `Debug` for `Mutex<T>` impl to unsized `T` [[#4134]](https://github.com/tokio-rs/tokio/pull/4134)
- tracing: instrument time::Sleep [[#4072]](https://github.com/tokio-rs/tokio/pull/4072)
- tracing: use structured location fields for spawned tasks [[#4128]](https://github.com/tokio-rs/tokio/pull/4128)

### Changed

- io: add assert in `copy_bidirectional` that `poll_write` is sensible [[#4125]](https://github.com/tokio-rs/tokio/pull/4125)
- macros: use qualified syntax when polling in `select!` [[#4192]](https://github.com/tokio-rs/tokio/pull/4192)
- runtime: handle `block_on` wakeups better [[#4157]](https://github.com/tokio-rs/tokio/pull/4157)
- task: allocate callback on heap immediately in debug mode [[#4203]](https://github.com/tokio-rs/tokio/pull/4203)
- tokio: assert platform-minimum requirements at build time [[#3797]](https://github.com/tokio-rs/tokio/pull/3797)

### Documented

- docs: conversion of doc comments to indicative mood [[#4174]](https://github.com/tokio-rs/tokio/pull/4174)
- docs: add returning on the first error example for `try_join!` [[#4133]](https://github.com/tokio-rs/tokio/pull/4133)
- docs: fixing broken links in `tokio/src/lib.rs` [[#4132]](https://github.com/tokio-rs/tokio/pull/4132)
- signal: add example with background listener [[#4171]](https://github.com/tokio-rs/tokio/pull/4171)
- sync: add more oneshot examples [[#4153]](https://github.com/tokio-rs/tokio/pull/4153)
- time: document `Interval::tick` cancel safety [[#4152]](https://github.com/tokio-rs/tokio/pull/4152)

## 1.12.0 (September 21, 2021)

### Fixed

- mpsc: ensure `try_reserve` error is consistent with `try_send` [[#4119]](https://github.com/tokio-rs/tokio/pull/4119)
- mpsc: use `spin_loop_hint` instead of `yield_now` [[#4115]](https://github.com/tokio-rs/tokio/pull/4115)
- sync: make `SendError` field public [[#4097]](https://github.com/tokio-rs/tokio/pull/4097)

### Added

- io: add POSIX AIO on FreeBSD [[#4054]](https://github.com/tokio-rs/tokio/pull/4054)
- io: add convenience method `AsyncSeekExt::rewind` [[#4107]](https://github.com/tokio-rs/tokio/pull/4107)
- runtime: add tracing span for `block_on` futures [[#4094]](https://github.com/tokio-rs/tokio/pull/4094)
- runtime: callback when a worker parks and unparks [[#4070]](https://github.com/tokio-rs/tokio/pull/4070)
- sync: implement `try_recv` for mpsc channels [[#4113]](https://github.com/tokio-rs/tokio/pull/4113)

### Documented

- docs: clarify CPU-bound tasks on Tokio [[#4105]](https://github.com/tokio-rs/tokio/pull/4105)
- mpsc: document spurious failures on `poll_recv` [[#4117]](https://github.com/tokio-rs/tokio/pull/4117)
- mpsc: document that `PollSender` impls `Sink` [[#4110]](https://github.com/tokio-rs/tokio/pull/4110)
- task: document non-guarantees of `yield_now` [[#4091]](https://github.com/tokio-rs/tokio/pull/4091)
- time: document paused time details better [[#4061]](https://github.com/tokio-rs/tokio/pull/4061), [[#4103]](https://github.com/tokio-rs/tokio/pull/4103)

## 1.11.0 (August 31, 2021)

### Fixed

- time: don't panic when Instant is not monotonic [[#4044]](https://github.com/tokio-rs/tokio/pull/4044)
- io: fix panic in `fill_buf` by not calling `poll_fill_buf` twice [[#4084]](https://github.com/tokio-rs/tokio/pull/4084)

### Added

- watch: add `watch::Sender::subscribe` [[#3800]](https://github.com/tokio-rs/tokio/pull/3800)
- process: add `from_std` to `ChildStd*` [[#4045]](https://github.com/tokio-rs/tokio/pull/4045)
- stats: initial work on runtime stats [[#4043]](https://github.com/tokio-rs/tokio/pull/4043)

### Changed

- tracing: change span naming to new console convention [[#4042]](https://github.com/tokio-rs/tokio/pull/4042)
- io: speed-up waking by using uninitialized array [[#4055]](https://github.com/tokio-rs/tokio/pull/4055), [[#4071]](https://github.com/tokio-rs/tokio/pull/4071), [[#4075]](https://github.com/tokio-rs/tokio/pull/4075)

### Documented

- time: make Sleep examples easier to find [[#4040]](https://github.com/tokio-rs/tokio/pull/4040)

## 1.10.1 (August 24, 2021)

### Fixed

- runtime: fix leak in UnownedTask [[#4063]](https://github.com/tokio-rs/tokio/pull/4063)

## 1.10.0 (August 12, 2021)

### Added

- io: add `(read|write)_f(32|64)[_le]` methods [[#4022]](https://github.com/tokio-rs/tokio/pull/4022)
- io: add `fill_buf` and `consume` to `AsyncBufReadExt` [[#3991]](https://github.com/tokio-rs/tokio/pull/3991)
- process: add `Child::raw_handle()` on windows [[#3998]](https://github.com/tokio-rs/tokio/pull/3998)

### Fixed

- doc: fix non-doc builds with `--cfg docsrs` [[#4020]](https://github.com/tokio-rs/tokio/pull/4020)
- io: flush eagerly in `io::copy` [[#4001]](https://github.com/tokio-rs/tokio/pull/4001)
- runtime: a debug assert was sometimes triggered during shutdown [[#4005]](https://github.com/tokio-rs/tokio/pull/4005)
- sync: use `spin_loop_hint` instead of `yield_now` in mpsc [[#4037]](https://github.com/tokio-rs/tokio/pull/4037)
- tokio: the test-util feature depends on rt, sync, and time [[#4036]](https://github.com/tokio-rs/tokio/pull/4036)

### Changes

- runtime: reorganize parts of the runtime [[#3979]](https://github.com/tokio-rs/tokio/pull/3979), [[#4005]](https://github.com/tokio-rs/tokio/pull/4005)
- signal: make windows docs for signal module show up on unix builds [[#3770]](https://github.com/tokio-rs/tokio/pull/3770)
- task: quickly send task to heap on debug mode [[#4009]](https://github.com/tokio-rs/tokio/pull/4009)

### Documented

- io: document cancellation safety of `AsyncBufReadExt` [[#3997]](https://github.com/tokio-rs/tokio/pull/3997)
- sync: document when `watch::send` fails [[#4021]](https://github.com/tokio-rs/tokio/pull/4021)

## 1.9.0 (July 22, 2021)

### Added

- net: allow customized I/O operations for `TcpStream` [[#3888]](https://github.com/tokio-rs/tokio/pull/3888)
- sync: add getter for the mutex from a guard [[#3928]](https://github.com/tokio-rs/tokio/pull/3928)
- task: expose nameable future for `TaskLocal::scope` [[#3273]](https://github.com/tokio-rs/tokio/pull/3273)

### Fixed

- Fix leak if output of future panics on drop [[#3967]](https://github.com/tokio-rs/tokio/pull/3967)
- Fix leak in `LocalSet` [[#3978]](https://github.com/tokio-rs/tokio/pull/3978)

### Changes

- runtime: reorganize parts of the runtime [[#3909]](https://github.com/tokio-rs/tokio/pull/3909), [[#3939]](https://github.com/tokio-rs/tokio/pull/3939), [[#3950]](https://github.com/tokio-rs/tokio/pull/3950), [[#3955]](https://github.com/tokio-rs/tokio/pull/3955), [[#3980]](https://github.com/tokio-rs/tokio/pull/3980)
- sync: clean up `OnceCell` [[#3945]](https://github.com/tokio-rs/tokio/pull/3945)
- task: remove mutex in `JoinError` [[#3959]](https://github.com/tokio-rs/tokio/pull/3959)

## 1.8.3 (July 26, 2021)

This release backports two fixes from 1.9.0

### Fixed

- Fix leak if output of future panics on drop [[#3967]](https://github.com/tokio-rs/tokio/pull/3967)
- Fix leak in `LocalSet` [[#3978]](https://github.com/tokio-rs/tokio/pull/3978)

## 1.8.2 (July 19, 2021)

Fixes a missed edge case from 1.8.1.

### Fixed

- runtime: drop canceled future on next poll [[#3965]](https://github.com/tokio-rs/tokio/pull/3965)

## 1.8.1 (July 6, 2021)

Forward ports 1.5.1 fixes.

### Fixed

- runtime: remotely abort tasks on `JoinHandle::abort` [[#3934]](https://github.com/tokio-rs/tokio/pull/3934)

## 1.8.0 (July 2, 2021)

### Added

- io: add `get_{ref,mut}` methods to `AsyncFdReadyGuard` and `AsyncFdReadyMutGuard` [[#3807]](https://github.com/tokio-rs/tokio/pull/3807)
- io: efficient implementation of vectored writes for `BufWriter` [[#3163]](https://github.com/tokio-rs/tokio/pull/3163)
- net: add ready/try methods to `NamedPipe{Client,Server}` [[#3866]](https://github.com/tokio-rs/tokio/pull/3866), [[#3899]](https://github.com/tokio-rs/tokio/pull/3899)
- sync: add `watch::Receiver::borrow_and_update` [[#3813]](https://github.com/tokio-rs/tokio/pull/3813)
- sync: implement `From<T>` for `OnceCell<T>` [[#3877]](https://github.com/tokio-rs/tokio/pull/3877)
- time: allow users to specify Interval behavior when delayed [[#3721]](https://github.com/tokio-rs/tokio/pull/3721)

### Added (unstable)

- rt: add `tokio::task::Builder` [[#3881]](https://github.com/tokio-rs/tokio/pull/3881)

### Fixed

- net: handle HUP event with `UnixStream` [[#3898]](https://github.com/tokio-rs/tokio/pull/3898)

### Documented

- doc: document cancellation safety [[#3900]](https://github.com/tokio-rs/tokio/pull/3900)
- time: add wait alias to sleep [[#3897]](https://github.com/tokio-rs/tokio/pull/3897)
- time: document auto-advancing behavior of runtime [[#3763]](https://github.com/tokio-rs/tokio/pull/3763)

## 1.7.2 (July 6, 2021)

Forward ports 1.5.1 fixes.

### Fixed

- runtime: remotely abort tasks on `JoinHandle::abort` [[#3934]](https://github.com/tokio-rs/tokio/pull/3934)

## 1.7.1 (June 18, 2021)

### Fixed

- runtime: fix early task shutdown during runtime shutdown [[#3870]](https://github.com/tokio-rs/tokio/pull/3870)

## 1.7.0 (June 15, 2021)

### Added

- net: add named pipes on windows [[#3760]](https://github.com/tokio-rs/tokio/pull/3760)
- net: add `TcpSocket` from `std::net::TcpStream` conversion [[#3838]](https://github.com/tokio-rs/tokio/pull/3838)
- sync: add `receiver_count` to `watch::Sender` [[#3729]](https://github.com/tokio-rs/tokio/pull/3729)
- sync: export `sync::notify::Notified` future publicly [[#3840]](https://github.com/tokio-rs/tokio/pull/3840)
- tracing: instrument task wakers [[#3836]](https://github.com/tokio-rs/tokio/pull/3836)

### Fixed

- macros: suppress `clippy::default_numeric_fallback` lint in generated code [[#3831]](https://github.com/tokio-rs/tokio/pull/3831)
- runtime: immediately drop new tasks when runtime is shut down [[#3752]](https://github.com/tokio-rs/tokio/pull/3752)
- sync: deprecate unused `mpsc::RecvError` type [[#3833]](https://github.com/tokio-rs/tokio/pull/3833)

### Documented

- io: clarify EOF condition for `AsyncReadExt::read_buf` [[#3850]](https://github.com/tokio-rs/tokio/pull/3850)
- io: clarify limits on return values of `AsyncWrite::poll_write` [[#3820]](https://github.com/tokio-rs/tokio/pull/3820)
- sync: add examples to Semaphore [[#3808]](https://github.com/tokio-rs/tokio/pull/3808)

## 1.6.3 (July 6, 2021)

Forward ports 1.5.1 fixes.

### Fixed

- runtime: remotely abort tasks on `JoinHandle::abort` [[#3934]](https://github.com/tokio-rs/tokio/pull/3934)

## 1.6.2 (June 14, 2021)

### Fixes

- test: sub-ms `time:advance` regression introduced in 1.6 [[#3852]](https://github.com/tokio-rs/tokio/pull/3852)

## 1.6.1 (May 28, 2021)

This release reverts [[#3518]](https://github.com/tokio-rs/tokio/issues/3518) because it doesn't work on some kernels due to a kernel bug. [[#3803]](https://github.com/tokio-rs/tokio/issues/3803)

## 1.6.0 (May 14, 2021)

### Added

- fs: try doing a non-blocking read before punting to the threadpool [[#3518]](https://github.com/tokio-rs/tokio/pull/3518)
- io: add `write_all_buf` to `AsyncWriteExt` [[#3737]](https://github.com/tokio-rs/tokio/pull/3737)
- io: implement `AsyncSeek` for `BufReader`, `BufWriter`, and `BufStream` [[#3491]](https://github.com/tokio-rs/tokio/pull/3491)
- net: support non-blocking vectored I/O [[#3761]](https://github.com/tokio-rs/tokio/pull/3761)
- sync: add `mpsc::Sender::{reserve_owned, try_reserve_owned}` [[#3704]](https://github.com/tokio-rs/tokio/pull/3704)
- sync: add a `MutexGuard::map` method that returns a `MappedMutexGuard` [[#2472]](https://github.com/tokio-rs/tokio/pull/2472)
- time: add getter for Interval's period [[#3705]](https://github.com/tokio-rs/tokio/pull/3705)

### Fixed

- io: wake pending writers on `DuplexStream` close [[#3756]](https://github.com/tokio-rs/tokio/pull/3756)
- process: avoid redundant effort to reap orphan processes [[#3743]](https://github.com/tokio-rs/tokio/pull/3743)
- signal: use `std::os::raw::c_int` instead of `libc::c_int` on public API [[#3774]](https://github.com/tokio-rs/tokio/pull/3774)
- sync: preserve permit state in `notify_waiters` [[#3660]](https://github.com/tokio-rs/tokio/pull/3660)
- task: update `JoinHandle` panic message [[#3727]](https://github.com/tokio-rs/tokio/pull/3727)
- time: prevent `time::advance` from going too far [[#3712]](https://github.com/tokio-rs/tokio/pull/3712)

### Documented

- net: hide `net::unix::datagram` module from docs [[#3775]](https://github.com/tokio-rs/tokio/pull/3775)
- process: updated example [[#3748]](https://github.com/tokio-rs/tokio/pull/3748)
- sync: `Barrier` doc should use task, not thread [[#3780]](https://github.com/tokio-rs/tokio/pull/3780)
- task: update documentation on `block_in_place` [[#3753]](https://github.com/tokio-rs/tokio/pull/3753)

## 1.5.1 (July 6, 2021)

### Fixed

- runtime: remotely abort tasks on `JoinHandle::abort` [[#3934]](https://github.com/tokio-rs/tokio/pull/3934)

## 1.5.0 (April 12, 2021)

### Added

- io: add `AsyncSeekExt::stream_position` [[#3650]](https://github.com/tokio-rs/tokio/pull/3650)
- io: add `AsyncWriteExt::write_vectored` [[#3678]](https://github.com/tokio-rs/tokio/pull/3678)
- io: add a `copy_bidirectional` utility [[#3572]](https://github.com/tokio-rs/tokio/pull/3572)
- net: implement `IntoRawFd` for `TcpSocket` [[#3684]](https://github.com/tokio-rs/tokio/pull/3684)
- sync: add `OnceCell` [[#3591]](https://github.com/tokio-rs/tokio/pull/3591)
- sync: add `OwnedRwLockReadGuard` and `OwnedRwLockWriteGuard` [[#3340]](https://github.com/tokio-rs/tokio/pull/3340)
- sync: add `Semaphore::is_closed` [[#3673]](https://github.com/tokio-rs/tokio/pull/3673)
- sync: add `mpsc::Sender::capacity` [[#3690]](https://github.com/tokio-rs/tokio/pull/3690)
- sync: allow configuring `RwLock` max reads [[#3644]](https://github.com/tokio-rs/tokio/pull/3644)
- task: add `sync_scope` for `LocalKey` [[#3612]](https://github.com/tokio-rs/tokio/pull/3612)

### Fixed

- chore: try to avoid `noalias` attributes on intrusive linked list [[#3654]](https://github.com/tokio-rs/tokio/pull/3654)
- rt: fix panic in `JoinHandle::abort()` when called from other threads [[#3672]](https://github.com/tokio-rs/tokio/pull/3672)
- sync: don't panic in `oneshot::try_recv` [[#3674]](https://github.com/tokio-rs/tokio/pull/3674)
- sync: fix notifications getting dropped on receiver drop [[#3652]](https://github.com/tokio-rs/tokio/pull/3652)
- sync: fix `Semaphore` permit overflow calculation [[#3644]](https://github.com/tokio-rs/tokio/pull/3644)

### Documented

- io: clarify requirements of `AsyncFd` [[#3635]](https://github.com/tokio-rs/tokio/pull/3635)
- runtime: fix unclear docs for `{Handle,Runtime}::block_on` [[#3628]](https://github.com/tokio-rs/tokio/pull/3628)
- sync: document that `Semaphore` is fair [[#3693]](https://github.com/tokio-rs/tokio/pull/3693)
- sync: improve doc on blocking mutex [[#3645]](https://github.com/tokio-rs/tokio/pull/3645)

## 1.4.0 (March 20, 2021)

### Added

- macros: introduce biased argument for `select!` [[#3603]](https://github.com/tokio-rs/tokio/pull/3603)
- runtime: add `Handle::block_on` [[#3569]](https://github.com/tokio-rs/tokio/pull/3569)

### Fixed

- runtime: avoid unnecessary polling of `block_on` future [[#3582]](https://github.com/tokio-rs/tokio/pull/3582)
- runtime: fix memory leak/growth when creating many runtimes [[#3564]](https://github.com/tokio-rs/tokio/pull/3564)
- runtime: mark `EnterGuard` with `must_use` [[#3609]](https://github.com/tokio-rs/tokio/pull/3609)

### Documented

- chore: mention fix for building docs in contributing guide [[#3618]](https://github.com/tokio-rs/tokio/pull/3618)
- doc: add link to `PollSender` [[#3613]](https://github.com/tokio-rs/tokio/pull/3613)
- doc: alias sleep to delay [[#3604]](https://github.com/tokio-rs/tokio/pull/3604)
- sync: improve `Mutex` FIFO explanation [[#3615]](https://github.com/tokio-rs/tokio/pull/3615)
- timer: fix double newline in module docs [[#3617]](https://github.com/tokio-rs/tokio/pull/3617)

## 1.3.0 (March 9, 2021)

### Added

- coop: expose an `unconstrained()` opt-out [[#3547]](https://github.com/tokio-rs/tokio/pull/3547)
- net: add `into_std` for net types without it [[#3509]](https://github.com/tokio-rs/tokio/pull/3509)
- sync: add `same_channel` method to `mpsc::Sender` [[#3532]](https://github.com/tokio-rs/tokio/pull/3532)
- sync: add `{try_,}acquire_many_owned` to `Semaphore` [[#3535]](https://github.com/tokio-rs/tokio/pull/3535)
- sync: add back `RwLockWriteGuard::map` and `RwLockWriteGuard::try_map` [[#3348]](https://github.com/tokio-rs/tokio/pull/3348)

### Fixed

- sync: allow `oneshot::Receiver::close` after successful `try_recv` [[#3552]](https://github.com/tokio-rs/tokio/pull/3552)
- time: do not panic on `timeout(Duration::MAX)` [[#3551]](https://github.com/tokio-rs/tokio/pull/3551)

### Documented

- doc: doc aliases for pre-1.0 function names [[#3523]](https://github.com/tokio-rs/tokio/pull/3523)
- io: fix typos [[#3541]](https://github.com/tokio-rs/tokio/pull/3541)
- io: note the EOF behavior of `read_until` [[#3536]](https://github.com/tokio-rs/tokio/pull/3536)
- io: update `AsyncRead::poll_read` doc [[#3557]](https://github.com/tokio-rs/tokio/pull/3557)
- net: update `UdpSocket` splitting doc [[#3517]](https://github.com/tokio-rs/tokio/pull/3517)
- runtime: add link to `LocalSet` on `new_current_thread` [[#3508]](https://github.com/tokio-rs/tokio/pull/3508)
- runtime: update documentation of thread limits [[#3527]](https://github.com/tokio-rs/tokio/pull/3527)
- sync: do not recommend `join_all` for `Barrier` [[#3514]](https://github.com/tokio-rs/tokio/pull/3514)
- sync: documentation for `oneshot` [[#3592]](https://github.com/tokio-rs/tokio/pull/3592)
- sync: rename `notify` to `notify_one` [[#3526]](https://github.com/tokio-rs/tokio/pull/3526)
- time: fix typo in `Sleep` doc [[#3515]](https://github.com/tokio-rs/tokio/pull/3515)
- time: sync `interval.rs` and `time/mod.rs` docs [[#3533]](https://github.com/tokio-rs/tokio/pull/3533)

## 1.2.0 (February 5, 2021)

### Added

- signal: make `Signal::poll_recv` method public [[#3383]](https://github.com/tokio-rs/tokio/pull/3383)

### Fixed

- time: make `test-util` paused time fully deterministic [[#3492]](https://github.com/tokio-rs/tokio/pull/3492)

### Documented

- sync: link to new broadcast and watch wrappers [[#3504]](https://github.com/tokio-rs/tokio/pull/3504)

## 1.1.1 (January 29, 2021)

Forward ports 1.0.3 fix.

### Fixed

- io: memory leak during shutdown [[#3477]](https://github.com/tokio-rs/tokio/pull/3477).

## 1.1.0 (January 22, 2021)

### Added

- net: add `try_read_buf` and `try_recv_buf` [[#3351]](https://github.com/tokio-rs/tokio/pull/3351)
- mpsc: Add `Sender::try_reserve` function [[#3418]](https://github.com/tokio-rs/tokio/pull/3418)
- sync: add `RwLock` `try_read` and `try_write` methods [[#3400]](https://github.com/tokio-rs/tokio/pull/3400)
- io: add `ReadBuf::inner_mut` [[#3443]](https://github.com/tokio-rs/tokio/pull/3443)

### Changed

- macros: improve `select!` error message [[#3352]](https://github.com/tokio-rs/tokio/pull/3352)
- io: keep track of initialized bytes in `read_to_end` [[#3426]](https://github.com/tokio-rs/tokio/pull/3426)
- runtime: consolidate errors for context missing [[#3441]](https://github.com/tokio-rs/tokio/pull/3441)

### Fixed

- task: wake `LocalSet` on `spawn_local` [[#3369]](https://github.com/tokio-rs/tokio/pull/3369)
- sync: fix panic in broadcast::Receiver drop [[#3434]](https://github.com/tokio-rs/tokio/pull/3434)

### Documented

- stream: link to new `Stream` wrappers in `tokio-stream` [[#3343]](https://github.com/tokio-rs/tokio/pull/3343)
- docs: mention that `test-util` feature is not enabled with full [[#3397]](https://github.com/tokio-rs/tokio/pull/3397)
- process: add documentation to process::Child fields [[#3437]](https://github.com/tokio-rs/tokio/pull/3437)
- io: clarify `AsyncFd` docs about changes of the inner fd [[#3430]](https://github.com/tokio-rs/tokio/pull/3430)
- net: update datagram docs on splitting [[#3448]](https://github.com/tokio-rs/tokio/pull/3448)
- time: document that `Sleep` is not `Unpin` [[#3457]](https://github.com/tokio-rs/tokio/pull/3457)
- sync: add link to `PollSemaphore` [[#3456]](https://github.com/tokio-rs/tokio/pull/3456)
- task: add `LocalSet` example [[#3438]](https://github.com/tokio-rs/tokio/pull/3438)
- sync: improve bounded `mpsc` documentation [[#3458]](https://github.com/tokio-rs/tokio/pull/3458)

## 1.0.3 (January 28, 2021)

### Fixed

- io: memory leak during shutdown [[#3477]](https://github.com/tokio-rs/tokio/pull/3477).

## 1.0.2 (January 14, 2021)

### Fixed

- io: soundness in `read_to_end` [[#3428]](https://github.com/tokio-rs/tokio/pull/3428).

## 1.0.1 (December 25, 2020)

This release fixes a soundness hole caused by the combination of `RwLockWriteGuard::map` and `RwLockWriteGuard::downgrade` by removing the `map` function. This is a breaking change, but breaking changes are allowed under our semver policy when they are required to fix a soundness hole. (See [this RFC](https://github.com/rust-lang/rfcs/blob/master/text/1122-language-semver.md#soundness-changes) for more.)

Note that we have chosen not to do a deprecation cycle or similar because Tokio 1.0.0 was released two days ago, and therefore the impact should be minimal.

Due to the soundness hole, we have also yanked Tokio version 1.0.0.

### Removed

- sync: remove `RwLockWriteGuard::map` and `RwLockWriteGuard::try_map` [[#3345]](https://github.com/tokio-rs/tokio/pull/3345)

### Fixed

- docs: remove stream feature from docs [[#3335]](https://github.com/tokio-rs/tokio/pull/3335)

## 1.0.0 (December 23, 2020)

Commit to the API and long-term support.

### Fixed

- sync: spurious wakeup in `watch` [[#3234]](https://github.com/tokio-rs/tokio/pull/3234).

### Changed

- io: rename `AsyncFd::with_io()` to `try_io()` [[#3306]](https://github.com/tokio-rs/tokio/pull/3306)
- fs: avoid OS specific `*Ext` traits in favor of conditionally defining the fn [[#3264]](https://github.com/tokio-rs/tokio/pull/3264).
- fs: `Sleep` is `!Unpin` [[#3278]](https://github.com/tokio-rs/tokio/pull/3278).
- net: pass `SocketAddr` by value [[#3125]](https://github.com/tokio-rs/tokio/pull/3125).
- net: `TcpStream::poll_peek` takes `ReadBuf` [[#3259]](https://github.com/tokio-rs/tokio/pull/3259).
- rt: rename `runtime::Builder::max_threads()` to `max_blocking_threads()` [[#3287]](https://github.com/tokio-rs/tokio/pull/3287).
- time: require `current_thread` runtime when calling `time::pause()` [[#3289]](https://github.com/tokio-rs/tokio/pull/3289).

### Removed

- remove `tokio::prelude` [[#3299]](https://github.com/tokio-rs/tokio/pull/3299).
- io: remove `AsyncFd::with_poll()` [[#3306]](https://github.com/tokio-rs/tokio/pull/3306).
- net: remove `{Tcp,Unix}Stream::shutdown()` in favor of `AsyncWrite::shutdown()` [[#3298]](https://github.com/tokio-rs/tokio/pull/3298).
- stream: move all stream utilities to `tokio-stream` until `Stream` is added to `std` [[#3277]](https://github.com/tokio-rs/tokio/pull/3277).
- sync: mpsc `try_recv()` due to unexpected behavior [[#3263]](https://github.com/tokio-rs/tokio/pull/3263).
- tracing: make unstable as `tracing-core` is not 1.0 yet [[#3266]](https://github.com/tokio-rs/tokio/pull/3266).

### Added

- fs: `poll_*` fns to `DirEntry` [[#3308]](https://github.com/tokio-rs/tokio/pull/3308).
- io: `poll_*` fns to `io::Lines`, `io::Split` [[#3308]](https://github.com/tokio-rs/tokio/pull/3308).
- io: `_mut` method variants to `AsyncFd` [[#3304]](https://github.com/tokio-rs/tokio/pull/3304).
- net: `poll_*` fns to `UnixDatagram` [[#3223]](https://github.com/tokio-rs/tokio/pull/3223).
- net: `UnixStream` readiness and non-blocking ops [[#3246]](https://github.com/tokio-rs/tokio/pull/3246).
- sync: `UnboundedReceiver::blocking_recv()` [[#3262]](https://github.com/tokio-rs/tokio/pull/3262).
- sync: `watch::Sender::borrow()` [[#3269]](https://github.com/tokio-rs/tokio/pull/3269).
- sync: `Semaphore::close()` [[#3065]](https://github.com/tokio-rs/tokio/pull/3065).
- sync: `poll_recv` fns to `mpsc::Receiver`, `mpsc::UnboundedReceiver` [[#3308]](https://github.com/tokio-rs/tokio/pull/3308).
- time: `poll_tick` fn to `time::Interval` [[#3316]](https://github.com/tokio-rs/tokio/pull/3316).

