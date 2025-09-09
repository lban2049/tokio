# Getting Started

This guide will walk you through setting up your first Tokio application. We'll start by creating a new Rust project, adding Tokio as a dependency, and then building a simple TCP echo server that sends back any data it receives.

### Setting up the Project

First, let's create a new Rust project using Cargo:

```bash Create a new project icon=lucide:terminal
cargo new my-tokio-app
cd my-tokio-app
```

Next, add the `tokio` crate as a dependency in your `Cargo.toml` file. We'll enable all features using the `full` feature flag. This is the easiest way to get started and ensures all the APIs you'll need are available.

```toml Cargo.toml icon=lucide:file-text
[dependencies]
tokio = { version = "1", features = ["full"] }
```

### Writing the Echo Server

Now, replace the contents of your `main.rs` file with the following code. This program will set up a server that listens on `127.0.0.1:8080`, and for each incoming connection, it will read data and write the same data back to the client.

```rust main.rs icon=logos:rust
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

Let's break down what's happening here:

-   `#[tokio::main]`: This is a macro that transforms the `async fn main()` into a synchronous `main()` function that initializes a Tokio runtime and executes the asynchronous code.
-   `TcpListener::bind("127.0.0.1:8080").await?`: We create a TCP listener bound to the specified address. The `.await` keyword is used because binding is an asynchronous operation.
-   `loop { ... }`: The server enters a loop to continuously accept new connections.
-   `listener.accept().await?`: This asynchronously waits for a new inbound connection. When one is established, it returns a tuple containing the socket and the address of the peer.
-   `tokio::spawn(async move { ... });`: For each connection, a new task is spawned. This allows the server to handle multiple connections concurrently. The `move` keyword transfers ownership of the `socket` to the new task.
-   `socket.read(&mut buf).await`: Inside the task, we read data from the socket into a buffer. This is another asynchronous operation, so we `.await` it.
-   `socket.write_all(&buf[0..n]).await`: We write the data we just read back to the socket, effectively "echoing" it.

### Running the Server

With the code in place, you can run the server from your terminal:

```bash Run the application icon=lucide:terminal
cargo run
```

The server is now running. To test it, open a new terminal window and use a tool like `netcat` or `telnet` to connect to it.

```bash Test with netcat icon=lucide:terminal
nc 127.0.0.1 8080
```

Once connected, type any message, press Enter, and you should see the same message echoed back to you. To stop the server, go back to the first terminal and press `Ctrl+C`.

Congratulations! You've just built your first asynchronous application with Tokio.

### Next Steps

Now that you have a basic application running, you're ready to learn more about the fundamental building blocks of Tokio.

<x-card data-title="Core Concepts" data-icon="lucide:puzzle" data-href="/concepts" data-cta="Explore Concepts">
Dive deeper into tasks, I/O, state management, and the runtime itself to understand how Tokio works under the hood.
</x-card>
