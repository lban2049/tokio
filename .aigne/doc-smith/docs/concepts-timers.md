# Timers

Tokio provides utilities for tracking time and scheduling work to be executed after a certain period. These tools are essential for handling timeouts, delays, and periodic tasks in asynchronous applications. The primary time-related components are:

*   **Sleep**: A future that completes at a specific point in time.
*   **Interval**: A stream that yields values at a fixed period.
*   **Timeout**: A wrapper that limits the maximum execution time for a future.

All timer utilities require being used within the context of a Tokio [Runtime](./concepts-runtime.md), which provides the necessary timer implementation.

## Sleeping: Pausing a Task

The simplest way to introduce a delay is to use `tokio::time::sleep`. This function returns a future that completes after the specified duration has elapsed. It's the asynchronous equivalent of `std::thread::sleep`.

No work is performed while the `Sleep` future is being awaited. This allows other tasks to run concurrently.

```rust main.rs icon=logos:rust
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    println!("Waiting...");
    sleep(Duration::from_millis(100)).await;
    println!("100 ms have elapsed");
}
```

If you need to wait until a specific moment in time, you can use `sleep_until(deadline)`, where `deadline` is an `Instant`.

**Cancellation**: Canceling a sleep operation is as simple as dropping the `Sleep` future. No extra cleanup is needed.

**Panics**: Using `sleep` outside of a Tokio runtime will cause a panic. This is because the function needs access to the runtime's timer driver. Ensure it is called within an `async` block that is managed by Tokio, for example via `#[tokio::main]` or `runtime.block_on()`.

## Intervals: Repeating an Operation

For tasks that need to run repeatedly on a schedule, Tokio provides `tokio::time::interval`. An interval yields ticks at a specified period. The key difference from calling `sleep` in a loop is that an `Interval` accounts for the time spent on the task itself, ensuring that ticks occur at a more regular frequency.

If you use `sleep` in a loop, the total time for each iteration will be the sleep duration *plus* the task execution time. With `interval`, the time between ticks remains consistent, even if the task takes some time to execute.

```rust main.rs icon=logos:rust
use tokio::time;

async fn task_that_takes_a_second() {
    println!("Executing task...");
    time::sleep(time::Duration::from_secs(1)).await;
}

#[tokio::main]
async fn main() {
    let mut interval = time::interval(time::Duration::from_secs(2));
    // The first tick completes immediately.
    for _i in 0..5 {
        interval.tick().await;
        task_that_takes_a_second().await;
    }
}
```
In the example above, the loop will run approximately every two seconds. If `sleep` were used instead of `interval.tick()`, it would run every three seconds (2s sleep + 1s task).

### Missed Tick Behavior

If the code between `tick().await` calls takes longer than the interval period, one or more ticks might be "missed." You can configure how the interval handles this using `set_missed_tick_behavior()` with one of the following strategies:

*   `Burst` (Default): The interval will fire ticks as fast as possible to catch up to where it should have been.
*   `Delay`: The next tick is scheduled from the current moment, effectively resetting the interval's phase.
*   `Skip`: The interval skips the missed ticks and waits for the next regularly scheduled tick.

## Timeouts: Setting a Deadline

To prevent a future from running indefinitely, you can wrap it in a `tokio::time::timeout`. This function takes a duration and a future, and returns a new future that resolves to a `Result`.

*   If the inner future completes before the duration elapses, the new future resolves to `Ok(value)`.
*   If the duration elapses before the inner future completes, the new future resolves to `Err(Elapsed)` and the inner future is canceled.

This is crucial for building robust network applications where remote services might become unresponsive.

```rust main.rs icon=logos:rust
use tokio::time::{timeout, Duration};

async fn long_running_operation() {
    // This operation might take a long time.
    tokio::time::sleep(Duration::from_secs(5)).await;
    println!("Operation complete!");
}

#[tokio::main]
async fn main() {
    let res = timeout(Duration::from_secs(1), long_running_operation()).await;

    if res.is_err() {
        println!("Operation timed out!");
    }
}
```

---

You now have a conceptual understanding of how to manage time-based operations in Tokio. For a complete list of functions and detailed configuration options, please refer to the [Time API Reference](./api-time.md).