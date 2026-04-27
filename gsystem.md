Let me break each project down seriously — scope, approach, phases, and what you're actually learning at each step.

---

# GO PROJECTS

---

## 1. Load Balancer

### What you're really building
A reverse proxy that sits in front of multiple backend servers, distributes traffic, and handles backend failures gracefully.

### Phase 1 — Dumb round-robin
```
Client → Load Balancer → [Server A, Server B, Server C]
```
- Open a TCP listener
- Maintain a list of backend addresses
- For each incoming connection, pick the next backend (atomic counter % len(backends))
- Proxy the connection — read from client, write to backend, read response, write back

**Key Go concepts:** `net.Listen`, `net.Dial`, `io.Copy`, goroutines per connection.

Don't use `net/http` yet. Work at the TCP level first. Feel the bytes.

### Phase 2 — Health checks
- Every N seconds, each backend gets a goroutine that dials it and checks if it responds
- Mark backends as `healthy` or `unhealthy`
- Round-robin only over healthy backends
- Protect the backends list with `sync.RWMutex` — reads are frequent, writes are rare

**You'll hit a real problem:** what if all backends go down? What if one comes back? Handle it explicitly, don't hand-wave it.

### Phase 3 — Connection pooling
- Instead of opening a new TCP connection to the backend on every request, maintain a pool of idle connections
- `sync.Pool` is tempting but wrong here (it's for short-lived objects). Implement a proper pool with a buffered channel of connections
- Add max pool size, connection TTL, validation on borrow

### Phase 4 — Observability
- Count requests per backend, error rates, latency histograms
- Expose a `/metrics` endpoint in Prometheus format
- This forces you to think about atomic counters and concurrent map access

### Pitfalls to expect
- `io.Copy` in both directions (client↔backend) requires two goroutines and careful shutdown sequencing
- Half-closed TCP connections will confuse you — study `CloseWrite()`
- Goroutine leaks if you don't handle errors properly — use `errgroup`

---

## 2. CLI Log Aggregator

### What you're really building
A tool that watches multiple files simultaneously, like `tail -f` but across N files, with filtering and formatting.

```bash
logagg --files app.log,error.log,access.log --filter "ERROR" --format json
```

### Phase 1 — Single file tail
- Open a file, seek to end, poll for new bytes in a loop with `time.Sleep`
- Print new lines as they arrive
- Handle file rotation (the file gets replaced — detect via inode or size decrease)

### Phase 2 — Multiple files, fan-in pattern
This is the core Go lesson of this project.

```go
func tailFile(ctx context.Context, path string, out chan<- LogLine) {
    // reads lines, sends to shared channel
}

func main() {
    out := make(chan LogLine, 100)
    for _, f := range files {
        go tailFile(ctx, f, out)
    }
    for line := range out {
        // process and print
    }
}
```

One goroutine per file, all feeding into one channel. The main goroutine consumes. This is the fan-in pattern — internalize it.

### Phase 3 — Graceful shutdown
- Use `context.WithCancel` — when the user hits Ctrl+C, cancel the context
- Every goroutine must respect `ctx.Done()`
- The main loop must drain the channel before exiting

```go
sigCh := make(chan os.Signal, 1)
signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
go func() {
    <-sigCh
    cancel()
}()
```

**This is where most beginners fail.** Graceful shutdown is non-trivial. Do it properly.

### Phase 4 — Filtering and formatting
- Filter lines by regex (compile once, reuse — `regexp.MustCompile`)
- Format output as plain text, JSON, or colored terminal output
- Parse structured logs (JSON lines) and pretty-print fields

### What you're learning
The fan-in pattern, `context` cancellation, signal handling, and real concurrency composition. These patterns appear in almost every serious Go codebase.

---

## 3. Rate Limiter Library

### What you're really building
A reusable package that can be imported and used as HTTP middleware or standalone.

```go
limiter := ratelimit.New(100, time.Minute) // 100 requests per minute
allowed := limiter.Allow("user-123")
```

### Phase 1 — Token bucket, single key
- Tokens refill at a fixed rate
- `Allow()` consumes one token, returns bool
- Naive implementation: `sync.Mutex` + struct fields for tokens and last refill time
- Test it with `go test -race` — the race detector is your best friend here

### Phase 2 — Per-key limiting
- Map of `string → bucket`
- Problem: the map itself needs protection, and you don't want one lock for everything
- Solution: sharded map — split keys across N mutexes by `hash(key) % N`
- This is a real performance pattern used in production systems

### Phase 3 — Sliding window algorithm
Token bucket has edge cases (burst at window boundaries). Implement sliding window log:
- Store timestamps of recent requests in a circular buffer
- Count how many fall within the window
- More accurate, more memory per key

### Phase 4 — Benchmark everything
```bash
go test -bench=. -benchmem -count=5
```
- Compare mutex vs channels vs atomic for your hot path
- You'll find that `sync/atomic` wins for simple counters but is harder to reason about
- Learn to read benchmark output and calculate ns/op

### Phase 5 — HTTP middleware
```go
func Middleware(limiter *Limiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            key := r.Header.Get("X-User-ID")
            if !limiter.Allow(key) {
                w.WriteHeader(http.StatusTooManyRequests)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

---

## 4. Key-Value Store with TCP Interface

### What you're really building
A simplified Redis. In-memory, concurrent, persistent.

```bash
> SET foo bar
OK
> GET foo
bar
> DEL foo
OK
```

### Phase 1 — In-memory store, no networking
- `map[string][]byte` protected by `sync.RWMutex`
- Operations: GET, SET, DEL, EXISTS
- Write thorough tests before touching networking

### Phase 2 — TCP server with a text protocol
Define a simple protocol:
```
SET key value\n
GET key\n
DEL key\n
```
- `net.Listen("tcp", ":6380")`
- One goroutine per connection
- Parse commands with `bufio.Scanner`
- Return responses line by line

**Don't invent binary protocols yet.** Text first, understand the parsing problem.

### Phase 3 — TTL support
- `SET foo bar EX 60` — expires in 60 seconds
- Store expiry timestamps alongside values
- Background goroutine that sweeps expired keys periodically (lazy deletion + active sweep)
- Now your struct is more complex — protect carefully

### Phase 4 — Persistence (Write-Ahead Log)
Every write goes to a log file before being applied:
```
SET foo bar 1714000000
DEL foo 1714000001
```
On startup, replay the log to reconstruct state. This is how real databases work. You'll immediately see why compaction matters — the log grows forever. Implement snapshotting: periodically dump the full map to disk, truncate the log.

### Phase 5 — Pipelining
Redis clients send multiple commands without waiting for responses. Handle this: read all commands from the buffer, execute them, write all responses. Significant throughput improvement.

---

# RUST PROJECTS

---

## 1. Memory Allocator

### What you're really building
Your own heap allocator using a fixed-size byte array as backing storage.

### Phase 1 — Fixed-size block allocator
Simplest possible version:
- 1MB static buffer: `static mut HEAP: [u8; 1024 * 1024] = [0; ...]`
- Divide into fixed-size blocks (e.g. 64 bytes each)
- Bitmap tracking which blocks are free
- `alloc(size)` finds a free block, marks it used, returns pointer
- `free(ptr)` marks the block free again

You need `unsafe` here. That's the point — understand when and why.

### Phase 2 — Variable-size allocator (free list)
Real allocators handle variable sizes:
- Each allocation has a header: `size`, `is_free`, pointer to next free block
- `alloc(size)` walks the free list, finds first-fit block
- `free(ptr)` marks block free and coalesces adjacent free blocks (fragmentation problem)

This is where it gets genuinely hard. Coalescing requires careful pointer arithmetic. You will write bugs here. The borrow checker will catch some but not all — pointer arithmetic is unsafe.

### Phase 3 — Implement the GlobalAlloc trait
```rust
use std::alloc::{GlobalAlloc, Layout};

struct MyAllocator;

unsafe impl GlobalAlloc for MyAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 { ... }
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) { ... }
}

