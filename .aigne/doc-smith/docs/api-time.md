# Time

Utilities for tracking time and scheduling work. This module provides several types for executing code after a set period.

These types require a Tokio runtime. They will panic if used outside of a runtime context.

<x-cards data-columns="2">
  <x-card data-title="Sleep" data-icon="lucide:timer-off">
    An asynchronous version of `std::thread::sleep`. It pauses the current task for a specified duration or until a specific instant.
  </x-card>
  <x-card data-title="Interval" data-icon="lucide:timer">
    A stream that yields a value at a fixed period. Useful for running a task on a schedule.
  </x-card>
  <x-card data-title="Timeout" data-icon="lucide:alarm-clock-off">
    Wraps a future, setting an upper bound on its execution time. If the future doesn't complete within the timeout, it's cancelled.
  </x-card>
  <x-card data-title="Instant" data-icon="lucide:clock">
    A measurement of a monotonically nondecreasing clock, used for timing operations.
  </x-card>
</x-cards>

## Sleep

Tokio provides two functions to pause a task: `sleep` and `sleep_until`.

### `sleep(duration: Duration)`

Waits until `duration` has elapsed. This is an asynchronous analog to `std::thread::sleep`.

```rust icon=logos:rust
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    sleep(Duration::from_millis(100)).await;
    println!("100 ms have elapsed");
}
```

### `sleep_until(deadline: Instant)`

Waits until a specific `Instant` in time is reached. 

```rust icon=logos:rust
use tokio::time::{sleep_until, Instant, Duration};

#[tokio::main]
async fn main() {
    sleep_until(Instant::now() + Duration::from_millis(100)).await;
    println!("100 ms have elapsed");
}
```

Both functions return a `Sleep` future. The `Sleep` future can be manipulated, for example, to reset its deadline.

```rust Reseting a Sleep future icon=logos:rust
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

## Interval

An `Interval` is a stream that yields values at a fixed period. This is useful for running a task on a regular schedule.

The key difference between `interval` and `sleep` in a loop is that an `Interval` accounts for the time spent in the task between ticks. If a task takes longer than the interval period, the next tick will fire immediately.

### `interval(period: Duration)`

Creates a new `Interval` that starts ticking immediately and continues every `period`.

```rust Execute a task every 2 seconds icon=logos:rust
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

### Missed Tick Behavior

If the consumer of an `Interval` is too slow, ticks may be missed. The `MissedTickBehavior` enum allows you to configure how the interval should behave in this scenario.

| Behavior | Description |
|---|---|
| `Burst`  | (Default) Fires ticks as quickly as possible until it's caught up. |
| `Delay`  | Resets the interval, scheduling the next tick from the current moment. |
| `Skip`   | Skips any missed ticks and waits for the next scheduled tick. |

You can change the behavior using `set_missed_tick_behavior`.

```rust icon=logos:rust
use tokio::time::{interval, Duration, MissedTickBehavior};

let mut interval = interval(Duration::from_millis(50));
interval.set_missed_tick_behavior(MissedTickBehavior::Skip);
```

## Timeout

Timeouts are used to limit the amount of time a future is allowed to execute. The `timeout` function wraps a future and returns a `Result`.

- `Ok(value)`: The future completed successfully within the time limit.
- `Err(Elapsed)`: The time limit was reached, and the future was cancelled.

### `timeout(duration: Duration, future: F)`

Requires a future to complete within a specified duration.

```rust Require an operation to complete within 1 second icon=logos:rust
use tokio::time::{timeout, Duration};

async fn long_future() {
    // some long-running work
    tokio::time::sleep(Duration::from_secs(5)).await;
}

#[tokio::main]
async fn main() {
    let res = timeout(Duration::from_secs(1), long_future()).await;

    if res.is_err() {
        println!("operation timed out");
    }
}
```

There is also a `timeout_at(deadline: Instant, future: F)` variant that takes a specific deadline instead of a duration.

## Instant

Tokio's `time::Instant` is a wrapper around the standard library's `std::time::Instant`. It is used by all Tokio time-related APIs and is essential for working correctly with Tokio's testing utilities, which allow for time manipulation.

It provides the same core functionality for measuring elapsed time.

```rust Measuring elapsed time icon=logos:rust
use tokio::time::{Duration, Instant, sleep};

#[tokio::main]
async fn main() {
    let instant = Instant::now();
    let three_secs = Duration::from_secs(3);
    sleep(three_secs).await;
    assert!(instant.elapsed() >= three_secs);
    println!("Successfully waited for at least 3 seconds.");
}
```
