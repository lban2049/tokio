# Getting Started

This guide will get you from zero to a running Tokio application in just a few minutes. We'll cover setting up your project and building a simple TCP echo server that handles multiple connections concurrently.

## Project Setup

First, you'll need to add the `tokio` crate as a dependency in your `Cargo.toml` file. 

When you're writing an application, we recommend enabling all features via the `full` flag. This ensures you have access to all of Tokio's APIs without running into roadblocks while you're building.

Add the following to your `Cargo.toml`:

```toml Cargo.toml icon=logos:rust
[dependencies]
tokio = { version = "1", features = ["full"] }
```

## A Basic TCP Echo Server

Let's build a simple server that accepts incoming TCP connections and sends back any data it receives. This is a classic example that demonstrates the core features of Tokio: asynchronous I/O and concurrent task management.

Create a new file `src/main.rs` and add the following code:

```rust main.rs icon=logos:rust
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

## Code Walkthrough

Let's break down what's happening in the code:

1.  **`#[tokio::main]`**: This is a macro that sets up the Tokio runtime. It transforms the `async fn main()` into a synchronous `main` function that initializes the runtime and executes the asynchronous code within it.

2.  **`TcpListener::bind("127.0.0.1:8080").await?`**: We create a `TcpListener` and bind it to port 8080. This is an asynchronous operation, so we use `.await` to wait for it to complete.

3.  **`listener.accept().await?`**: The `loop` continuously calls `accept()`. Each time, `accept()` waits asynchronously for a new incoming connection. When a client connects, it returns a new `TcpStream` (our `socket`) and the client's address.

4.  **`tokio::spawn(async move { ... })`**: To handle multiple clients concurrently, we spawn a new asynchronous task for each incoming connection. The `tokio::spawn` function takes an `async` block and runs it on the Tokio runtime without blocking the main loop. This allows the `loop` to immediately go back to waiting for the next connection.

5.  **`socket.read(...)` and `socket.write_all(...)`**: Inside the spawned task, we repeatedly read data from the client into a buffer and then write that same data back to the client. This is the 'echo' logic. Both are asynchronous operations, so they are marked with `.await`.

## Running the Server

Now, you can run the application from your terminal:

```sh
cargo run
```

The server will start and listen for connections on `127.0.0.1:8080`.

To test it, open a new terminal window and use a tool like `telnet` or `netcat` to connect:

```sh
telnet 127.0.0.1 8080
```

Once connected, anything you type will be echoed back to you by the server. You can even open multiple terminal windows and connect simultaneously to see the concurrent handling in action.

## Next Steps

Congratulations! You've successfully built and run your first asynchronous application with Tokio. You've seen how to set up a project, perform non-blocking I/O, and handle concurrent operations using tasks.

To dive deeper into how Tokio manages these concurrent operations, head over to the [Tasks & Scheduling](./tasks-scheduling.md) guide.