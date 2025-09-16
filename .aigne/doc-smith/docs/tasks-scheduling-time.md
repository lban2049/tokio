# Time, Delays, and Timeouts

Tokio's time module provides essential utilities for managing time-based asynchronous operations. Whether you need to pause a task, run something periodically, or ensure an operation doesn't run indefinitely, the `tokio::time` module has the tools you need. These types must be used from within the context of a Tokio [Runtime](./tasks-scheduling-runtime.md).

This guide covers three primary components:

- **`Sleep`**: A future that completes at a specific point in time, useful for creating delays.
- **`Interval`**: A stream that yields values at a fixed period, ideal for recurring tasks.
- **`Timeout`**: A wrapper that limits the maximum execution time for a future.

## Delays with `sleep`

The most straightforward way to introduce a delay in an asynchronous task is by using the `sleep` function. It creates a future that completes after a specified duration has passed. It's an asynchronous equivalent to `std::thread::sleep`.

```rust Waiting for a duration icon=logos:rust
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    sleep(Duration::from_millis(100)).await;
    println!("100 ms have elapsed");
}
```

If you need to wait until a specific moment in time, you can use `sleep_until(deadline)`, which takes an `Instant` as an argument.

### The `Sleep` Future

Both `sleep` and `sleep_until` return a `Sleep` future. This future can be manipulated, most notably by resetting its deadline. This is useful for reusing the timer in loops without allocating a new one each time, often in combination with `tokio::select!`.

In the example below, the `sleep` future is pinned and its deadline is reset within the loop after it elapses.

```rust Reusing a Sleep future icon=logos:rust
use tokio::time::{self, Duration, Instant};

#[tokio::main]
async fn main() {
    let sleep = time::sleep(Duration::from_millis(10));
    tokio::pin!(sleep);

    loop {
        tokio::select! {
            () = &mut sleep => {
                println!("timer elapsed");
                // Reset the deadline to 50ms from now
                sleep.as_mut().reset(Instant::now() + Duration::from_millis(50));
            },
        }
    }
}
```

### Important Considerations

Creating a timer-based future like `Sleep` will panic if a Tokio runtime is not active or if the runtime was not configured with the timer enabled. Ensure your `Builder` includes `.enable_time()` or `.enable_all()`, or that you are using the `#[tokio::main]` macro (which enables it by default).

```rust
// This will panic because sleep() is called outside the runtime context.
// let rt = tokio::runtime::Builder::new_current_thread().build().unwrap();
// rt.block_on(sleep(Duration::from_secs(1)));

// This is correct. The async block is executed inside the runtime.
let rt = tokio::runtime::Builder::new_current_thread().enable_all().build().unwrap();
rt.block_on(async {
    sleep(Duration::from_secs(1)).await;
});
```

## Periodic Tasks with `interval`

For tasks that need to run repeatedly on a fixed schedule, `tokio::time::interval` is more suitable than a loop with `sleep`. An `Interval` automatically adjusts for the time spent on other work between ticks, ensuring that the ticks average out to the specified period.

If you used `sleep` in a loop, the total time for each iteration would be `sleep_duration + task_duration`. With `interval`, the time between ticks remains consistent.

```rust Running a task every two seconds icon=logos:rust
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
In this example, even though the task takes one second, a new tick will occur approximately every two seconds.

### Handling Missed Ticks

If the code between `interval.tick().await` calls takes longer than the interval's period, one or more ticks might be "missed". You can configure how `Interval` behaves in this scenario using `MissedTickBehavior`.

| Behavior | Description |
| :--- | :--- |
| `Burst` (Default) | The interval will fire ticks as quickly as possible to catch up to where it should have been. |
| `Delay` | The next tick is scheduled relative to when `tick()` was last called, effectively delaying the schedule. |
| `Skip` | The interval skips the missed ticks and schedules the next tick at the next multiple of the original period. |

```rust Setting MissedTickBehavior icon=logos:rust
use tokio::time::{interval, Duration, MissedTickBehavior};

let mut interval = interval(Duration::from_millis(50));
interval.set_missed_tick_behavior(MissedTickBehavior::Delay);
```

## Bounding Execution with `timeout`

To prevent a future from running indefinitely, you can wrap it with `tokio::time::timeout`. If the wrapped future doesn't complete within the specified duration, the `timeout` future completes with an `Err`.

The `timeout` function returns a `Result`. If the future completes successfully within the time limit, it returns `Ok(value)`. Otherwise, it returns `Err(Elapsed)`.

```rust Using timeout icon=logos:rust
use tokio::time::{timeout, Duration};
use tokio::sync::oneshot;

async fn some_long_operation() {
    // Simulate work
    tokio::time::sleep(Duration::from_millis(500)).await;
}

#[tokio::main]
async fn main() {
    match timeout(Duration::from_millis(100), some_long_operation()).await {
        Ok(_) => println!("Operation completed successfully."),
        Err(_) => println!("Operation timed out!"),
    };
}
```

If you need to get the original future back after a timeout, you can use the `into_inner()` method on the `Timeout` future.

---

You now have the tools to control the timing of your asynchronous operations. Understanding how to manage delays, periodic tasks, and timeouts is crucial for building robust, reliable applications. To learn more about the engine that drives these tasks, proceed to the next section.

<x-card data-title="Next: The Runtime" data-href="/tasks-scheduling/runtime" data-icon="lucide:cpu">
  Understand the Tokio runtime, its schedulers, and how to configure it.
</x-card>