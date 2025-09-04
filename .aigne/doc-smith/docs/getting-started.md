# Getting Started

This guide will walk you through setting up a new project with Tokio and building a simple, asynchronous TCP echo server. By the end, you'll have a working application that can handle multiple concurrent connections.

## 1. Add Tokio as a dependency

To begin, create a new Rust project and add Tokio to your dependencies. The easiest way to get started is to enable all features via the `full` feature flag in your `Cargo.toml` file. This ensures that all the APIs you might need are available as you build your application.

**Cargo.toml**
```toml
tokio = { version = "1", features = ["full"] }
```

## 2. Write an asynchronous TCP echo server

Next, let's write the code for our server. This application will listen for incoming TCP connections on `127.0.0.1:8080`. For each connection, it will read data from the socket and write the same data back to the client.

**main.rs**
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

## 3. Run the application

With the code in place, you can run the server using Cargo:

```sh
cargo run
```

To test the server, you can connect to it using a tool like `netcat` or `telnet`. Open a new terminal window and run:

```sh
telnet 127.0.0.1 8080
```

Anything you type into the `telnet` session will be echoed back by the server.

## 4. How it works

Let's briefly break down what the code is doing:

-   `#[tokio::main]`: This is a macro that transforms the `async fn main()` into a synchronous `main` function that initializes a Tokio runtime and executes the asynchronous code.
-   `TcpListener::bind("127.0.0.1:8080").await?`: We create a `TcpListener` to listen for incoming connections. This is an asynchronous operation, so we use `.await` to wait for it to complete.
-   `listener.accept().await?`: The `accept` method waits for a new connection. When a connection is established, it returns a new `TcpStream` (our `socket`) and the address of the peer. The `loop` ensures the server continues to accept new connections indefinitely.
-   `tokio::spawn(async move { ... })`: For each incoming connection, we spawn a new asynchronous task. This allows the server to process multiple connections concurrently. The main task can immediately go back to accepting new connections while the spawned task handles its specific connection.
-   `socket.read(&mut buf).await` and `socket.write_all(...).await`: These are the asynchronous I/O operations. They read data from the socket into a buffer and write the contents of the buffer back to the socket. The `.await` keyword pauses the task until the operation is complete without blocking the entire thread, allowing other tasks to run.

## Next Steps

You've successfully built your first asynchronous application with Tokio! To gain a deeper understanding of the concepts you've just used, we recommend exploring the [Core Concepts](./concepts.md) documentation.