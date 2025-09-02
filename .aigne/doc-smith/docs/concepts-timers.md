# Timers

Tokio provides utilities for tracking time and scheduling work to run in the future. These tools are essential for handling delays, periodic tasks, and deadlines in asynchronous applications. All timer utilities require a Tokio `Runtime` to be active.

Tokio's time-related functionality is primarily composed of three types:

<x-cards data-columns="3">
  <x-card data-title="Sleep" data-icon="lucide:timer-off">Pauses the current task for a specified duration or until a specific instant in time.</x-card>
  <x-card data-title="Interval" data-icon="lucide:repeat">Creates a stream that yields values at a fixed period, ideal for recurring tasks.</x-card>
  <x-card data-title="Timeout" data-icon="lucide:alarm-clock">Wraps a future, enforcing a time limit on its execution.</x-card>
</x-cards>

## Sleep: Pausing Execution

The most basic timer is `tokio::time::sleep`. This function creates a future that completes after a specified duration has passed. It's an asynchronous equivalent of `std::thread::sleep`.

You can use `sleep` to introduce a delay into a task without blocking the entire thread, allowing other tasks to run concurrently.

```rust
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    println!("Waiting...");
    sleep(Duration::from_millis(500)).await;
    println!("500 ms have elapsed");
}
```

Similarly, `sleep_until(deadline)` can be used to pause a task until a specific `Instant` is reached.

> **Note on Panics**: Using timer functions like `sleep` requires a Tokio runtime with the timer enabled. If you call a timer function outside of a running Tokio runtime, or on a runtime that was built without the timer (`Builder::enable_time` or `Builder::enable_all`), your program will panic.

## Interval: Repeating on a Schedule

For tasks that need to run repeatedly at a fixed frequency, you could use `sleep` in a loop. However, this approach can lead to drift over time, as the time taken by the task itself is not accounted for. The total cycle time becomes `sleep_duration + task_duration`.

Tokio's `tokio::time::interval` provides a more precise solution. An `Interval` produces a stream of "ticks" at a specified period. When you `await` the next tick, the interval calculates the correct delay to maintain the desired frequency, factoring in the time your task spent working since the last tick.

The diagram below illustrates the difference:

```d2
direction: down

subsystem_A: "Inaccurate: Loop with sleep" {
  timeline_A: {
    direction: right
    shape: sequence_diagram

    start_1: "Cycle 1 Start (0s)"
    sleep_1: "sleep(2s).await"
    work_1: "work(1s).await"
    end_1: "Cycle 1 End (~3s)"

    start_1 -> sleep_1
    sleep_1 -> work_1
    work_1 -> end_1
  }
  style.stroke: "#D83B01"
  "Total time drifts by +1s each cycle"
}

subsystem_B: "Accurate: Loop with interval" {
  timeline_B: {
    direction: right
    shape: sequence_diagram

    start_2: "Cycle 1 Start (0s)"
    tick_1: "interval.tick().await"
    work_2: "work(1s).await"
    end_2: "Cycle 1 End (2s)"
    tick_2: "interval.tick().await"

    start_2 -> tick_1: "Immediate"
    tick_1 -> work_2
    work_2 -> end_2
    end_2 -> tick_2: "Waits ~1s"
  }
  style.stroke: "#107C10"
  "Total time remains consistent at 2s per cycle"
}
```

Here is a practical example of using `interval`:

```rust
use tokio::time;

async fn task_that_takes_a_second() {
    println!("Executing task...");
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

### Handling Missed Ticks

If your task takes longer than the interval period, a tick is considered "missed." `Interval` provides a `MissedTickBehavior` enum to control how it catches up. You can set it using `set_missed_tick_behavior()`.

*   **`Burst` (Default):** The interval fires ticks as fast as possible until it catches up. The timestamps of the ticks are what they would have been if no delay occurred.
*   **`Delay`:** The next tick is scheduled relative to when the last `tick()` call completed, effectively resetting the interval's schedule. This ensures the full period always passes between ticks.
*   **`Skip`:** The interval skips any missed ticks and schedules the next tick at the next multiple of the original period. This can result in a shorter-than-usual delay to get back on schedule.

## Timeout: Enforcing Deadlines

To prevent a future from running indefinitely, you can wrap it with `tokio::time::timeout`. If the future doesn't complete within the specified duration, it is cancelled, and the `timeout` future resolves to an error.

The `timeout` function returns a `Result`. If the inner future completes in time, you get `Ok(value)`; otherwise, you get `Err(Elapsed)`.

```rust
use tokio::time::{timeout, Duration};
use tokio::sync::oneshot;

async fn long_running_task() {
    // Simulate a task that might or might not complete in time.
    tokio::time::sleep(Duration::from_millis(150)).await;
}

#[tokio::main]
async fn main() {
    let duration = Duration::from_millis(100);

    if let Err(_) = timeout(duration, long_running_task()).await {
        println!("The task timed out after {:?}", duration);
    } else {
        println!("The task completed within {:?}", duration);
    }
}
```

Like `sleep`, `timeout` has a `timeout_at(deadline)` variant that works with a specific `Instant`.

These timer utilities provide the fundamental building blocks for controlling the flow of time in your asynchronous applications. For more detailed information, see the [`tokio::time` API reference](./api-time.md).