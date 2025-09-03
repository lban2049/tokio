# Getting Started

This guide will walk you through setting up a new project with Tokio and building a simple TCP echo server. By the end, you'll have a running asynchronous application.

## 1. Create a New Project

First, let's create a new Rust project using Cargo.

```bash
cargo new my-tokio-app
cd my-tokio-app
```

## 2. Add Tokio as a Dependency

Tokio is modular, with different features available behind feature flags. To get started easily, we'll enable all features using the `full` flag.

Add the following line to your `Cargo.toml` file:

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

This ensures that all the APIs you might need while building your application are readily available.

## 3. Write the Code

Now, let's write our TCP echo server. Open `src/main.rs` and replace its contents with the following code:

```rust,no_run
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Bind a listener to the address
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    println!("Server listening on port 8080");

    loop {
        // The second item contains the IP and port of the new connection.
        let (mut socket, _) = listener.accept().await?;

        // Spawn a new task to handle each connection.
        tokio::spawn(async move {
            let mut buf = [0; 1024];

            // In a loop, read data from the socket and write it back.
            loop {
                let n = match socket.read(&mut buf).await {
                    // Socket closed
                    Ok(0) => return,
                    Ok(n) => n,
                    Err(e) => {
                        eprintln!("failed to read from socket; err = {:?}", e);
                        return;
                    }
                };

                // Write the data back to the socket
                if let Err(e) = socket.write_all(&buf[0..n]).await {
                    eprintln!("failed to write to socket; err = {:?}", e);
                    return;
                }
            }
        });
    }
}
```

Let's break down what's happening:

-   `#[tokio::main]`: This is a macro that transforms the `async fn main()` into a synchronous `main` function that initializes a Tokio runtime and executes the asynchronous code.
-   `TcpListener::bind("...").await?`: This line creates a TCP listener that binds to the specified address. The `.await` keyword is used because binding is an asynchronous operation.
-   `listener.accept().await?`: This asynchronously waits for a new incoming connection. When a connection is established, it returns a tuple containing a socket and the address of the peer.
-   `tokio::spawn(async move { ... })`: This creates a new asynchronous task. The connection is moved into this task and handled concurrently, allowing the main loop to continue accepting new connections without waiting for the previous one to finish.
-   `socket.read(&mut buf).await`: This reads data from the socket into a buffer. It returns the number of bytes read. If it returns `Ok(0)`, the connection has been closed by the client.
-   `socket.write_all(&buf[0..n]).await`: This writes the data from the buffer back to the socket, echoing it to the client.

## 4. Run the Application

Now you can run the server:

```bash
cargo run
```

You should see the output `Server listening on port 8080`.

To test it, open a new terminal window and use a tool like `telnet` or `netcat` to connect to the server:

```bash
telnet 127.0.0.1 8080
```

Anything you type into the `telnet` session will be echoed back by the server. To close the connection, press `Ctrl+]` in `telnet` and type `quit`.

## Next Steps

Congratulations! You've successfully built your first asynchronous application with Tokio. 

To better understand the components you just used and the principles behind Tokio, it's a good idea to dive into the core concepts.

<x-card data-title="Core Concepts" data-icon="lucide:book-open" data-href="/concepts" data-cta="Learn More">
  Explore the fundamental concepts and components that make up the Tokio runtime, providing a solid foundation for advanced usage.
</x-card>