#[global_allocator]
static A: MyAllocator = MyAllocator;
```

Now your allocator runs your entire program. Run the test suite. See what breaks.

### What you're learning
Pointer arithmetic in Rust, `unsafe`, alignment requirements, memory fragmentation. After this, every time you see `Box`, `Vec`, or `Arc`, you'll know exactly what's happening underneath.

---

## 2. Binary File Parser

### What you're really building
A parser for a real binary format that produces a structured, typed representation you can inspect and manipulate.

**Recommended format: PNG** — well-documented, manageable complexity, visually satisfying.

### Phase 1 — Understand the format
PNG structure:
```
8-byte signature
[chunk: 4-byte length][4-byte type][data][4-byte CRC]
[chunk: ...]
...
IEND chunk
```
Read the spec. Draw it on paper. Understand it before writing a line of code.

### Phase 2 — Model the types
```rust
struct PngChunk {
    length: u32,
    chunk_type: [u8; 4],
    data: Vec<u8>,
    crc: u32,
}

enum ChunkType {
    IHDR(ImageHeader),
    IDAT(Vec<u8>),
    IEND,
    Unknown(String),
}

struct ImageHeader {
    width: u32,
    height: u32,
    bit_depth: u8,
    color_type: u8,
    // ...
}
```

Model it precisely. Rust enums are perfect for this — use them.

### Phase 3 — Parse bytes manually
```rust
fn parse_u32_be(bytes: &[u8]) -> u32 {
    u32::from_be_bytes(bytes[0..4].try_into().unwrap())
}
```
No `nom`, no `byteorder` at first. Do it manually to feel the byte manipulation. Then swap to `byteorder` crate and appreciate what it saves you.

### Phase 4 — Validate
- Verify the 8-byte PNG signature
- Verify CRC for each chunk (use `crc32` crate)
- Return proper `Result<T, ParseError>` — define your own error type with `thiserror`

### Phase 5 — Reconstruct
Take your parsed structure and write it back to bytes. If `parse(write(parse(file))) == parse(file)`, you're correct. This is a round-trip test and it's brutal at catching bugs.

---

## 3. Threadpool from Scratch

### What you're really building
A fixed-size pool of worker threads that execute submitted jobs, with graceful shutdown.

```rust
let pool = ThreadPool::new(4);
pool.execute(|| println!("job 1"));
pool.execute(|| println!("job 2"));
// pool drops → waits for all jobs to finish
```

### Phase 1 — Basic structure
```rust
type Job = Box<dyn FnOnce() + Send + 'static>;

struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Job>,
}

struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}
```

The `Send + 'static` bounds are non-negotiable — understand why. `Send` means safe to send across threads. `'static` means no borrowed references that could dangle.

### Phase 2 — Shared job queue
```rust
let (tx, rx) = mpsc::channel::<Job>();
let rx = Arc::new(Mutex<mpsc::Receiver<Job>>>(Mutex::new(rx));

// Each worker:
let rx = Arc::clone(&rx);
thread::spawn(move || loop {
    let job = rx.lock().unwrap().recv();
    match job {
        Ok(job) => job(),
        Err(_) => break, // channel closed = shutdown signal
    }
});
```

`Arc<Mutex<Receiver>>` — understand exactly why each layer is necessary. This is the most educational type in the codebase.

### Phase 3 — Graceful shutdown
Implement `Drop` for `ThreadPool`:
```rust
impl Drop for ThreadPool {
    fn drop(&mut self) {
        drop(self.sender.take()); // close channel → workers get Err → exit loop
        for worker in &mut self.workers {
            worker.thread.join().unwrap();
        }
    }
}
```

Closing the sender signals all workers to stop. Joining ensures they finish their current job. This is the correct shutdown sequence — anything else leaks threads or drops work.

### Phase 4 — Add a work-stealing queue
The `mpsc` channel is a single shared queue. A more sophisticated approach: each worker has its own queue, and idle workers steal from busy workers' queues. This is what Tokio and Rayon actually do. Hard but extremely educational.

---

## 4. Simple Shell

### What you're really building
A UNIX shell: prompt, parse input, execute commands, handle pipes, handle signals.

```bash
$ ls -la | grep rust | wc -l
$ cd /tmp
$ echo hello > output.txt
$ ^C  (doesn't kill the shell)
```

### Phase 1 — Read, parse, execute
- Print prompt, read a line with `std::io::stdin()`
- Split on whitespace → command + args
- `std::process::Command::new(cmd).args(args).spawn()` and `.wait()`

This works in 20 lines. Don't stop here.

### Phase 2 — Built-in commands
`cd`, `exit`, `export` cannot be implemented as child processes — they must modify the shell's own state:
```rust
match cmd {
    "cd" => std::env::set_current_dir(path)?,
    "exit" => std::process::exit(0),
    _ => spawn_external(cmd, args),
}
```

### Phase 3 — Pipes
```bash
ls | grep foo
```
- Create a `pipe()` — two file descriptors, one read end, one write end
- Left command's stdout → write end
- Right command's stdin → read end
- Both spawn concurrently, shell waits for both

In Rust:
```rust
use std::os::unix::io::FromRawFd;
// nix crate for pipe(), dup2()
```
You need the `nix` crate here. Pipes require `unsafe` Unix syscalls.

### Phase 4 — Signal handling
- `Ctrl+C` sends SIGINT to the foreground process group, not the shell
- Shell must catch SIGINT and not die
- Use the `signal-hook` crate

```rust
use signal_hook::{consts::SIGINT, iterator::Signals};
```

### Phase 5 — Redirections
```bash
command > file.txt
command >> file.txt
command < input.txt
```
Open files, get file descriptors, dup2 to stdout/stdin before exec. Raw Unix I/O.

---

# The HTTP Server (Do This Last, In Both Languages)

### Spec (identical for both implementations)
- TCP listener on port 8080
- Parse raw HTTP/1.1 requests manually (no framework)
- Handle: GET, POST
- Serve static files from a directory
- Handle concurrent connections
- Return proper status codes and headers

### What to parse
```
GET /index.html HTTP/1.1\r\n
Host: localhost:8080\r\n
\r\n
```
Split on `\r\n`, parse request line, parse headers into a map, read body if Content-Length present.

### Go approach
- Goroutine per connection
- `bufio.Reader` for parsing
- `sync.WaitGroup` for graceful shutdown

### Rust approach
- One thread per connection first (simple)
- Then refactor to Tokio async (async/await, `tokio::net::TcpListener`)
- You'll feel exactly what async buys you and what it costs in complexity

### What the comparison teaches you
- Go: goroutines are cheap, you barely think about it
- Rust sync threads: heavier, but predictable
- Rust async: maximum performance, maximum complexity, requires understanding the executor model

After doing both, you'll have a genuine opinion about Go vs Rust tradeoffs — not a Reddit take, an informed one.

---

## General Advice on Approach

**Don't skip phases.** The temptation is to jump to the interesting part. Phase 1 of every project exists to give you a working foundation you can break deliberately in later phases.

**Write tests from phase 1.** Both languages have excellent test tooling built in. `go test` and `cargo test`. If you can't test it, you don't understand it well enough yet.

**Use the linters.** `golangci-lint` for Go, `clippy` for Rust. Treat their warnings as lessons, not noise.

**Read error messages completely.** Rust's compiler errors are famously good. Go's are terse but precise. Neither language rewards skimming.

**Timebox phases.** Give each phase a week max. Imperfect and shipped beats perfect and abandoned.
