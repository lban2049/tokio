# Contributing & Support

Welcome to the Tokio community! Whether you're looking for help, wanting to report a bug, or eager to contribute code, this guide provides the resources you need to get involved with the project.

## Getting Help

If you have questions about using Tokio, we encourage you to seek answers through the following resources. Many common questions are already addressed in our official documentation.

- **[Guides](https://tokio.rs/tokio/tutorial)**: The official tutorials are the best place to start learning about Tokio and its concepts.
- **[API Documentation](https://docs.rs/tokio/latest/tokio)**: For detailed information on specific functions, structs, and modules.

If you can't find an answer in the documentation, our vibrant community is ready to help:

- **[Discord Server](https://discord.gg/tokio)**: Join our active Discord for real-time chat with other users and maintainers.
- **[GitHub Discussions](https://github.com/tokio-rs/tokio/discussions)**: Ask questions, share ideas, and engage in longer-form discussions with the community.

## Contributing to Tokio

:balloon: Thanks for your help improving the project! We are so happy to have you! We have a [contributing guide](https://github.com/tokio-rs/tokio/blob/master/CONTRIBUTING.md) to help you get involved in the Tokio project. Please read it to learn about our development process, how to propose bugfixes and improvements, and how to build and test your changes.

### License for Contributions

This project is licensed under the MIT license. Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in Tokio by you shall be licensed as MIT, without any additional terms or conditions.

## Project Policies

To ensure stability and a predictable development cycle, the Tokio project adheres to the following policies.

### Supported Rust Versions (MSRV)

Tokio maintains a rolling Minimum Supported Rust Version (MSRV) policy. When we increase the MSRV, the new Rust version must have been released at least six months prior. This policy applies to minor releases; we do not increase the MSRV in patch releases.

**The current MSRV is 1.70.**

Here is the MSRV history for past minor releases:

| Tokio Version | MSRV      |
|---------------|-----------|
| 1.39 to now   | Rust 1.70 |
| 1.30 to 1.38  | Rust 1.63 |
| 1.27 to 1.29  | Rust 1.56 |
| 1.17 to 1.26  | Rust 1.49 |
| 1.15 to 1.16  | Rust 1.46 |
| 1.0 to 1.14   | Rust 1.45 |

### Release Schedule

Tokio doesn't follow a fixed release schedule, but we typically make one minor release each month. We make patch releases for bugfixes as necessary.

### Long-Term Support (LTS) & Bug Patching Policy

We designate certain minor releases as Long-Term Support (LTS) versions. Critical bugfixes will be backported to all active LTS releases. Each LTS release will receive backported fixes for at least one year.

If you need to depend on a fixed minor release in your project, we strongly recommend using an LTS version.

**Current LTS Releases:**

| Version | Supported Until | MSRV      |
|---------|-----------------|-----------|
| `1.43.x`  | March 2026      | Rust 1.70 |
| `1.47.x`  | September 2026  | Rust 1.70 |

To use a specific LTS minor version, you can specify it in your `Cargo.toml` with a tilde, which will match the newest patch release for that minor version.

```toml Cargo.toml icon=logos:rust
tokio = { version = "~1.43", features = ["full"] }
```

**Previous LTS Releases:**

| Version  | Supported Until   |
|----------|-------------------|
| `1.8.x`  | February 2022     |
| `1.14.x` | June 2022         |
| `1.18.x` | June 2023         |
| `1.20.x` | September 2023    |
| `1.25.x` | March 2024        |
| `1.32.x` | September 2024    |
| `1.36.x` | March 2025        |
| `1.38.x` | July 2025         |
