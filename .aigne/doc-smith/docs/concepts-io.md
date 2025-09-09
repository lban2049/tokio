# Asynchronous I/O

At the heart of Tokio is its non-blocking, asynchronous I/O model. This is the asynchronous version of `std::io`, designed to prevent your application from blocking threads while waiting for I/O operations to complete. Instead of waiting, tasks can yield control back to the Tokio scheduler, allowing it to run other tasks. This is the key to building applications that can handle a massive number of concurrent connections with only a few OS threads.

Tokio provides a comprehensive suite of tools for various I/O needs, from networking to filesystem operations and inter-process communication.

## The Core I/O Traits

Just like the standard library, Tokio's I/O functionality is built around a pair of core traits: `AsyncRead` and `AsyncWrite`. These are the asynchronous counterparts to `std::io::Read` and `std::io::Write`.

- **`AsyncRead`**: A trait for types that can be read from asynchronously.
- **`AsyncWrite`**: A trait for types that can be written to asynchronously.

Unlike their synchronous counterparts, these traits only contain the essential methods for asynchronous operations. A rich set of utility methods (like `read_to_string`, `write_all`, etc.) are provided by the `AsyncReadExt` and `AsyncWriteExt` extension traits, which are automatically available for any type that implements `AsyncRead` or `AsyncWrite`.

For example, here's how you can read up to 10 bytes from a file:

```rust Rust code for reading from a file icon=logos:rust
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

### Buffered I/O

To improve efficiency and reduce the number of system calls, Tokio provides buffered readers and writers, similar to the standard library. The `BufReader` and `BufWriter` structs wrap any `AsyncRead` or `AsyncWrite` type, respectively, to buffer operations. `BufReader` also enables more convenient methods, like reading line by line.

```rust Reading a line from a file icon=logos:rust
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

When using `BufWriter`, remember to call `.flush().await` to ensure all buffered data is written to the underlying writer.

## Types of Asynchronous I/O

Tokio provides a set of modules for different kinds of I/O and asynchronous interactions with the operating system.

<x-cards data-columns="2">
  <x-card data-title="Networking" data-icon="lucide:network" data-href="/api/net">
    Non-blocking TCP, UDP, and Unix Domain Sockets for building high-performance network services.
  </x-card>
  <x-card data-title="Filesystem" data-icon="lucide:folder" data-href="/api/fs">
    Asynchronous APIs for file and filesystem manipulation, such as reading, writing, and creating directories.
  </x-card>
  <x-card data-title="Processes" data-icon="lucide:terminal" data-href="/api/process">
    Tools for spawning and managing child processes asynchronously, including capturing their standard I/O streams.
  </x-card>
  <x-card data-title="Signals" data-icon="lucide:radio-tower" data-href="/api/signal">
    Asynchronous handling of Unix and Windows OS signals, allowing for graceful shutdown and other signal-based logic.
  </x-card>
</x-cards>

### Networking Example: TCP Echo Server

Here is a complete example of a simple TCP echo server that listens for incoming connections and writes any data it receives back to the client.

```rust A simple TCP echo server icon=logos:rust
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

### Filesystem: The Blocking Reality

It's important to understand that most operating systems do not provide true asynchronous file system APIs. To work around this, Tokio's `fs` module uses the `spawn_blocking` function internally. This means that file operations are executed on a dedicated thread pool for blocking tasks, preventing them from blocking the main asynchronous tasks on the runtime's core threads.

While this provides an asynchronous API, it carries performance implications. For optimal performance, it's recommended to batch file operations into as few calls as possible, for example by using `tokio::fs::write` for the entire file at once, or wrapping a `File` in a `BufWriter`.

### Process Management

Tokio allows you to manage child processes asynchronously using `tokio::process::Command`. It provides a familiar builder API, similar to `std::process::Command`, but its execution methods are `async`.

Here is an example that spawns the `echo` command and captures its output:

```rust Spawning a command and capturing output icon=logos:rust
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Use `output` which returns a future instead of
    // a `Child` immediately.
    let output = Command::new("echo").arg("hello").arg("world")
                        .output()
                        .await?;

    assert!(output.status.success());
    assert_eq!(output.stdout, b"hello world\n");
    Ok(())
}
```

### Standard I/O

Tokio also provides asynchronous APIs for standard input, output, and error via the `tokio::io::stdin`, `stdout`, and `stderr` functions. These are asynchronous versions of the standard library's handles and implement `AsyncRead` and `AsyncWrite`. Note that these functions **must** be called from within the context of a Tokio runtime.

## Next Steps

With a solid understanding of asynchronous I/O, you are ready to explore how to manage state and communication between tasks.

<x-card data-title="Synchronization" data-icon="lucide:link" data-href="/concepts/synchronization" data-cta="Learn about Synchronization">
  Explore Tokio's synchronization primitives like channels and mutexes for coordinating asynchronous tasks.
</x-card>
