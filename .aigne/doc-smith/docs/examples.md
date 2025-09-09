# Examples

This section provides a collection of working code examples to demonstrate various Tokio features and use cases. These examples are designed to be practical and easy to adapt for your own projects.

## TCP Echo Server

A classic networking example: a simple TCP server that accepts incoming connections and echoes back any data it receives. This demonstrates core Tokio concepts like asynchronous I/O and task spawning.

First, ensure your `Cargo.toml` includes Tokio with the necessary features:

```toml Cargo.toml icon=mdi:file-document-outline
tokio = { version = "1", features = ["full"] }
```

Here is the complete server implementation:

```rust TCP Echo Server icon=logos:rust
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

## Further Exploration

For more complex and varied use cases, the Tokio project provides additional resources.

<x-cards>
  <x-card data-title="Mini-Redis Project" data-icon="lucide:database" data-href="https://github.com/tokio-rs/mini-redis/" data-cta="View on GitHub">
    For a larger, "real-world" example, explore the mini-redis repository. It's an incomplete, asynchronous Redis client and server built with Tokio, showcasing how various components work together in a larger application.
  </x-card>
  <x-card data-title="Official Examples Directory" data-icon="lucide:folder-git-2" data-href="https://github.com/tokio-rs/tokio/tree/master/examples" data-cta="Browse Examples">
    The main Tokio repository contains a directory with many more examples. These cover a wide range of functionalities, including channels, filesystem operations, timers, and various networking scenarios.
  </x-card>
</x-cards>

After reviewing these examples, you might want to dive deeper into the [Core Concepts](./concepts.md) or consult the comprehensive [API Reference](./api.md) for specific details.