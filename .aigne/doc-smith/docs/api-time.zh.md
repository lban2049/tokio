# Time

用于跟踪时间的实用工具。该模块提供了多种类型，用于在设定的时间段后执行代码。这些类型必须在 Tokio [`Runtime`](./api-runtime.md) 的上下文中使用。

主要的时间相关实用工具如下：

```d2
direction: down

"时间实用工具": {
    shape: package
    grid-columns: 3

    "Sleep": {
        shape: rectangle
        label: "Sleep\n(等待指定的持续时间)"
    }

    "Interval": {
        shape: rectangle
        label: "Interval\n(周期性地执行代码)"
    }

    "Timeout": {
        shape: rectangle
        label: "Timeout\n(限制 future 的执行时间)"
    }
}

"你的异步任务" -> "Sleep": "用于暂停执行"
"你的异步任务" -> "Interval": "用于重复任务"
"你的异步任务" -> "Timeout": "被其包装以强制执行截止时间"

```

## 函数

### sleep()

等待直到 `duration` 过去。这是 `std::thread::sleep` 的异步模拟。

`pub fn sleep(duration: Duration) -> Sleep`

在等待 sleep future 时不执行任何工作。`Sleep` 以毫秒级粒度运行，在某些平台（如 Windows）上可能具有更大的分辨率。

#### 取消

通过丢弃返回的 future 来取消 sleep 实例。无需额外的清理工作。

#### Panics

如果在 Tokio 运行时中没有配置计时器，此函数会 panic。如果运行时构建时未使用 `Builder::enable_time` 或 `Builder::enable_all`，或者在 Tokio 运行时上下文之外调用 `sleep`，则会发生这种情况。

#### 示例

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    sleep(Duration::from_millis(100)).await;
    println!("100 ms have elapsed");
}
```

### sleep_until()

等待直到 `deadline` 到达。

`pub fn sleep_until(deadline: Instant) -> Sleep`

在等待 sleep future 完成时不执行任何工作。与 `sleep` 一样，它以毫秒级粒度运行。

#### Panics

与 `sleep()` 类似，如果当前没有设置计时器，此函数会 panic。

#### 示例

```rust
use tokio::time::{sleep_until, Instant, Duration};

#[tokio::main]
async fn main() {
    sleep_until(Instant::now() + Duration::from_millis(100)).await;
    println!("100 ms have elapsed");
}
```

### interval()

创建一个新的 `Interval`，它以指定的 `period` 产生滴答。第一个滴答会立即完成。

`pub fn interval(period: Duration) -> Interval`

在循环中使用 `interval` 和 `sleep` 的主要区别在于，`Interval` 会考虑两次滴答之间花费的时间。如果一个任务需要一些时间，`interval` 会缩短下一个等待周期以维持整体频率，而使用 `sleep` 的循环则会产生漂移。

#### Panics

如果 `period` 为零，此函数会 panic。

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
此代码大约每两秒执行一次任务。如果使用 `sleep` 而不是 `interval.tick()`，它将每三秒执行一次。

### interval_at()

创建一个新的 `Interval`，它以 `period` 的间隔产生滴答，第一个滴答在指定的 `start` 时间完成。

`pub fn interval_at(start: Instant, period: Duration) -> Interval`

#### Panics

如果 `period` 为零，此函数会 panic。

#### 示例

```rust
use tokio::time::{interval_at, Duration, Instant};

#[tokio::main]
async fn main() {
    let start = Instant::now() + Duration::from_millis(50);
    let mut interval = interval_at(start, Duration::from_millis(10));

    interval.tick().await; // 50ms 后滴答
    interval.tick().await; // 10ms 后滴答
    interval.tick().await; // 10ms 后滴答

    // 大约 70ms 过去了。
}
```

### timeout()

要求 future 在指定的 `duration` 过去之前完成。

`pub fn timeout<F>(duration: Duration, future: F) -> Timeout<F::IntoFuture>`

如果 future 及时完成，其值将作为 `Ok(value)` 返回。否则，将返回 `Err(Elapsed)`，并且 future 会被取消。超时检查在轮询 future *之前* 进行，因此一个不让出（yield）的 future 可能会超过截止时间而不会返回错误。

#### Panics

如果当前没有设置计时器，此函数会 panic。

#### 示例

```rust
use tokio::time::{timeout, Duration};
use tokio::sync::oneshot;

async fn long_future() {
    // 模拟工作
    tokio::time::sleep(Duration::from_secs(5)).await;
}

