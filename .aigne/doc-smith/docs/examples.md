# Examples

This section provides a collection of runnable examples to help you understand how to use Tokio's various features. These examples are designed to be copied and run with minimal setup. For a more comprehensive set of examples, you can explore the [Tokio repository on GitHub](https://github.com/tokio-rs/tokio/tree/master/examples).

## TCP Echo Server

Here is a basic TCP echo server that listens for incoming connections and sends back any data it receives. This is a classic example to demonstrate asynchronous I/O and task management.

### Setup

First, add the necessary dependencies to your `Cargo.toml` file. The `full` feature flag enables all public Tokio APIs, which is convenient for getting started.

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

### Code

Now, you can use the following code in your `main.rs` file:

```rust,no_run
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

This code sets up a listener on port 8080. For each incoming connection, it spawns a new asynchronous task to handle communication. The task reads data into a buffer and writes it back to the same socket until the connection is closed.

## Handling Blocking Operations

Asynchronous tasks should not perform blocking operations directly, as this can halt the progress of other tasks on the same thread. For blocking or CPU-intensive work, use the `spawn_blocking` function to move the work to a dedicated thread pool.

```rust,no_run
#[tokio::main]
async fn main() {
    // This is running on a core thread.

    let blocking_task = tokio::task::spawn_blocking(|| {
        // This is running on a blocking thread.
        // Blocking here is ok.
        // For example, a computationally expensive task or a synchronous file read.
        "done"
    });

    // We can wait for the blocking task like this:
    // If the blocking task panics, the unwrap below will propagate the
    // panic.
    let result = blocking_task.await.unwrap();
    println!("Blocking task finished with result: {}", result);
}
```

The `spawn_blocking` function takes a closure and executes it on a separate thread pool managed by the Tokio runtime. This prevents the main async scheduler from being blocked. The `await` on the returned `JoinHandle` allows the asynchronous task to wait for the blocking operation to complete without halting the thread.

## Further Exploration

For more advanced and real-world use cases, the following resources provide a wealth of examples.

<x-cards data-columns="2">
  <x-card data-title="Official Tokio Examples" data-icon="lucide:github" data-href="https://github.com/tokio-rs/tokio/tree/master/examples" data-cta="View on GitHub">
    A comprehensive collection of examples in the official Tokio repository, covering various modules and features.
  </x-card>
  <x-card data-title="Mini-Redis" data-icon="lucide:database" data-href="https://github.com/tokio-rs/mini-redis" data-cta="View on GitHub">
    A larger, "real-world" example of a client-server application built with Tokio, demonstrating more complex application structure.
  </x-card>
</x-cards>