# Timers

Tokio provides utilities for tracking time and scheduling work to happen in the future. These tools are essential for handling timeouts, delays, and periodic tasks in asynchronous applications.

At a high level, the `tokio::time` module offers three main types of utilities:

- **`Sleep`**: A future that completes at a specific point in time.
- **`Interval`**: A stream that yields values at a fixed period.
- **`Timeout`**: A wrapper that limits the maximum execution time for a future.

All timer utilities must be used from within the context of a Tokio [Runtime](./concepts-runtime.md), as they rely on its internal timer mechanism.

## Sleep: Pausing a Task

The most basic timer utility is `sleep`, which creates a future that completes after a specified duration has passed. It's the asynchronous equivalent of `std::thread::sleep`.

```rust
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    sleep(Duration::from_millis(100)).await;
    println!("100 ms have elapsed");
}
```

No work is performed while the `sleep` future is being awaited. It simply parks the current task and allows the scheduler to run other tasks until the specified time has been reached.

For waiting until a specific moment, you can use `sleep_until(instant)`. The `Sleep` future returned by these functions can also be modified. For instance, you can `reset` it to a new deadline, which is useful in `select!` loops.

```rust,no_run
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

## Interval: Repeating an Operation

An `Interval` is used to execute a task on a fixed schedule. It creates a stream that yields a value at a specified period.

The key difference between using `interval` and `sleep` inside a loop is how they handle the time spent on other work. An `Interval` accounts for the time that has passed since the *last* tick. If the work between ticks takes longer than expected, the next `.tick().await` will complete sooner (or immediately) to catch up.

This diagram illustrates the difference:

```d2
direction: down

"Loop with sleep(2s)": {
  shape: sequence_diagram

  "Task": {
    "Work (1s)": {lifespan: 1}
    "sleep(2s)": {lifespan: 2}
    "Work (1s) ": {lifespan: 1}
    "sleep(2s) ": {lifespan: 2}
  }
  
  note over "Task": "Total cycle time: 3s"
}

"Loop with interval(2s)": {
  shape: sequence_diagram

  "Task": {
    "Work (1s)": {lifespan: 1}
    "tick() await (1s)": {lifespan: 1}
    "Work (1s) ": {lifespan: 1}
    "tick() await (1s) ": {lifespan: 1}
  }

  note over "Task": "Total cycle time: 2s (as scheduled)"
}
```

Here is a practical example:

```rust
use tokio::time;

async fn task_that_takes_a_second() {
    println!("Performing a task...");
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

In this example, the task is executed approximately every two seconds, even though the task itself takes one second to run.

### Missed Tick Behavior

If a significant amount of time passes between calls to `.tick().await`, the `Interval` is said to have "missed" one or more ticks. The default behavior, `MissedTickBehavior::Burst`, is to fire ticks as quickly as possible until it has caught up. Other behaviors like `Delay` and `Skip` can be configured for more control over this scenario.

## Timeout: Setting a Deadline

Often, you need to limit how long an asynchronous operation is allowed to run. The `timeout` function wraps a future and cancels it if it doesn't complete within a specified duration.

It returns a `Result`. If the future completes before the timeout, it returns `Ok` with the future's output. If the timeout elapses first, it returns `Err(Elapsed)`.

```rust
use tokio::time::{timeout, Duration};

async fn long_running_operation() {
    // some work that might take too long
    sleep(Duration::from_secs(5)).await;
}

#[tokio::main]
async fn main() {
    let res = timeout(Duration::from_secs(1), long_running_operation()).await;

    if res.is_err() {
        println!("Operation timed out!");
    }
}
```

It's important to note that the timeout is checked *before* polling the wrapped future. If the future runs a long, CPU-bound computation without yielding (i.e., without reaching an `.await`), it might run past the deadline without being canceled immediately.

---

These timer utilities are fundamental building blocks for creating reliable, time-sensitive applications. For more detailed information on their APIs, see the [`tokio::time` API reference](./api-time.md). The next section explores the [Tokio Runtime](./concepts-runtime.md) in more detail.