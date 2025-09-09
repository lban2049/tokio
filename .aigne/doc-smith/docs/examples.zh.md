# 示例

本节提供了一系列可运行的代码示例，以演示 Tokio 的各种功能和用例。这些示例旨在做到实用，并易于在您自己的项目中进行调整。

## TCP 回声服务器

一个经典的联网示例：一个简单的 TCP 服务器，它接受传入的连接，并回显其接收到的任何数据。该示例演示了 Tokio 的核心概念，如异步 I/O 和任务生成。

首先，请确保您的 `Cargo.toml` 文件中包含了具有必要功能的 Tokio：

```toml Cargo.toml icon=mdi:file-document-outline
tokio = { version = "1", features = ["full"] }
```

以下是完整的服务器实现：

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

            // 在一个循环中，从套接字读取数据并将数据写回。
            loop {
                let n = match socket.read(&mut buf).await {
                    // 套接字已关闭
                    Ok(0) => return,
                    Ok(n) => n,
                    Err(e) => {
                        eprintln!("failed to read from socket; err = {:?}", e);
                        return;
                    }
                };

                // 将数据写回
                if let Err(e) = socket.write_all(&buf[0..n]).await {
                    eprintln!("failed to write to socket; err = {:?}", e);
                    return;
                }
            }
        });
    }
}
```

## 进一步探索

对于更复杂和多样化的用例，Tokio 项目提供了额外的资源。

<x-cards>
  <x-card data-title="Mini-Redis 项目" data-icon="lucide:database" data-href="https://github.com/tokio-rs/mini-redis/" data-cta="在 GitHub 上查看">
    要查看一个更大、更“真实”的示例，请探索 mini-redis 仓库。这是一个使用 Tokio 构建的、尚未完成的异步 Redis 客户端和服务器，它展示了各种组件在大型应用中如何协同工作。
  </x-card>
  <x-card data-title="官方示例目录" data-icon="lucide:folder-git-2" data-href="https://github.com/tokio-rs/tokio/tree/master/examples" data-cta="浏览示例">
    Tokio 主仓库中有一个目录，包含了更多示例。这些示例涵盖了广泛的功能，包括通道、文件系统操作、计时器以及各种网络场景。
  </x-card>
</x-cards>

查看完这些示例后，您可能希望深入了解 [核心概念](./concepts.md)，或查阅全面的 [API 参考](./api.md) 以获取具体细节。