# 时间

用于追踪时间的实用工具。该模块提供了多种类型，用于在设定的时间段后执行代码。这些类型必须在 Tokio [`Runtime`](./api-runtime.md) 的上下文中使用。

```d2
direction: down

subsystem: {
  shape: package
  label: "Future / Stream"
}

sleep: {
  shape: step
  label: "Sleep\n（在一段时间后完成的 Future）"
}

interval: {
  shape: queue
  label: "Interval\n（以固定周期产生的 Stream）"
}

timeout: {
  shape: hexagon
  label: "Timeout\n（为另一个 Future 包装上时间限制）"
}

subsystem -> timeout: "包装"
sleep -> interval: "内部使用"
```

关键组件包括：

*   **[`Sleep`](#sleep):** 一个在特定 `Instant` 完成的 future。
*   **[`Interval`](#interval):** 一个以固定周期产生值的 stream。
*   **[`Timeout`](#timeout):** 一个包装器，如果 future 执行时间过长，则会取消它。
*   **[`Instant`](#instant):** 一个单调非递减时钟的测量值。

---

## Instant

一个单调非递减时钟的测量值，可用于测量基准或计时操作。该类型是标准库 `std::time::Instant` 的包装器，但与 Tokio 的测试实用工具（如 `time::pause()` 和 `time::advance()`）集成。

### `now()`

返回一个与“现在”对应的 instant。

```rust
use tokio::time::{Duration, Instant, sleep};

#[tokio::main]
async fn main() {
    let now = Instant::now();
    sleep(Duration::from_secs(1)).await;
    let elapsed = now.elapsed();
    assert!(elapsed >= Duration::from_secs(1));
    println!("Elapsed: {:?}", elapsed);
}
```

### 其他方法

| 方法 | 描述 |
|---|---|
| `from_std(std)` | 从一个 `std::time::Instant` 创建一个 `tokio::time::Instant`。 |
| `into_std(self)` | 将值转换为一个 `std::time::Instant`。 |
| `duration_since(earlier)` | 返回从另一个 instant 到当前 instant 所经过的时间。 |
| `checked_duration_since(earlier)` | 返回表示已过时间的 `Option<Duration>`，如果 `earlier` 更晚则返回 `None`。 |
| `elapsed()` | 返回自创建此 instant 以来所经过的时间。 |
| `checked_add(duration)` | 将一个 `Duration` 添加到 `Instant`，如果发生溢出则返回 `None`。 |
| `checked_sub(duration)` | 从 `Instant` 中减去一个 `Duration`，如果结果会早于时钟的纪元时间则返回 `None`。 |


---

## Sleep

一个不执行任何工作并在特定 `Instant` 完成的 future。它以毫秒粒度运行。

### `sleep(duration: Duration) -> Sleep`

等待直到 `duration` 经过。这是 `std::thread::sleep` 的异步模拟。

**取消：** 丢弃 `Sleep` future 足以取消等待。

**Panic：** 如果在 Tokio 运行时上下文之外（即没有活动的计时器驱动程序）调用此函数，则会发生 panic。

```rust
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    sleep(Duration::from_millis(100)).await;
    println!("100 ms have elapsed");
}
```

### `sleep_until(deadline: Instant) -> Sleep`

等待直到达到指定的 `deadline`。

```rust
use tokio::time::{sleep_until, Instant, Duration};

#[tokio::main]
async fn main() {
    let deadline = Instant::now() + Duration::from_millis(100);
    sleep_until(deadline).await;
    println!("100 ms have elapsed");
}
```

### Sleep 结构体

`Sleep` future 可以在创建后被操作，例如，重置其截止时间。

#### `reset(self: Pin<&mut Self>, deadline: Instant)`

将 `Sleep` 实例重置为新的截止时间。这对于在循环中重用计时器而无需创建新状态非常有用。

```rust
use tokio::time::{self, Duration, Instant};

#[tokio::main]
async fn main() {
    let sleep = time::sleep(Duration::from_millis(10));
    tokio::pin!(sleep);

    loop {
        tokio::select! {
            () = &mut sleep => {
                println!("timer elapsed");
                sleep.as_mut().reset(Instant::now() + Duration::from_millis(50));
            },
        }
    }
}
```

---

## Interval

一个以固定周期产生值的 stream。与循环中的 `sleep` 不同，`Interval` 会计算 `tick()` 调用之间花费的时间，从而防止时间漂移。

### `interval(period: Duration) -> Interval`

创建一个新的 `Interval`，它会立即开始计时，并每隔 `period` 时间触发一次。

**Panic：** 如果 `period` 为零，则会发生 panic。

```rust
use tokio::time;

async fn task_that_takes_a_second() {
    println!("hello");
    time::sleep(time::Duration::from_secs(1)).await
}

#[tokio::main]
async fn main() {
    let mut interval = time::interval(time::Duration::from_secs(2));
    for _i in 0..5 {
        interval.tick().await;
        task_that_takes_a_second().await;
    }
}
```

### `interval_at(start: Instant, period: Duration) -> Interval`

创建一个新的 `Interval`，它在指定的 `start` 时间开始计时。

```rust
use tokio::time::{interval_at, Duration, Instant};

#[tokio::main]
async fn main() {
    let start = Instant::now() + Duration::from_millis(50);
    let mut interval = interval_at(start, Duration::from_millis(10));

    interval.tick().await; // 50ms 后触发
    interval.tick().await; // 10ms 后触发
    // 从开始到现在大约过去了 60ms。
}
```

### `tick(&mut self) -> impl Future<Output = Instant>`

当到达 interval 中的下一个 instant 时完成。

### 错过 Tick 的行为

如果 `Interval` 的消费者调用 `tick()` 的时间超过了指定的周期，那么一个 tick 就被认为是“错过了”。`MissedTickBehavior` 枚举配置了 interval 如何追赶。

| 行为 | 描述 |
|---|---|
| `Burst` **(默认)** | 尽可能快地触发 tick，直到时间上追赶到它应该在的位置。产生的 instant 与没有错过任何 tick 的情况相同。 |
| `Delay` | 从调用 `tick()` 的那一刻起重置 interval 的周期，从而有效地延迟所有未来的 tick。Tick 的时间不会被缩短。 |
| `Skip` | 跳过任何错过的 tick，并在从开始时间算起的原始周期的下一个倍数处安排下一个 tick。这可能会缩短下一个 tick 的持续时间。 |

你可以使用 `set_missed_tick_behavior()` 来更改此行为。

---

## Timeout

一个为 future 或 stream 设置执行时间上限的包装器。如果 future 未在指定时间内完成，它将被取消，并返回一个 `Elapsed` 错误。

### `timeout(duration: Duration, future: F) -> Timeout<F>`

要求 future 在 `duration` 经过之前完成。

**Panic：** 如果在 Tokio 运行时上下文之外调用此函数，则会发生 panic。

```rust
use tokio::time::{timeout, Duration};

async fn long_future() {
    // 一些长时间运行的工作
    sleep(Duration::from_secs(5)).await;
}

#[tokio::main]
async fn main() {
    let res = timeout(Duration::from_secs(1), long_future()).await;

    if res.is_err() {
        println!("operation timed out");
    }
}
```

### `timeout_at(deadline: Instant, future: F) -> Timeout<F>`

要求 future 在特定的 `deadline` 之前完成。

```rust
use tokio::time::{timeout_at, Instant, Duration};
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (_tx, rx) = oneshot::channel::<()>();

    let deadline = Instant::now() + Duration::from_millis(10);

    if let Err(_) = timeout_at(deadline, rx).await {
        println!("did not receive value within 10 ms");
    }
}
```

### Timeout 结构体

通过消费 `Timeout` 包装器可以检索原始的 future。

#### `into_inner(self) -> T`

消费 `Timeout`，返回其底层的 future。如果你想在 future 被包装后继续使用它，这会很有用。

---

### 重导出

为方便起见，`std::time::Duration` 被重导出为 `tokio::time::Duration`。
