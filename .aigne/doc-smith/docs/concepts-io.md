# Asynchronous I/O

Tokio provides a comprehensive suite of non-blocking I/O primitives for building high-performance network applications, working with the filesystem, and managing inter-process communication. These utilities are designed to be asynchronous, meaning they integrate with the Tokio runtime to prevent blocking threads, allowing a small number of threads to handle many concurrent operations.

This section covers the fundamental concepts behind Tokio's I/O operations. For detailed API information, please refer to the [API Reference](./api.md).

## The `tokio::io` Module: Core Primitives

The foundation of Tokio's I/O is the `tokio::io` module, which is the asynchronous equivalent of `std::io`. It defines two fundamental traits:

- **`AsyncRead`**: An asynchronous version of `std::io::Read` for reading bytes from a source.
- **`AsyncWrite`**: An asynchronous version of `std::io::Write` for writing bytes to a destination.

When an operation on an `AsyncRead` or `AsyncWrite` type would need to wait for data, it yields control back to the Tokio scheduler instead of blocking the thread. This allows other tasks to run while the I/O operation is pending.

Utility methods for these traits are provided through the `AsyncReadExt` and `AsyncWriteExt` extension traits, which are automatically available for any type that implements `AsyncRead` or `AsyncWrite`.

Here is an example of reading up to 10 bytes from a file:

```rust
use tokio::io::{self, AsyncReadExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = [0; 10];

    // read up to 10 bytes
    let n = f.read(&mut buffer).await?;

    println!("The bytes: {:?}", &buffer[..n]);
    Ok(())
}
```

### Buffered Readers and Writers

To improve efficiency and reduce the number of system calls, Tokio provides buffered I/O types similar to the standard library. The `BufReader` and `BufWriter` structs wrap any async reader or writer to provide in-memory buffering.

`BufReader` adds convenient methods like `read_line` for reading data in chunks:

```rust
use tokio::io::{self, BufReader, AsyncBufReadExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::open("foo.txt").await?;
    let mut reader = BufReader::new(f);
    let mut buffer = String::new();

    // read a line into buffer
    reader.read_line(&mut buffer).await?;

    println!("{}", buffer);
    Ok(())
}
```

When using `BufWriter`, it is important to call the `flush()` method to ensure that all buffered data is written to the underlying writer.

```rust
use tokio::io::{self, BufWriter, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let f = File::create("foo.txt").await?;
    {
        let mut writer = BufWriter::new(f);

        // Write a byte to the buffer.
        writer.write(&[42u8]).await?;

        // Flush the buffer before it goes out of scope.
        writer.flush().await?;

    } // The buffer is discarded on drop unless flushed.

    Ok(())
}
```

## Networking with `tokio::net`

The `tokio::net` module provides asynchronous TCP, UDP, and Unix Domain Socket APIs. These types integrate with the Tokio runtime to handle network I/O without blocking.

Key components include:
- **`TcpListener` & `TcpStream`**: For building TCP clients and servers.
- **`UdpSocket`**: For UDP communication.
- **`UnixListener` & `UnixStream`**: For stream-based communication over Unix sockets (on Unix-like systems).

Below is an example of a simple TCP echo server that accepts connections and writes back any data it receives.

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

            // In a loop, read data from the socket and write it back.
            loop {
                let n = match socket.read(&mut buf).await {
                    Ok(0) => return, // socket closed
                    Ok(n) => n,
                    Err(e) => {
                        eprintln!("failed to read from socket; err = {:?}", e);
                        return;
                    }
                };

                if let Err(e) = socket.write_all(&buf[0..n]).await {
                    eprintln!("failed to write to socket; err = {:?}", e);
                    return;
                }
            }
        });
    }
}
```

## Filesystem Operations with `tokio::fs`

The `tokio::fs` module provides asynchronous APIs for file and directory manipulation. It's important to understand that most operating systems do not offer true asynchronous filesystem APIs. To overcome this, Tokio uses its blocking thread pool (`spawn_blocking`) to execute filesystem operations in the background, preventing them from blocking the main runtime threads.

This module is intended for ordinary files. For special files like named pipes, it is better to use dedicated types like `tokio::net::unix::pipe`.

Here's how you can read the entire contents of a file into a string:

```rust
async fn read_file_contents() -> std::io::Result<()> {
    let contents = tokio::fs::read_to_string("my_file.txt").await?;
    println!("File has {} lines.", contents.lines().count());
    Ok(())
}
```

## Managing Processes with `tokio::process`

Tokio allows you to manage child processes asynchronously through the `tokio::process` module. The `Command` struct mimics the API of `std::process::Command` but provides asynchronous methods like `spawn`, `status`, and `output`.

This example spawns the `echo` command and captures its output:

```rust
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let output = Command::new("echo")
        .arg("hello")
        .arg("world")
        .output()
        .await?;

    assert!(output.status.success());
    assert_eq!(output.stdout, b"hello world\n");

    Ok(())
}
```

## Next Steps

You've learned about the core concepts of asynchronous I/O in Tokio. To dive deeper into the specific APIs, explore the following sections:

<x-cards data-columns="2">
  <x-card data-title="I/O API Reference" data-icon="lucide:file-text" data-href="/api/io">
    Detailed documentation for asynchronous I/O traits, helpers, and type definitions.
  </x-card>
  <x-card data-title="Networking API Reference" data-icon="lucide:globe" data-href="/api/net">
    API documentation for TCP, UDP, and Unix socket types for network communication.
  </x-card>
  <x-card data-title="Filesystem API Reference" data-icon="lucide:folder" data-href="/api/fs">
    API documentation for asynchronous file and filesystem manipulation operations.
  </x-card>
  <x-card data-title="Processes API Reference" data-icon="lucide:terminal-square" data-href="/api/process">
    API documentation for asynchronous process management.
  </x-card>
</x-cards>