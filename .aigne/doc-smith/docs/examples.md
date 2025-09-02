# Examples

This section provides a collection of working code examples to help you understand how to use various Tokio features. These examples are designed to be practical and easy to follow, demonstrating common use cases.

For a more comprehensive set of examples, you can explore the [official Tokio examples directory on GitHub](https://github.com/tokio-rs/tokio/tree/master/examples).

## TCP Echo Server

A simple yet complete TCP echo server that listens for incoming connections and echoes back any data it receives. This is a great starting point for building network applications.

First, ensure your `Cargo.toml` is configured to include the necessary Tokio features. The `full` feature is recommended for getting started easily.

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

Here is the server implementation:

```rust
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = [0; 1024];

            // In a loop, read data from the socket and write the data back.
            loop {
                let n = match socket.read(&mut buf).await {
                    // socket closed
                    Ok(0) => return,
                    Ok(n) => n,
                    Err(e) => {
                        eprintln!("failed to read from socket; err = {:?}", e);
                        return;
                    }
                };

                // Write the data back
                if let Err(e) = socket.write_all(&buf[0..n]).await {
                    eprintln!("failed to write to socket; err = {:?}", e);
                    return;
                }
            }
        });
    }
}
```

### How It Works

1.  **`TcpListener::bind(...)`**: Binds a new TCP listener to the specified address. The `.await` pauses execution until the listener is successfully bound.
2.  **`listener.accept().await`**: In an infinite loop, the server waits for new incoming connections. Execution is paused until a connection is established.
3.  **`tokio::spawn(...)`**: For each new connection, a new asynchronous task is spawned. This allows the server to handle multiple clients concurrently without blocking the main loop.
4.  **`socket.read(...)` and `socket.write_all(...)`**: Inside the spawned task, the server repeatedly reads data from the socket into a buffer and writes the exact same data back to the socket, effectively "echoing" it.

## Handling Blocking Operations

Tokio's cooperative scheduler requires tasks to yield control so other tasks can run. However, some operations are inherently blocking, such as heavy CPU computations or traditional, synchronous file I/O. To handle these without stalling the runtime, you should use `tokio::task::spawn_blocking`.

This function moves the blocking operation to a dedicated thread pool, allowing the main runtime to continue processing other asynchronous tasks.

```rust
#[tokio::main]
async fn main() {
    // This is running on a core thread.

    let blocking_task = tokio::task::spawn_blocking(|| {
        // This is running on a blocking thread.
        // Blocking here is ok.
        // For example, a heavy computation.
        std::thread::sleep(std::time::Duration::from_secs(1));
        "done"
    });

    // We can wait for the blocking task like this:
    // If the blocking task panics, the unwrap below will propagate the
    // panic.
    let result = blocking_task.await.unwrap();
    println!("Blocking task finished: {}", result);
}
```

### How It Works

1.  The closure passed to `spawn_blocking` is executed on a separate thread from Tokio's blocking thread pool.
2.  This prevents the potentially long-running operation from halting the progress of other asynchronous tasks on the main scheduler.
3.  The main task can `.await` the `JoinHandle` returned by `spawn_blocking` to receive the result once the computation is complete, without blocking the executor.

## More Advanced Examples

For larger, real-world examples that demonstrate how to structure a full application with Tokio, check out these resources.

<x-cards data-columns="2">
  <x-card data-title="Mini-Redis" data-icon="lucide:database" data-href="https://github.com/tokio-rs/mini-redis/">
    A complete, asynchronous Redis client and server. It's an excellent example of a real-world application built with Tokio, showcasing channels, shared state, and graceful shutdown.
  </x-card>
  <x-card data-title="Official Examples" data-icon="lucide:book-open" data-href="https://github.com/tokio-rs/tokio/tree/master/examples">
    The official Tokio repository contains a wide variety of smaller examples, each focusing on a specific feature like networking, channels, or timers.
  </x-card>
</x-cards>