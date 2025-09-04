# 时间

用于跟踪时间的实用工具。该模块提供了多种类型，用于在设定的时间段后执行代码。这些类型必须在 Tokio [`Runtime`](./api-runtime.md) 的上下文中使​​用。

<x-cards data-columns="3">
  <x-card data-title="Sleep" data-icon="lucide:timer">
    一个不执行任何操作并在特定 `Instant` 完成的 future。
  </x-card>
  <x-card data-title="Interval" data-icon="lucide:repeat">
    一个以固定周期产生值的流。它使用 `Duration` 进行初始化，并在每次持续时间结束后重复产生值。
  </x-card>
  <x-card data-title="Timeout" data-icon="lucide:alarm-clock-off">
    包装一个 future 或流，为其允许执行的时间设置上限。
  </x-card>
</x-cards>

## 函数

### sleep

`pub fn sleep(duration: Duration) -> Sleep`

等待直到 `duration` 过去。这是 `std::thread::sleep` 的异步模拟。

等待 sleep future 完成时不会执行任何工作。`Sleep` 以毫秒级粒度运行，不应用于需要高分辨率计时器的任务。

要按计划定期运行某些内容，请参阅 [`interval`](#interval)。

#### 取消

通过丢弃返回的 future 来取消 sleep 实例。无需额外的清理工作。

#### Panics

如果没有设置当前计时器，此函数会发生 panic，如果运行时构建器的 `enable_time` 或 `enable_all` 功能未启用，则会触发此情况。如果在 Tokio 运行时之外创建计时器，也可能发生 panic。

#### 示例

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    sleep(Duration::from_millis(100)).await;
    println!("100 ms have elapsed");
}
```

### sleep_until

`pub fn sleep_until(deadline: Instant) -> Sleep`

等待直到达到 `deadline`。

等待 sleep future 完成时不会执行任何工作。`Sleep` 以毫秒级粒度运行，不应用于需要高分辨率计时器的任务。

#### 示例

```rust
use tokio::time::{sleep_until, Instant, Duration};

#[tokio::main]
async fn main() {
    sleep_until(Instant::now() + Duration::from_millis(100)).await;
    println!("100 ms have elapsed");
}
```

### interval

`pub fn interval(period: Duration) -> Interval`

创建一个新的 `Interval`，它以 `period` 的间隔产生 tick。第一个 tick 会立即完成。默认的 missed tick 行为是 `Burst`。

一个 interval 会无限期地 tick。丢弃 `Interval` 值会取消它。

#### Panics

如果 `period` 为零，此函数会发生 panic。

#### 示例

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

### interval_at

`pub fn interval_at(start: Instant, period: Duration) -> Interval`

创建一个新的 `Interval`，它以 `period` 的间隔产生，第一个 tick 在 `start` 时完成。

#### Panics

如果 `period` 为零，此函数会发生 panic。

#### 示例

```rust
use tokio::time::{interval_at, Duration, Instant};

#[tokio::main]
async fn main() {
    let start = Instant::now() + Duration::from_millis(50);
    let mut interval = interval_at(start, Duration::from_millis(10));

    interval.tick().await; // 50ms 后 tick
    interval.tick().await; // 10ms 后 tick
    interval.tick().await; // 10ms 后 tick

    // 大约 70ms 已经过去。
}
```

### timeout

`pub fn timeout<F>(duration: Duration, future: F) -> Timeout<F::IntoFuture>`

要求一个 `Future` 在指定的持续时间过去之前完成。如果 future 及时完成，其值将在 `Ok` 中返回。否则，将返回一个包含 `Elapsed` 错误的 `Err`，并且 future 将被取消。

#### 取消

通过丢弃 `Timeout` future 来取消超时。可以通过调用 `Timeout::into_inner` 来检索原始的 future。

#### 示例

```rust
use tokio::time::timeout;
use tokio::sync::oneshot;
use std::time::Duration;

#[tokio::main]
async fn main() {
    let (_tx, rx) = oneshot::channel::<()>();

    if let Err(_) = timeout(Duration::from_millis(10), rx).await {
        println!("did not receive value within 10 ms");
    }
}
```

### timeout_at

`pub fn timeout_at<F>(deadline: Instant, future: F) -> Timeout<F::IntoFuture>`

要求一个 `Future` 在指定的 `Instant` 时间点之前完成。

#### 示例

```rust
use tokio::time::{Instant, timeout_at};
use tokio::sync::oneshot;
use std::time::Duration;