#[tokio::main]
async fn main() {
    if let Err(_) = timeout(Duration::from_secs(1), long_future()).await {
        println!("operation timed out");
    }
}
```

### timeout_at()

要求 future 在指定的 `deadline` 之前完成。

`pub fn timeout_at<F>(deadline: Instant, future: F) -> Timeout<F::IntoFuture>`

此函数的功能与 `timeout` 类似，但它接受一个特定的 `Instant` 作为截止时间。

#### 示例

```rust
use tokio::time::{Instant, timeout_at, Duration};
use tokio::sync::oneshot;

# async fn dox() {
let (tx, rx) = oneshot::channel();
# tx.send(()).unwrap();

// 将 future 包装在 `Timeout` 中，设置为在未来的 10 毫秒后到期。
if let Err(_) = timeout_at(Instant::now() + Duration::from_millis(10), rx).await {
    println!("did not receive value within 10 ms");
}
# }
```

## 结构体

### Sleep

`sleep` 和 `sleep_until` 返回的 future，在指定的瞬间完成。

此类型未实现 `Unpin`。如果与 `select!` 一起使用或手动轮询它，必须先将其固定，例如使用 `tokio::pin!`。

#### 方法

| Method | Description |
|---|---|
| `deadline()` | 返回 future 将完成的 `Instant`。 |
| `is_elapsed()` | 如果 `Sleep` 实例已过去，则返回 `true`。 |
| `reset(deadline: Instant)` | 将 `Sleep` 实例重置为新的截止时间。 |

#### 示例：重置 Sleep

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

一个以固定间隔产生的流。

此类型允许你等待一系列的瞬间。与在循环中调用 `sleep` 不同，它会补偿对 `tick()` 的调用之间所花费的时间。

#### 方法

| Method | Description |
|---|---|
| `tick()` | 异步等待下一个滴答，返回其被调度到的 `Instant`。 |
| `poll_tick(&mut self, cx: &mut Context<'_'>)` | 轮询以检查是否已到达下一个滴答。 |
| `reset()` | 重置间隔，使其在当前时间之后一个周期完成。 |
| `reset_at(deadline: Instant)` | 设置下一个滴答在给定的瞬间到期。 |
| `period()` | 返回间隔的周期。 |
| `missed_tick_behavior()` | 返回当前的 `MissedTickBehavior` 策略。 |
| `set_missed_tick_behavior(behavior)` | 设置 `MissedTickBehavior` 策略。 |

### Timeout<T>

`timeout` 和 `timeout_at` 返回的 future。

此 future 包装另一个 future，限制其执行时间。

#### 方法

| Method | Description |
|---|---|
| `get_ref()` | 获取对底层 future 的引用。 |
| `get_mut()` | 获取对底层 future 的可变引用。 |
| `into_inner()` | 消费 timeout，返回底层的 future。 |

### Instant

一个单调非递减时钟的度量。

此类型是 `std::time::Instant` 的包装器，用于与 Tokio 的内部时钟对齐，这对于使用 `time::pause()` 和 `time::advance()` 等功能进行测试特别有用。

#### 方法

| Method | Description |
|---|---|
| `now()` | 返回与“现在”对应的 `Instant`。 |
| `duration_since(earlier: Instant)` | 返回 `earlier` 与此瞬间之间经过的 `Duration`。 |
| `elapsed()` | 返回自创建此瞬间以来经过的 `Duration`。 |
| `checked_add(duration: Duration)` | 将 `Duration` 添加到 `Instant`，溢出时返回 `None`。 |
| `checked_sub(duration: Duration)` | 从 `Instant` 中减去 `Duration`，溢出时返回 `None`。 |

## 枚举

### MissedTickBehavior

定义当 `Interval` 错过一个滴答时的行为，例如，当滴答之间的任务花费的时间超过了间隔周期。

#### Burst

尽可能快地滴答直到赶上进度。这是默认行为。它会导致 `Interval` 快速触发滴答，以赶上它在时间上应该达到的位置。

```text
预期滴答: |     1     |     2     |     3     |     4     |     5     |     6     |
实际滴答:   | 工作 -----|          延迟         | 工作 | 工作 | 工作 -| 工作 -----|
```

#### Delay

将下一个滴答重新安排在上次调用 `tick()` 之后一个完整的 `period`，从而有效地将调度向前推移。

```text
预期滴答: |     1     |     2     |     3     |     4     |     5     |     6     |
实际滴答:   | 工作 -----|          延迟         | 工作 -----| 工作 -----| 工作 -----|
```

#### Skip

跳过任何错过的滴答，并在从原始开始时间算起的下一个 `period` 的倍数处安排下一个滴答。

```text
预期滴答: |     1     |     2     |     3     |     4     |     5     |     6     |
实际滴答:   | 工作 -----|          延迟         | 工作 ---| 工作 -----| 工作 -----|
```