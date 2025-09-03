# Time

Utilities for tracking time. This module provides a number of types for executing code after a set period of time. These types must be used from within the context of the Tokio [`Runtime`](./api-runtime.md).

The main time-related utilities are:

```d2
direction: down

"Time Utilities": {
    shape: package
    grid-columns: 3

    "Sleep": {
        shape: rectangle
        label: "Sleep\n(Wait for a specific duration)"
    }

    "Interval": {
        shape: rectangle
        label: "Interval\n(Execute code periodically)"
    }

    "Timeout": {
        shape: rectangle
        label: "Timeout\n(Limit future execution time)"
    }
}

"Your Async Task" -> "Sleep": "Uses to pause execution"
"Your Async Task" -> "Interval": "Uses for repeated tasks"
"Your Async Task" -> "Timeout": "Is wrapped by to enforce a deadline"

```

## Functions

### sleep()

Waits until `duration` has elapsed. This is an asynchronous analog to `std::thread::sleep`.

`pub fn sleep(duration: Duration) -> Sleep`

No work is performed while awaiting the sleep future. `Sleep` operates at millisecond granularity and may have a larger resolution on some platforms (like Windows).

#### Cancellation

Canceling a sleep instance is done by dropping the returned future. No additional cleanup work is required.

#### Panics

This function panics if no timer is configured in the Tokio runtime. This can happen if the runtime is built without `Builder::enable_time` or `Builder::enable_all`, or if `sleep` is called outside of a Tokio runtime context.

#### Example

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    sleep(Duration::from_millis(100)).await;
    println!("100 ms have elapsed");
}
```

### sleep_until()

Waits until `deadline` is reached.

`pub fn sleep_until(deadline: Instant) -> Sleep`

No work is performed while awaiting the sleep future to complete. Like `sleep`, it operates at millisecond granularity.

#### Panics

This function panics if there is no current timer set, similar to `sleep()`.

#### Example

```rust
use tokio::time::{sleep_until, Instant, Duration};

#[tokio::main]
async fn main() {
    sleep_until(Instant::now() + Duration::from_millis(100)).await;
    println!("100 ms have elapsed");
}
```

### interval()

Creates a new `Interval` that yields ticks at a specified `period`. The first tick completes immediately.

`pub fn interval(period: Duration) -> Interval`

The key difference between `interval` and `sleep` in a loop is that `Interval` accounts for the time spent between ticks. If a task takes some time, `interval` will shorten the next wait period to maintain the overall frequency, whereas a loop with `sleep` would drift.

#### Panics

This function panics if `period` is zero.

#### Example

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
This code executes the task approximately every two seconds. If `sleep` were used instead of `interval.tick()`, it would execute every three seconds.

### interval_at()

Creates a new `Interval` that yields with an interval of `period`, with the first tick completing at the specified `start` time.

`pub fn interval_at(start: Instant, period: Duration) -> Interval`

#### Panics

This function panics if `period` is zero.

#### Example

```rust
use tokio::time::{interval_at, Duration, Instant};

#[tokio::main]
async fn main() {
    let start = Instant::now() + Duration::from_millis(50);
    let mut interval = interval_at(start, Duration::from_millis(10));

    interval.tick().await; // ticks after 50ms
    interval.tick().await; // ticks after 10ms
    interval.tick().await; // ticks after 10ms

    // approximately 70ms have elapsed.
}
```

### timeout()

Requires a future to complete before a specified `duration` has elapsed.

`pub fn timeout<F>(duration: Duration, future: F) -> Timeout<F::IntoFuture>`

If the future completes in time, its value is returned as `Ok(value)`. Otherwise, an `Err(Elapsed)` is returned, and the future is canceled. The timeout is checked *before* polling the future, so a future that doesn't yield might run past the deadline without returning an error.

#### Panics

This function panics if there is no current timer set.

#### Example

```rust
use tokio::time::{timeout, Duration};
use tokio::sync::oneshot;

