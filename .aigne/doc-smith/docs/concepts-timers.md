# Timers

Tokio provides utilities for tracking time and scheduling work to be executed after a set period. These tools are essential for handling delays, periodic tasks, and operations with deadlines.

All timer utilities must be used within the context of a Tokio [Runtime](./concepts-runtime.md), as they rely on its internal timer for scheduling.

Tokio's primary time-related components are:

- **`Sleep`**: A future that completes at a specific instant.
- **`Interval`**: A stream that yields values at a fixed period.
- **`Timeout`**: A wrapper that limits the maximum execution time for a future.

Let's explore each of these concepts.

## Sleep: Waiting for a Duration

The most basic timer primitive is `sleep`. It creates a future that completes after a specified duration has passed. This is the asynchronous equivalent of `std::thread::sleep`.

`sleep` is useful when you need to pause a task for a fixed amount of time without blocking the entire thread.

```rust
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    sleep(Duration::from_millis(100)).await;
    println!("100 ms have elapsed");
}
```

To wait until a specific moment in time, you can use `sleep_until(instant)`. If you drop the `Sleep` future before it completes, the timer is cancelled, and no additional cleanup is needed.

## Interval: Repeating on a Schedule

While you could use `sleep` in a loop to perform a task periodically, this approach can lead to drift. The time it takes to execute the task itself is not accounted for, so the actual period will be `sleep_duration + task_duration`.

For more precise periodic tasks, Tokio provides `interval`. An `interval` measures time from the completion of the *last* tick, ensuring that ticks occur at a more consistent frequency.

The diagram below illustrates the difference:

```d2
direction: down

"Loop with sleep(2s)": {
  shape: package
  grid-columns: 1

  "Timeline": {
    shape: sequence_diagram
    "Task": {shape: step}
    "Sleep": {shape: step}

    "0s" -> "Task": "Tick 1"
    "Task" -> "Sleep": "Work (1s)"
    "Sleep" -> "3s": "Sleep (2s)"
    "3s": "Tick 2"
  }
  "Total period becomes ~3 seconds."
}

"Interval(2s)": {
  shape: package
  grid-columns: 1

  "Timeline": {
    shape: sequence_diagram
    "Task": {shape: step}
    "Wait": {shape: step}

    "0s" -> "Task": "Tick 1"
    "Task" -> "Wait": "Work (1s)"
    "Wait" -> "2s": "Wait (~1s)"
    "2s": "Tick 2"
  }
  "Total period remains 2 seconds."
}
```

Here is a practical example of using `interval` to run a task every two seconds, where the task itself takes one second:

```rust
use tokio::time;

async fn task_that_takes_a_second() {
    println!("Executing task...");
    time::sleep(time::Duration::from_secs(1)).await
}

#[tokio::main]
async fn main() {
    let mut interval = time::interval(time::Duration::from_secs(2));
    for _i in 0..3 {
        interval.tick().await;
        task_that_takes_a_second().await;
    }
}
```
In this example, the message "Executing task..." will be printed at approximately t=0s, t=2s, and t=4s.

### Handling Missed Ticks

If your task takes longer to execute than the interval's period, the interval is said to have "missed a tick." You can configure how `Interval` behaves in this scenario using `MissedTickBehavior`:

- **`Burst` (Default):** The interval will fire ticks as fast as possible until it catches up.
- **`Delay`:** The next tick is scheduled relative to the current time, effectively resetting the interval's phase.
- **`Skip`:** The interval skips the missed ticks and schedules the next tick at the next regular interval point.

## Timeout: Bounding Execution Time

Often, you need to ensure an operation doesn't take too long to complete. The `timeout` function wraps a future and imposes a time limit on its execution.

If the inner future completes within the duration, `timeout` returns `Ok(value)`. If the duration elapses first, the inner future is cancelled, and `timeout` returns `Err(Elapsed)`.

```rust
use tokio::time::{timeout, Duration};

async fn long_running_task() {
    // This task takes longer than the timeout.
    sleep(Duration::from_secs(5)).await;
    println!("Task finished!");
}

#[tokio::main]
async fn main() {
    let res = timeout(Duration::from_secs(1), long_running_task()).await;

    if res.is_err() {
        println!("The operation timed out!");
    }
}
```

This is useful for adding deadlines to I/O operations, such as network requests or database queries, to prevent your application from hanging indefinitely.

---

Tokio's time utilities—`sleep`, `interval`, and `timeout`—provide the essential tools for managing time-based logic in asynchronous applications. For a complete list of functions and detailed options, see the [Time API Reference](./api-time.md).