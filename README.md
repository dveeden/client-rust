# TiKV Client (Rust)

[![Build Status](https://travis-ci.org/tikv/client-rust.svg?branch=master)](https://travis-ci.org/tikv/client-rust)

[Docs](https://www.tikv.dev/doc/rust-client/tikv_client/)

This crate provides an easy-to-use client for [TiKV](https://github.com/tikv/tikv), a distributed, transactional key-value database written in Rust.

This crate lets you connect to a TiKV cluster and use either a transactional or raw (simple get/put style without transactional consistency guarantees) API to access and update your data.

The TiKV Rust client is an open source (Apache 2) project maintained by the TiKV Authors. We welcome contributions, see below for more info.

## Getting started

The TiKV client is a Rust library (crate). To use this crate in your project, add the following dependency to your `Cargo.toml`:

```toml
[dependencies]
tikv-client = 0.0.99
```

The minimum supported version of Rust is 1.40. The minimum supported version of TiKV is 5.0.

The general flow of using the client crate is to create either a raw or transaction client object (which can be configured) then send commands using the client object, or use it to create transactions objects. In the latter case, the transaction is built up using various commands and then committed (or rolled back).

### Examples

Raw mode:

```rust
use tikv_client::RawClient;

let client = RawClient::new(vec!["127.0.0.1:2379"]).await?;
client.put("key".to_owned(), "value".to_owned()).await?;
let value = client.get("key".to_owned()).await?;
```

Transactional mode:

```rust
use tikv_client::TransactionClient;

let txn_client = TransactionClient::new(vec!["127.0.0.1:2379"]).await?;
let mut txn = txn_client.begin_optimistic().await?;
txn.put("key".to_owned(), "value".to_owned()).await?;
let value = txn.get("key".to_owned()).await?;
txn.commit().await?;
```

## API summary

The TiKV Rust client supports several levels of abstraction. The most convenient way to use the client is via `RawClient` and `TransactionClient`. This gives a very high-level API which mostly abstracts over the distributed nature of the store and has sensible defaults for all protocols. This interface can be configured, primarily when creating the client or transaction objects via the `Config` and `TransactionOptions` structs. Using some options, you can take over parts of the protocols (such as retrying failed messages) yourself.

The lowest level of abstraction is to create and send gRPC messages directly to TiKV (and PD) nodes. The `tikv-client-store` and `tikv-client-pd` crates make this easier than using the protobuf definitions and a gRPC library directly, but give you the same level of control.

In between these levels of abstraction, you can send and receive individual messages to the TiKV cluster, but take advantage of library code for common operations such as resolving data to regions and thus nodes in the cluster, or retrying failed messages. This can be useful for testing a TiKV cluster or for some advanced use cases. See the `client_rust::request` module for this API, and `client_rust::raw::lowering` and `client_rust::transaction::lowering` for convenience methods for creating request objects.

The rest of this document describes only the `RawClient`/`TransactionClient` APIs.

Important note: It is **not recommended or supported** to use both the raw and transactional APIs on the same database.

### Types

`Key`: a key in the store. `String` and `Vec<u8>` implement `Into<Key>`, so you can pass them directly into client functions.

`Value`: a value in the store; just an alias of `Vec<u8>`.

`KvPair`: a pair of a `Key` and a `Value`. It provides convenience methods for conversion to and from other types.

`BoundRange`: used for range related requests like `scan`. It implements `From` for Rust ranges so you can pass a Rust range of keys to the request, e.g., `client.delete_range(vec![]..)`.

### Raw requests

| Request            | Main parameter type | Result type      | Noteworthy Behavior                            |
| ------------------ | ------------------- | ---------------- | ---------------------------------------------- |
| `put`              | `KvPair`            |                  |                                                |
| `get`              | `Key`               | `Option<Value>`  |                                                |
| `delete`           | `Key`               |                  |                                                |
| `delete_range`     | `BoundRange`        |                  |                                                |
| `scan`             | `BoundRange`        | `Vec<KvPair>`    |                                                |
| `batch_put`        | `Iter<KvPair>`      |                  |                                                |
| `batch_get`        | `Iter<Key>`         | `Vec<KvPair>`    | Skips non-existent keys; does not retain order |
| `batch_delete`     | `Iter<Key>`         |                  |                                                |
| `batch_scan`       | `Iter<BoundRange>`  | `Vec<KvPair>`    | See docs for `each_limit` parameter behavior. The order of ranges is retained. |
| `batch_scan_keys`  | `Iter<BoundRange>`  | `Vec<Key>`       | See docs for `each_limit` parameter behavior. The order of ranges is retained. |
| `compare_and_swap` | `Key` + 2x `Value`  | `(Option<Value>, bool)` |                                         | 

### Transactional requests

| Request                | Main parameter type | Result type     | Noteworthy Behavior                                                |
| ---------------------- | ------------------- | --------------- | ------------------------------------------------------------------ |
| `put`                  | `KvPair`            |                 |                                                                    |
| `get`                  | `Key`               | `Option<value>` |                                                                    |
| `get_for_update`       | `Key`               | `Option<value>` |                                                                    |
| `key_exists`           | `Key`               | `bool`          |                                                                    |
| `delete`               | `Key`               |                 |                                                                    |
| `scan`                 | `BoundRange`        | `Iter<KvPair>`  |                                                                    |
| `scan_keys`            | `BoundRange`        | `Iter<Key>`     |                                                                    |
| `batch_get`            | `Iter<Key>`         | `Iter<KvPair>`  | Skips non-existent keys; does not retain order                     |
| `batch_get_for_update` | `Iter<Key>`         | `Iter<KvPair>`  | Skips non-existent keys; does not retain order                     |
| `lock_keys`            | `Iter<Key>`         |                 |                                                                    |
| `send_heart_beat`      |                     | `u64` (TTL)     |                                                                    |
| `gc`                   | `Timestamp`         | `bool`          | Returns true if the latest safepoint in PD equals the parameter    |

# Development and contributing

We welcome your contributions! Contributing code is great, we also appreciate filing [issues](https://github.com/tikv/client-rust/issues/new) to identify bugs and provide feedback, adding tests or examples, and improvements to documentation.

## Building and testing

We use the standard Cargo workflows, e.g., `cargo build` to build and `cargo test` to run unit tests. You will need to use a nightly Rust toolchain to build and run tests.

Running integration tests or manually testing the client with a TiKV cluster is a little bit more involved. The easiest way is to use [TiUp](https://github.com/pingcap/tiup) to initialise a cluster on your local machine:

```
tiup playground nightly --db 0 --tiflash 0 --monitor false
```

Then if you want to run integration tests:

```
PD_ADDRS="127.0.0.1:2379" cargo test --package tikv-client --test integration_tests --features integration-tests
```

## Creating a PR

We use a standard GitHub PR workflow. We run CI on every PR and require all PRs to build without warnings (including clippy and Rustfmt warnings), pass tests, have a DCO sign-off (use `-s` when you commit, the DCO bot will guide you through completing the DCO agreement for your first PR), and have at least one review. If any of this is difficult for you, don't worry about it and ask on the PR.

To run CI-like tests locally, we recommend you run `cargo clippy`, `cargo test`, and `cargo fmt` before submitting your PR. See above for running integration tests, but you probably won't need to worry about this for your first few PRs.

Please follow PingCAP's  [Rust style guide](https://pingcap.github.io/style-guide/rust/). All code PRs should include new tests or test cases.

## Getting help

If you need help, either to find something to work on, or with any technical problem, the easiest way to get it is via Slack. We monitor the client-rust (better for general client questions) and sig-transaction (better for technical questions about TiKV's transaction protocol) channels on the [tikv-wg slack](https://tikv.org/chat).

You can also get help on GitHub issues or PRs directly. You can just ask a question; if you don't get a response, you should ping @nrc or @ekexium.
