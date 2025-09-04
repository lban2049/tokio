# Time

Utilities for tracking time. This module provides a number of types for executing code after a set period of time. These types must be used from within the context of the Tokio [`Runtime`](./api-runtime.md).

<x-cards data-columns="3">
  <x-card data-title="Sleep" data-icon="lucide:timer">
    A future that does no work and completes at a specific `Instant` in time.
  </x-card>
  <x-card data-title="Interval" data-icon="lucide:repeat">
    A stream yielding a value at a fixed period. It is initialized with a `Duration` and repeatedly yields each time the duration elapses.
  </x-card>
  <x-card data-title="Timeout" data-icon="lucide:alarm-clock-off">
    Wraps a future or stream, setting an upper bound to the amount of time it is allowed to execute.
  </x-card>
</x-cards>

## Functions

### sleep

`pub fn sleep(duration: Duration) -> Sleep`

Waits until `duration` has elapsed. This is an asynchronous analog to `std::thread::sleep`.

No work is performed while awaiting on the sleep future to complete. `Sleep` operates at millisecond granularity and should not be used for tasks that require high-resolution timers.

To run something regularly on a schedule, see [`interval`](#interval).

#### Cancellation

Canceling a sleep instance is done by dropping the returned future. No additional cleanup work is required.

#### Panics

This function panics if there is no current timer set, which can be triggered if the `enable_time` or `enable_all` features of the runtime builder are not enabled. It can also panic if a timer is created outside of a Tokio runtime.

#### Example

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

Waits until `deadline` is reached.

No work is performed while awaiting on the sleep future to complete. `Sleep` operates at millisecond granularity and should not be used for tasks that require high-resolution timers.

#### Example

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

Creates a new `Interval` that yields ticks with an interval of `period`. The first tick completes immediately. The default missed tick behavior is `Burst`.

An interval will tick indefinitely. Dropping the `Interval` value cancels it.

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

### interval_at

`pub fn interval_at(start: Instant, period: Duration) -> Interval`

Creates a new `Interval` that yields with an interval of `period`, with the first tick completing at `start`.

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

### timeout

`pub fn timeout<F>(duration: Duration, future: F) -> Timeout<F::IntoFuture>`

Requires a `Future` to complete before the specified duration has elapsed. If the future completes in time, its value is returned in `Ok`. Otherwise, an `Err` containing an `Elapsed` error is returned, and the future is canceled.

#### Cancellation

Cancelling a timeout is done by dropping the `Timeout` future. The original future can be retrieved by calling `Timeout::into_inner`.

#### Example

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

Requires a `Future` to complete before the specified `Instant` in time.

#### Example

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

## Structs

### Sleep

A future returned by `sleep` and `sleep_until` that completes at a specified instant.

This type does not implement `Unpin`. If you use it with `select!` or by calling `poll`, you must pin it first.

#### Methods

| Method | Description |
|---|---|
| `deadline(&self) -> Instant` | Returns the instant at which the future will complete. |
| `is_elapsed(&self) -> bool` | Returns `true` if `Sleep` has elapsed. |
| `reset(self: Pin<&mut Self>, deadline: Instant)` | Resets the `Sleep` instance to a new deadline. |

#### Example: Resetting a `Sleep` in a loop

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

A type that allows you to wait for a sequence of instants with a certain duration between them.

#### Methods

| Method | Description |
|---|---|
| `tick(&mut self) -> impl Future<Output = Instant>` | Completes when the next instant in the interval has been reached. |
| `poll_tick(&mut self, cx: &mut Context<'_'>) -> Poll<Instant>` | Polls for the next instant in the interval to be reached. |
| `reset(&mut self)` | Resets the interval to complete one period after the current time. |
| `reset_at(&mut self, deadline: Instant)` | Sets the next tick to expire at the given instant. |
| `period(&self) -> Duration` | Returns the period of the interval. |
| `missed_tick_behavior(&self) -> MissedTickBehavior` | Returns the current `MissedTickBehavior` strategy. |
| `set_missed_tick_behavior(&mut self, behavior: MissedTickBehavior)` | Sets the `MissedTickBehavior` strategy. |


### Timeout<T>

A future that cancels another future if it takes too long to complete. Returned by `timeout` and `timeout_at`.

#### Methods

| Method | Description |
|---|---|
| `get_ref(&self) -> &T` | Gets a reference to the underlying future. |
| `get_mut(&mut self) -> &mut T` | Gets a mutable reference to the underlying future. |
| `into_inner(self) -> T` | Consumes this timeout, returning the underlying future. |

### Instant

A measurement of a monotonically nondecreasing clock. This is a Tokio-aware wrapper around `std::time::Instant`.

It is opaque and useful only with `Duration`. It allows measuring the duration between two instants or comparing them.

#### Methods

| Method | Description |
|---|---|
| `now() -> Instant` | Returns an instant corresponding to "now". |
| `elapsed(&self) -> Duration` | Returns the amount of time elapsed since this instant was created. |
| `duration_since(&self, earlier: Instant) -> Duration` | Returns the amount of time elapsed from another instant to this one. |
| `checked_add(&self, duration: Duration) -> Option<Instant>` | Adds a duration to the instant, returning `None` on overflow. |
| `checked_sub(&self, duration: Duration) -> Option<Instant>` | Subtracts a duration from the instant, returning `None` if it would go before the earliest representable time. |

## Enums

### MissedTickBehavior

Defines the behavior of an `Interval` when it misses a tick. A tick is missed if too much time is spent without calling `Interval::tick()`.

#### Burst

This is the default behavior. Ticks as fast as possible until caught up. The yielded `Instant`s are the same as they would have been if no ticks were missed.

```text
Expected ticks: |     1     |     2     |     3     |     4     |     5     |     6     |
Actual ticks:   | work -----|          delay          | work | work | work -| work -----|
```

#### Delay

Tick at multiples of `period` from when `tick` was last called, rather than from the original start time. This effectively resets the interval schedule after a long delay.

```text
Expected ticks: |     1     |     2     |     3     |     4     |     5     |     6     |
Actual ticks:   | work -----|          delay          | work -----| work -----| work -----|
```

#### Skip

Skips any missed ticks and schedules the next tick on the next multiple of `period` from the original start time.

```text
Expected ticks: |     1     |     2     |     3     |     4     |     5     |     6     |
Actual ticks:   | work -----|          delay          | work ---| work -----| work -----|
```

#### Example of changing behavior

```rust
use tokio::time::{interval, Duration, MissedTickBehavior};

# async fn task_that_takes_more_than_50_millis() { tokio::time::sleep(Duration::from_millis(60)).await; }

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let mut interval = interval(Duration::from_millis(50));
    interval.set_missed_tick_behavior(MissedTickBehavior::Delay);

    // The first tick is immediate
    interval.tick().await;

    task_that_takes_more_than_50_millis().await;
    // The Interval has missed a tick

    // This resolves immediately because the deadline has passed.
    interval.tick().await;

    // With Delay behavior, the next tick is scheduled 50ms from the *previous*
    // tick's completion, not from its originally scheduled time.
    interval.tick().await;
}
```