#[tokio::main]
async fn main() {
    let (_tx, rx) = oneshot::channel::<()>();

    let deadline = Instant::now() + Duration::from_millis(10);
    if let Err(_) = timeout_at(deadline, rx).await {
        println!("did not receive value within 10 ms");
    }
}
```

## 结构体

### Sleep

由 `sleep` 和 `sleep_until` 返回的 future，在指定的时刻完成。

此类型未实现 `Unpin`。如果与 `select!` 一起使用或通过调用 `poll` 来使用它，必须先将其 pin。

#### 方法

| Method | Description |
|---|---|
| `deadline(&self) -> Instant` | 返回 future 将完成的时刻。 |
| `is_elapsed(&self) -> bool` | 如果 `Sleep` 已经过去，则返回 `true`。 |
| `reset(self: Pin<&mut Self>, deadline: Instant)` | 将 `Sleep` 实例重置为新的截止时间。 |

#### 示例：在循环中重置 `Sleep`

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

### Interval

一种允许您等待一系列时刻的类型，这些时刻之间有特定的持续时间。

#### 方法

| Method | Description |
|---|---|
| `tick(&mut self) -> impl Future<Output = Instant>` | 当 interval 中的下一个时刻到达时完成。 |
| `poll_tick(&mut self, cx: &mut Context<'_'>) -> Poll<Instant>` | 轮询以检查 interval 中的下一个时刻是否已到达。 |
| `reset(&mut self)` | 重置 interval，使其在当前时间之后的一个周期完成。 |
| `reset_at(&mut self, deadline: Instant)` | 设置下一个 tick 在给定的时刻到期。 |
| `period(&self) -> Duration` | 返回 interval 的周期。 |
| `missed_tick_behavior(&self) -> MissedTickBehavior` | 返回当前的 `MissedTickBehavior` 策略。 |
| `set_missed_tick_behavior(&mut self, behavior: MissedTickBehavior)` | 设置 `MissedTickBehavior` 策略。 |


### Timeout<T>

一个 future，如果另一个 future 完成时间过长，它会取消该 future。由 `timeout` 和 `timeout_at` 返回。

#### 方法

| Method | Description |
|---|---|
| `get_ref(&self) -> &T` | 获取对底层 future 的引用。 |
| `get_mut(&mut self) -> &mut T` | 获取对底层 future 的可变引用。 |
| `into_inner(self) -> T` | 消费此超时，返回底层的 future。 |

### Instant

一个单调非递减时钟的测量值。这是对 `std::time::Instant` 的 Tokio 感知包装器。

它是不透明的，仅与 `Duration` 一起使用才有意义。它允许测量两个时刻之间的持续时间或比较它们。

#### 方法

| Method | Description |
|---|---|
| `now() -> Instant` | 返回与“现在”对应的时刻。 |
| `elapsed(&self) -> Duration` | 返回自创建此时刻以来经过的时间量。 |
| `duration_since(&self, earlier: Instant) -> Duration` | 返回从另一个时刻到此时刻所经过的时间量。 |
| `checked_add(&self, duration: Duration) -> Option<Instant>` | 将一个持续时间添加到时刻，溢出时返回 `None`。 |
| `checked_sub(&self, duration: Duration) -> Option<Instant>` | 从时刻中减去一个持续时间，如果结果早于最早可表示的时间，则返回 `None`。 |

## 枚举

### MissedTickBehavior

定义 `Interval` 在错过一个 tick 时的行为。如果在没有调用 `Interval::tick()` 的情况下花费了太多时间，则会错过一个 tick。

#### Burst

这是默认行为。尽可能快地 tick 直到追上。产生的 `Instant` 与没有错过任何 tick 的情况相同。

```text
Expected ticks: |     1     |     2     |     3     |     4     |     5     |     6     |
Actual ticks:   | work -----|          delay          | work | work | work -| work -----|
```

#### Delay

从上次调用 `tick` 的时间开始，以 `period` 的倍数进行 tick，而不是从原始开始时间。这有效地在长时间延迟后重置了 interval 的调度。

```text
Expected ticks: |     1     |     2     |     3     |     4     |     5     |     6     |
Actual ticks:   | work -----|          delay          | work -----| work -----| work -----|
```

#### Skip

跳过任何错过的 tick，并在从原始开始时间的下一个 `period` 倍数上调度下一个 tick。

```text
Expected ticks: |     1     |     2     |     3     |     4     |     5     |     6     |
Actual ticks:   | work -----|          delay          | work ---| work -----| work -----|
```

#### 更改行为的示例

```rust
use tokio::time::{interval, Duration, MissedTickBehavior};

# async fn task_that_takes_more_than_50_millis() { tokio::time::sleep(Duration::from_millis(60)).await; }

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let mut interval = interval(Duration::from_millis(50));
    interval.set_missed_tick_behavior(MissedTickBehavior::Delay);

    // 第一个 tick 是立即的
    interval.tick().await;

    task_that_takes_more_than_50_millis().await;
    // Interval 错过了一个 tick

    // 这会立即解析，因为截止时间已过。
    interval.tick().await;

    // 使用 Delay 行为，下一个 tick 被安排在 *上一个*
    // tick 完成后的 50 毫秒，而不是从其最初的预定时间开始。
    interval.tick().await;
}
```