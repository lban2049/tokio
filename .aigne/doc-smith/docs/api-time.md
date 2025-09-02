# Time

Utilities for tracking time. This module provides a number of types for executing code after a set period of time. These types must be used from within the context of the Tokio [`Runtime`](./api-runtime.md).

```d2
direction: down

subsystem: {
  shape: package
  label: "Future / Stream"
}

sleep: {
  shape: step
  label: "Sleep\n(A Future that completes after a duration)"
}

interval: {
  shape: queue
  label: "Interval\n(A Stream that yields at a fixed period)"
}

timeout: {
  shape: hexagon
  label: "Timeout\n(Wraps another Future with a time limit)"
}

subsystem -> timeout: "Wraps"
sleep -> interval: "Internally uses"
```

Key components include:

*   **[`Sleep`](#sleep):** A future that completes at a specific `Instant` in time.
*   **[`Interval`](#interval):** A stream that yields values at a fixed period.
*   **[`Timeout`](#timeout):** A wrapper that cancels a future if it takes too long to complete.
*   **[`Instant`](#instant):** A measurement of a monotonically nondecreasing clock.

---

## Instant

A measurement of a monotonically nondecreasing clock, useful for measuring benchmarks or timing operations. This type is a wrapper around the standard library's `std::time::Instant` but is integrated with Tokio's test utilities like `time::pause()` and `time::advance()`.

### `now()`

Returns an instant corresponding to "now".

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

### Other Methods

| Method | Description |
|---|---|
| `from_std(std)` | Creates a `tokio::time::Instant` from a `std::time::Instant`. |
| `into_std(self)` | Converts the value into a `std::time::Instant`. |
| `duration_since(earlier)` | Returns the amount of time elapsed from another instant to this one. |
| `checked_duration_since(earlier)` | Returns `Option<Duration>` representing the time elapsed, or `None` if `earlier` is later. |
| `elapsed()` | Returns the amount of time elapsed since this instant was created. |
| `checked_add(duration)` | Adds a `Duration` to the `Instant`, returning `None` on overflow. |
| `checked_sub(duration)` | Subtracts a `Duration` from the `Instant`, returning `None` if it would go before the clock's epoch. |


---

## Sleep

A future that does no work and completes at a specific `Instant` in time. It operates at millisecond granularity.

### `sleep(duration: Duration) -> Sleep`

Waits until `duration` has elapsed. This is an asynchronous analog to `std::thread::sleep`.

**Cancellation:** Dropping the `Sleep` future is sufficient to cancel the wait.

**Panics:** This function panics if called outside of a Tokio runtime context (i.e., without an active timer driver).

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

Waits until the specified `deadline` is reached.

```rust
use tokio::time::{sleep_until, Instant, Duration};

#[tokio::main]
async fn main() {
    let deadline = Instant::now() + Duration::from_millis(100);
    sleep_until(deadline).await;
    println!("100 ms have elapsed");
}
```

### The `Sleep` Struct

The `Sleep` future can be manipulated after creation, for example, to reset its deadline.

#### `reset(self: Pin<&mut Self>, deadline: Instant)`

Resets the `Sleep` instance to a new deadline. This is useful for reusing a timer in a loop without creating new state.

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

A stream that yields a value at a fixed period. Unlike `sleep` in a loop, `Interval` accounts for the time spent between calls to `tick()`, preventing drift.

### `interval(period: Duration) -> Interval`

Creates a new `Interval` that starts ticking immediately and continues every `period`.

**Panics:** Panics if `period` is zero.

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

Creates a new `Interval` that begins ticking at the specified `start` time.

```rust
use tokio::time::{interval_at, Duration, Instant};

#[tokio::main]
async fn main() {
    let start = Instant::now() + Duration::from_millis(50);
    let mut interval = interval_at(start, Duration::from_millis(10));

    interval.tick().await; // ticks after 50ms
    interval.tick().await; // ticks after 10ms
    // approximately 60ms have elapsed since start.
}
```

### `tick(&mut self) -> impl Future<Output = Instant>`

Completes when the next instant in the interval has been reached.

### Missed Tick Behavior

If the consumer of an `Interval` takes longer than the specified period to call `tick()`, a tick is considered "missed". The `MissedTickBehavior` enum configures how the interval catches up.

| Behavior | Description |
|---|---|
| `Burst` **(Default)** | Fires ticks as quickly as possible until it is caught up in time to where it should be. The instants yielded are the same as if no ticks were missed. |
| `Delay` | Resets the interval's period from the moment `tick()` was called, effectively delaying all future ticks. Ticks are not shortened. |
| `Skip` | Skips any missed ticks and schedules the next tick at the next multiple of the original period from the start time. This may shorten the next tick duration. |

You can change this behavior using `set_missed_tick_behavior()`.

---

## Timeout

A wrapper for futures or streams that sets an upper bound on their execution time. If the future does not complete within the specified time, it is canceled, and an `Elapsed` error is returned.

### `timeout(duration: Duration, future: F) -> Timeout<F>`

Requires a future to complete before `duration` has elapsed.

**Panics:** This function panics if called outside of a Tokio runtime context.

```rust
use tokio::time::{timeout, Duration};

async fn long_future() {
    // some long-running work
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

Requires a future to complete before a specific `deadline`.

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

### The `Timeout` Struct

The original future can be retrieved by consuming the `Timeout` wrapper.

#### `into_inner(self) -> T`

Consumes the `Timeout`, returning the underlying future. This is useful if you want to continue working with the future after it has been wrapped.

---

### Re-exports

For convenience, `std::time::Duration` is re-exported as `tokio::time::Duration`.