# workerpool

[![Build Status](https://travis-ci.org/lucasem/workerpool.svg?branch=master)](https://travis-ci.org/lucasem/workerpool)
[![crates.io](https://img.shields.io/crates/v/workerpool.svg)](https://crates.io/crates/workerpool)
[![docs.rs](https://docs.rs/workerpool/badge.svg)](https://docs.rs/workerpool)

A worker threadpool used to execute a number of jobs atop stateful workers
in parallel. It spawns a specified number of worker threads and replenishes
the pool if any worker threads panic.

A single `Worker` runs in its own thread, to be implemented according to the trait:

```rust
pub trait Worker : Default {
    type Input: Send;
    type Output: Send;

    fn execute(&mut self, Self::Input) -> Self::Output;
}
```

## Usage

```toml
[dependencies]
workerpool = "1.2"
```

To use [crossbeam's
channels](https://docs.rs/crossbeam-channel/0.3.8/crossbeam_channel/)
instead of [`std::sync::mpsc`](https://doc.rust-lang.org/stable/std/sync/mpsc/),
enable the `crossbeam` feature:

```toml
[dependencies]
workerpool = { version = "1.2", features = ["crossbeam"] }
```

This crate provides `Pool<W> where W: Worker`. With a pool, there are four
primary functions of interest:
- `Pool::<MyWorker>::new(n_threads)` creates a new pool for a particular `Worker`.
- `pool.execute(inp)` _non-blocking_ executes the worker and ignores the return value.
- `pool.execute_to(tx, inp)` _non-blocking_ executes the worker and sends return value to
  the given [Sender](https://doc.rust-lang.org/std/sync/mpsc/struct.Sender.html).
- `pool.join()` _blocking_ waits for all tasks (from `execute` and
  `execute_to`) to complete.

A worker is provided in `workerpool::thunk`, a stateless `ThunkWorker<T>`.
It executes on inputs of `Thunk<T>`, effectively argumentless functions that
are `Sized + Send`. These thunks are creates by wrapping functions which
return `T` with `Thunk::of`.

```rust
extern crate workerpool;

use workerpool::Pool;
use workerpool::thunk::{Thunk, ThunkWorker};
use std::sync::mpsc::channel;

fn main() {
    let n_workers = 4;
    let n_jobs = 8;
    let pool = Pool::<ThunkWorker<i32>>::new(n_workers);
    
    let (tx, rx) = channel();
    for i in 0..n_jobs {
        pool.execute_to(tx.clone(), Thunk::of(move || i * i));
    }
    
    assert_eq!(140, rx.iter().take(n_jobs as usize).sum());
}
```

For stateful workers, you have to implement `Worker` yourself.

Suppose there's a line-delimited process, such as `cat` or `tr`, which you'd
like running on many threads for use in a pool-like manner. You may create
and use a worker, with maintained state of the stdin/stdout for the process,
as follows:

```rust
extern crate workerpool;

use workerpool::{Worker, Pool};
use std::process::{Command, ChildStdin, ChildStdout, Stdio};
use std::io::prelude::*;
use std::io::{self, BufReader};
use std::sync::mpsc::channel;

struct LineDelimitedProcess {
    stdin: ChildStdin,
    stdout: BufReader<ChildStdout>,
}
impl Default for LineDelimitedProcess {
    fn default() -> Self {
        let child = Command::new("cat")
            .stdin(Stdio::piped())
            .stdout(Stdio::piped())
            .stderr(Stdio::inherit())
            .spawn()
            .unwrap();
        Self {
            stdin: child.stdin.unwrap(),
            stdout: BufReader::new(child.stdout.unwrap()),
        }
    }
}
impl Worker for LineDelimitedProcess {
    type Input = Box<[u8]>;
    type Output = io::Result<String>;

    fn execute(&mut self, inp: Self::Input) -> Self::Output {
        self.stdin.write_all(&*inp)?;
        self.stdin.write_all(b"\n")?;
        self.stdin.flush()?;
        let mut s = String::new();
        self.stdout.read_line(&mut s)?;
        s.pop(); // exclude newline
        Ok(s)
    }
}

fn main() {
    let n_workers = 4;
    let n_jobs = 8;
    let pool = Pool::<LineDelimitedProcess>::new(n_workers);
    
    let (tx, rx) = channel();
    for i in 0..n_jobs {
        let inp = Box::new([97 + i]);
        pool.execute_to(tx.clone(), inp);
    }
    
    // output is a permutation of "abcdefgh"
    let mut output = rx.iter()
        .take(n_jobs as usize)
        .fold(String::new(), |mut a, b| {
            a.push_str(&b.unwrap());
            a
        })
        .into_bytes();
    output.sort();
    assert_eq!(output, b"abcdefgh");
}
```


## Similar libraries

* [threadpool](http://github.com/rust-threadpool/rust-threadpool)
* [rust-scoped-pool](http://github.com/reem/rust-scoped-pool)
* [scoped-threadpool-rs](https://github.com/Kimundi/scoped-threadpool-rs)
* [crossbeam](https://github.com/aturon/crossbeam)

## License

This work is derivative of
[threadpool](http://github.com/rust-threadpool/rust-threadpool).

Licensed under either of

 * Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
 * MIT license ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally
submitted for inclusion in the work by you, as defined in the Apache-2.0
license, shall be dual licensed as above, without any additional terms or
conditions.
