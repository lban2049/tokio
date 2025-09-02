# Getting Started

This guide provides a step-by-step walkthrough to set up a new Tokio project and build a simple, working TCP echo server. By the end, you will have a running asynchronous application.

## 1. Setting Up Your Project

First, you'll need a new Rust project. If you don't have one, you can create it with Cargo:

```bash
cargo new my-tokio-app
cd my-tokio-app
```

Next, add the `tokio` crate as a dependency in your `Cargo.toml` file.

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

We enable the `full` feature flag to include all public APIs. This is recommended for applications to ensure you have access to all the necessary tools without needing to specify individual features as you build.

## 2. Writing the Echo Server

Now, replace the content of `src/main.rs` with the following code to create the TCP echo server.

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;

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

### Code Breakdown

- **`#[tokio::main]`**: This is a macro that transforms the `async fn main` into a synchronous `main` function that initializes a Tokio runtime and executes the asynchronous code.
- **`TcpListener::bind("127.0.0.1:8080").await?`**: This line creates a `TcpListener` bound to the specified address. It waits asynchronously for the listener to be successfully created.
- **`listener.accept().await?`**: The `accept` method waits for a new incoming connection. When a connection is established, it returns a new `TcpSocket` and the address of the peer.
- **`tokio::spawn(async move { ... })`**: This function spawns a new asynchronous task. The server handles each incoming connection concurrently in its own task, allowing it to manage multiple clients at once. The `move` keyword transfers ownership of the `socket` to the new task.
- **`socket.read(&mut buf).await`**: This reads data from the socket into the buffer `buf`. The `.await` pauses the task until data is available.
- **`socket.write_all(&buf[0..n]).await`**: This writes the data that was just read from the buffer back to the socket, echoing it to the client.

## 3. Running the Application

With the code in place, you can run the server using Cargo:

```bash
cargo run
```

The server is now running and waiting for incoming connections. To test it, open a new terminal window and use a tool like `netcat` or `telnet` to connect to it:

```bash
telnet 127.0.0.1 8080
```

Once connected, type any message and press Enter. The server will echo the message back to you. To stop the server, you can use `Ctrl+C` in the terminal where it's running.

## Next Steps

Congratulations! You've successfully built your first asynchronous application with Tokio. To continue your journey, you can explore the core concepts that power Tokio or browse more examples.

<x-cards data-columns="2">
  <x-card data-title="Core Concepts" data-icon="lucide:puzzle" data-href="/concepts">
    Dive deeper into the fundamental components of Tokio, including tasks, asynchronous I/O, synchronization, and the runtime.
  </x-card>
  <x-card data-title="Examples" data-icon="lucide:lightbulb" data-href="/examples">
    Explore a collection of working code examples that demonstrate various Tokio features and common use cases.
  </x-card>
</x-cards>