async fn long_future() {
    // Simulate work
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

Requires a future to complete before a specified `deadline`.

`pub fn timeout_at<F>(deadline: Instant, future: F) -> Timeout<F::IntoFuture>`

This functions similarly to `timeout` but takes a specific `Instant` as the deadline.

#### Example

```rust
use tokio::time::{Instant, timeout_at, Duration};
use tokio::sync::oneshot;

# async fn dox() {
let (tx, rx) = oneshot::channel();
# tx.send(()).unwrap();

// Wrap the future with a `Timeout` set to expire 10 milliseconds into the
// future.
if let Err(_) = timeout_at(Instant::now() + Duration::from_millis(10), rx).await {
    println!("did not receive value within 10 ms");
}
# }
```

## Structs

### Sleep

A future returned by `sleep` and `sleep_until` that completes at a specified instant.

This type does not implement `Unpin`. If you use it with `select!` or poll it manually, you must pin it first, for example with `tokio::pin!`.

#### Methods

| Method | Description |
|---|---|
| `deadline()` | Returns the `Instant` at which the future will complete. |
| `is_elapsed()` | Returns `true` if the `Sleep` instance has elapsed. |
| `reset(deadline: Instant)` | Resets the `Sleep` instance to a new deadline. |

#### Example: Resetting a Sleep

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

A stream that yields at a regular interval.

This type allows you to wait on a sequence of instants. Unlike calling `sleep` in a loop, it compensates for the time spent between calls to `tick()`.

#### Methods

| Method | Description |
|---|---|
| `tick()` | Asynchronously waits for the next tick, returning the `Instant` it was scheduled for. |
| `poll_tick(&mut self, cx: &mut Context<'_'>)` | Polls for the next tick to be reached. |
| `reset()` | Resets the interval to complete one period after the current time. |
| `reset_at(deadline: Instant)` | Sets the next tick to expire at the given instant. |
| `period()` | Returns the period of the interval. |
| `missed_tick_behavior()` | Returns the current `MissedTickBehavior` strategy. |
| `set_missed_tick_behavior(behavior)` | Sets the `MissedTickBehavior` strategy. |

### Timeout<T>

A future returned by `timeout` and `timeout_at`.

This future wraps another future, limiting its execution time.

#### Methods

| Method | Description |
|---|---|
| `get_ref()` | Gets a reference to the underlying future. |
| `get_mut()` | Gets a mutable reference to the underlying future. |
| `into_inner()` | Consumes the timeout, returning the underlying future. |

### Instant

A measurement of a monotonically nondecreasing clock.

This type is a wrapper around `std::time::Instant` and is used to align with Tokio's internal clock, which is especially useful for testing with features like `time::pause()` and `time::advance()`.

#### Methods

| Method | Description |
|---|---|
| `now()` | Returns an `Instant` corresponding to "now". |
| `duration_since(earlier: Instant)` | Returns the `Duration` elapsed between `earlier` and this instant. |
| `elapsed()` | Returns the `Duration` elapsed since this instant was created. |
| `checked_add(duration: Duration)` | Adds a `Duration` to the `Instant`, returning `None` on overflow. |
| `checked_sub(duration: Duration)` | Subtracts a `Duration` from the `Instant`, returning `None` on overflow. |

## Enums

### MissedTickBehavior

Defines the behavior of an `Interval` when it misses a tick, for example, if the task between ticks takes longer than the interval period.

#### Burst

Ticks as fast as possible until caught up. This is the default behavior. It results in the `Interval` firing ticks rapidly to catch up to where it should have been in time.

```text
Expected ticks: |     1     |     2     |     3     |     4     |     5     |     6     |
Actual ticks:   | work -----|          delay          | work | work | work -| work -----|
```

#### Delay

Reschedules the next tick to be a full `period` from the time `tick()` was last called, effectively shifting the schedule forward.

```text
Expected ticks: |     1     |     2     |     3     |     4     |     5     |     6     |
Actual ticks:   | work -----|          delay          | work -----| work -----| work -----|
```

#### Skip

Skips any missed ticks and schedules the next tick at the next multiple of `period` from the original start time.

```text
Expected ticks: |     1     |     2     |     3     |     4     |     5     |     6     |
Actual ticks:   | work -----|          delay          | work ---| work -----| work -----|
```