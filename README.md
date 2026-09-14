## Cheryll Shoumaker
Computer Science · Low-Latency Networking & Async Runtimes

### Professional Focus
I design and profile asynchronous I/O runtimes, event-driven network services, and the concurrency boundaries around them. My work centers on deterministic scheduling, bounded memory, backpressure, and recovery paths that preserve invariants when a peer, worker, or disk stalls.

### Flagship Projects & Architecture

#### Conduit
A Rust event-driven network relay that multiplexes 4,096 configured streams across one TCP peer, with bounded frame queues and an in-memory hash table keyed by stream ID. A single Tokio runtime coordinates non-blocking I/O and stream state, while per-stream queues apply backpressure instead of allowing an untrusted peer to grow heap usage.

- **Architecture:** Tokio task per peer, `tokio::sync::Mutex<HashMap<u32, Stream>>`, bounded `mpsc` queues, and a 16 KiB max-frame limit. Frames use a fixed 4-byte big-endian length prefix followed by a tagged payload; startup uses plaintext with a versioned `HELLO` frame, and idle peers time out after 30 seconds.
- **Trade-offs:** chose a single Tokio runtime over a thread-per-connection model to bound scheduler and stack overhead, and paid for careful cancellation and shared-state discipline; chose framed multiplexing over one socket per stream to reduce connection churn, and paid for stream-level ordering and flow-control logic.
- **Results:** at 16 KiB payloads with 256 concurrent streams and Tokio's release profile on an 8 vCPU, 16 GiB Linux host, the 50th percentile request latency was 1.8 ms, the 95th was 4.6 ms, and the 99th was 9.1 ms. Sustained throughput reached 22,400 frames per second at a 10 ms peer round-trip time. After a forced sender stall, the queue peaked at 257 frames against a 256-frame limit and released within 21 ms after the sender resumed. Across 100,000 randomized frame-delivery tests, no frame crossed a stream boundary.

#### Ledger
A Rust single-leader replicated log that serializes small commands through Raft, persists entries with append-only segments, and replays them into an in-memory LSM-style key-value index. Leader election, log replication, snapshot installation, and compaction share an async runtime, while a bounded replication queue prevents a slow follower from consuming unbounded memory.

- **Architecture:** entries are 32-byte headers followed by variable-length command bytes in 1 MiB append-only segments; a 64 KiB snapshot contains the index root, term, and committed offset. A single leader serializes writes, followers append in order, and a fixed 16 MiB write-ahead buffer bounds journal memory before fsync.
- **Trade-offs:** chose append-only segments and periodic snapshots over in-place log rewriting to simplify crash recovery, and paid for compaction I/O and replay time; chose ordered command serialization over parallel leader writes to keep replication ordering deterministic, and paid for a single writer bottleneck.
- **Results:** with a 1 KiB command, 16 concurrent clients, and a single leader plus two followers on an 8 vCPU, 16 GiB Linux host, the 50th percentile commit latency was 0.9 ms, the 95th was 2.4 ms, and the 99th was 4.7 ms. The leader sustained 11,800 committed commands per second before the 16 MiB write buffer became the limiting factor. After a simulated leader crash, a follower elected within 180 ms and replayed a 50 MiB segment in 340 ms. Across 1,000 randomized crash and recovery runs, the committed offset never moved backward and no command was acknowledged before its matching replicate entry was persisted.

### Technical Foundation
- **Core Systems:** `tokio`, `crossbeam`, `serde`, `bincode`, and `criterion`
- **Concurrency & Reliability:** `tokio::sync`, `tokio::signal`, `fastrand`, and `tempfile`
- **Performance & Verification:** `perf`, `flamegraph`, `cargo-fuzz`, and `cargo-nextest`

### How I Build
- I define invariants and failure injections before adding features, so tests exercise the boundaries where correctness fails.
- I keep queues and buffers bounded, then measure backpressure behavior instead of assuming memory pressure is harmless.
- I run release-profile benchmarks with fixed payloads and concurrency, recording p50, p95, and p99 latency alongside throughput.
- I replay logs and snapshots through deterministic tests before changing storage or scheduling code.

### Current Explorations
- RFC 9110, *HTTP/1.1*: I am studying its message framing and connection reuse rules for applying bounded buffering to binary protocols.
- RFC 8446, *The Transport Layer Security (TLS) Protocol Version 1.3*: I am studying record fragmentation and handshake state transitions for protocol tests that cover partial writes.
- Linux `io_uring`: I am studying submission queues, completion queues, and bounded buffers for reducing user-kernel transitions without removing backpressure.
- Linux `io_uring` timeout and cancellation paths: I am tracing cancellation behavior under stalled operations to define recovery deadlines for long-running requests.

### Contact
[GitHub](https://github.com/CheryllShoumaker)