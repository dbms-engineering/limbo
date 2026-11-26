# limbo

A study implementation of the design described in:

> Pekka Enberg, Sasu Tarkoma, Jon Crowcroft, and Ashwin Rao. 2024.
> **Serverless Runtime / Database Co-Design With Asynchronous I/O.**
> In *7th International Workshop on Edge Systems, Analytics and Networking (EdgeSys '24)*,
> April 22, 2024, Athens, Greece. ACM, pp. 49-54.
> DOI: [10.1145/3642968.3654821](https://doi.org/10.1145/3642968.3654821)
> Open-access copy (CC BY 4.0): [University of Cambridge repository](https://www.repository.cam.ac.uk/handle/1810/376948)

This repository has two purposes:

1. **Code provenance.** All code here traces back to that paper. Each design decision should
   point to the section of the paper it implements, or say explicitly where it departs from it.
2. **Understanding.** We rebuild the ideas from first principles to understand *why* a
   synchronous embedded database limits a multi-tenant runtime, and *how* asynchronous bytecode
   I/O and a decoupled storage layer remove that limit.

This is not the Limbo project maintained by Turso, and it is not affiliated with the authors.
It is an independent reimplementation for learning.

---

## Contents

1. [The problem](#1-the-problem)
2. [Background: SQLite's execution model](#2-background-sqlites-execution-model)
3. [The proposal](#3-the-proposal)
4. [Asynchronous bytecode instructions](#4-asynchronous-bytecode-instructions)
5. [Decoupling the query and storage engines](#5-decoupling-the-query-and-storage-engines)
6. [Evaluation reported in the paper](#6-evaluation-reported-in-the-paper)
7. [Limitations and future work stated by the authors](#7-limitations-and-future-work-stated-by-the-authors)
8. [Questions this repository investigates](#8-questions-this-repository-investigates)
9. [Repository layout](#9-repository-layout)
10. [Building](#10-building)
11. [Glossary](#11-glossary)
12. [References](#12-references)

---

## 1. The problem

Serverless platforms (for example Cloudflare Workers) run many tenants' functions in a single OS
process, close to users at the edge. Most real applications need state, and that state usually
lives in a cloud database. Every query then costs a network round trip, which cancels out the
latency advantage of running at the edge.

The paper calls embedding an in-process database such as SQLite in the runtime "the holy grail"
for low-latency access: a query becomes a function call instead of a network request.

The obstacle is that **SQLite's I/O is synchronous**. When a query needs a page that is not in
the page cache, the calling thread blocks in a `read()` system call. In a multi-tenant runtime
this has two consequences:

- **Latency.** One tenant's disk wait stalls the thread, and every tenant scheduled on it.
- **Density.** Concurrency requires more OS threads. Once the thread count exceeds the number
  of cores, tail latency degrades sharply (Section 6). This limits how many tenants a
  provider can place per CPU and per unit of DRAM.

Paper thesis: *serverless runtimes need to be co-designed with databases to retain their latency
advantage for edge-native applications.*

## 2. Background: SQLite's execution model

SQLite has two halves (Figure 1 in the paper):

```
            +---------------------------+
  core      | Interface (C API)         |
            | SQL command processor     |   SQL text -> bytecode program
            | Virtual machine (VDBE)    |   executes bytecode
            +---------------------------+
  backend   | B-tree                    |   tables and indexes
            | Pager                     |   page cache, transactions
            | OS interface              |   read(), write(), fsync()  <-- blocks
            +---------------------------+
```

The query lifecycle for `SELECT * FROM users`:

1. `sqlite3_open()` opens the database file through the OS interface (POSIX `open`).
2. `sqlite3_prepare()` compiles SQL into a bytecode program.
3. `sqlite3_step()` executes bytecode until a row is produced (`SQLITE_ROW`) or the program ends
   (`SQLITE_DONE`).
4. When an instruction such as `Next` moves a cursor onto a B-tree page that is not cached, the
   pager calls `read()`. **The thread blocks inside `sqlite3_step()`.**

Nothing in this design gives control back to the caller while I/O is in flight.

## 3. The proposal

The design, named *Limbo* in the paper, rests on two building blocks:

1. **Asynchronous bytecode instructions**, so the library never blocks.
2. **Decoupled query and storage engines**, so the runtime, not the library, owns I/O.

```
  +------------------------------------------------------------+
  | Serverless runtime                                         |
  |                                                            |
  |   +-----------------------+         +-------------------+  |
  |   | Database library      |         | Event loop        |  |
  |   |                       |         |                   |  |
  |   | Interface             |         |  poll completions |  |
  |   | SQL command processor |         |        |          |  |
  |   | Virtual machine  -----+--+      |        v          |  |
  |   | B-tree                |  |      |  resume the query |  |
  |   | Pager                 |  |      +--------+----------+  |
  |   +-----------------------+  |               |             |
  |                              | submit I/O,   |             |
  |                              | return        |             |
  |                              v               |             |
  |                  +--------------------------+|             |
  |                  | I/O module (provided by  |<             |
  |                  | the runtime: io_uring)   |              |
  |                  +--------------------------+              |
  +------------------------------------------------------------+
```

The OS interface layer is removed from the library. The application or runtime provides a
**virtual I/O module**. The library submits requests through it and returns control. The
runtime polls for completions in its own event loop and resumes the query when the data is
ready. The paper's prototype implements the module with Linux `io_uring`, but the mechanism
does not depend on any particular I/O interface.

The interface and SQL command processor are kept conceptually the same. The prototype
reimplements them in Rust and reuses the SQLite parser through the `sqlite3-parser` crate, a
Rust translation of the SQLite parser.

## 4. Asynchronous bytecode instructions

Every bytecode instruction that may perform I/O is split into a pair: one instruction *submits*
the I/O, the other *waits* for it.

| SQLite instruction | Limbo replacement             |
|--------------------|-------------------------------|
| `OpenRead`         | `OpenReadAsync`, `OpenReadAwait` |
| `Rewind`           | `RewindAsync`, `RewindAwait`  |
| `Next`             | `NextAsync`, `NextAwait`      |

Bytecode for `SELECT * FROM users` on
`CREATE TABLE users (id INT PRIMARY KEY, username TEXT)` (Figure 2 in the paper):

```
SQLite                                 Limbo
addr  opcode       p1  p2  p3          addr  opcode          p1  p2  p3
0     Init         0   8   0           0     Init            0   11  0
1     OpenRead     0   2   0           1     OpenReadAsync   0   2   0
2     Rewind       0   7   0           2     OpenReadAwait   0   0   0
3       Column     0   0   1           3     RewindAsync     0   0   0
4       Column     0   1   2           4     RewindAwait     0   10  0
5       ResultRow  1   2   0           5     RowId           0   1   0
6     Next         0   3   0           6     Column          0   1   1
7     Halt         0   0   0           7     ResultRow       0   2   0
8     Transaction  0   0   1           8     NextAsync       0   0   0
9     Goto         0   1   0           9     NextAwait       0   4   0
                                       10    Halt            0   0   0
                                       11    Transaction     0   0   0
                                       12    Goto            0   1   0
```

The register layout is the same as SQLite's. Only the I/O instructions change.

Note on naming: the bytecode listing uses `OpenReadAwait` and `RewindAwait`, but the paper's
prose and caption also use `OpenWaitAsync` and `RewindWait`. This repository follows the
bytecode listing.

### Control flow for `NextAsync` / `NextAwait`

```
application                     sqlite3_step()                     I/O module
    |                                 |                                  |
    |---- step() -------------------->|                                  |
    |                                 | NextAsync: page not cached       |
    |                                 |---- submit read ---------------->|
    |<--- SQLITE_IO ------------------|                                  |
    |                                 |                                  |
    |  (run other tenants' work)      |                                  |
    |                                 |                                  |
    |<--------------------------------+------- completion (event loop) -|
    |---- step() -------------------->|                                  |
    |                                 | NextAwait: page now available    |
    |<--- SQLITE_ROW -----------------|                                  |
```

`sqlite3_step()` gains a new result code, `SQLITE_IO`, which means "I/O was submitted; call me
again later". The caller can then either step again right away (`NextAwait` blocks on the I/O),
or do other work until the event loop reports the completion.

## 5. Decoupling the query and storage engines

Asynchronous instructions remove blocking. Decoupling is what makes **co-design** possible:

- The **query engine** (parser, compiler, VM, B-tree, pager) lives inside the database library.
- **Storage I/O** lives outside the library, behind the virtual I/O module, which the runtime
  implements.

Because the runtime owns I/O, it can:

- batch and schedule I/O from many tenants in a single event loop;
- multiplex many tenants on few threads (coroutines instead of OS threads);
- swap the backing store, for example local `io_uring`, or a remote storage node in an
  Aurora-style disaggregated design, without changing the query engine.

The paper notes that the same architecture could also run outside serverless runtimes, for
example in web browsers.

## 6. Evaluation reported in the paper

The evaluation asks: *what is the impact of synchronous and asynchronous I/O on query tail
latency?*

**Setup**

| Parameter      | Value |
|----------------|-------|
| Machine        | AMD Ryzen 9 3900XT, 12 cores |
| Workload       | Each tenant runs `SELECT * FROM users LIMIT 100`, 1000 times |
| Tenants        | 1 to 100, in steps of 10 |
| Data           | One separate 1 MiB database file per tenant |
| Measurement    | Latency distribution with HdrHistogram (p50, p90, p99, p99.9) |
| SQLite tenancy | One OS thread per tenant, via `rusqlite` |
| Limbo tenancy  | One Rust coroutine per tenant. A simulated I/O loop lets a coroutine run at most 10 iterations, then schedules another coroutine when a query returns an I/O status |

**Results**

- At p50, p90 and p99, the two systems are comparable.
- At p99.9, SQLite's latency jumps as tenants increase from 10 to 20, 20 to 30, 40 to 50 and
  60 to 70, close to multiples of the core count. Synchronous I/O limits multi-tenancy.
- The authors do not yet explain a drop in SQLite's p99.9 above 80 tenants.
- Limbo stays flat. The paper reports **up to a 100x reduction in tail latency**.

**Scope of the result.** This is a preliminary microbenchmark: a single reader per tenant, no
writers, and read-only queries.

## 7. Limitations and future work stated by the authors

- **SQLite compatibility.** The prototype is a clean-slate implementation apart from the parser.
  Reusing SQLite's front end is hard because SQLite is only partly modular, and the backend
  would have to propagate I/O status up to `sqlite3_step()`. The authors propose achieving
  compatibility through a test suite that runs against both SQLite and Limbo.
- **Evaluation methodology.** Planned work: realistic workloads, throughput measurements, TPC-H,
  writers, multiple readers, transaction coordination within one runtime instance, and
  distributed coordination across runtime instances.
- **Runtime architecture.** Serverless functions are stateless and can run on any instance. An
  in-process database means either every instance must have access to the data, or requests
  must be routed to the instances that hold it. Data locality becomes a runtime concern.
- **Privacy and security.** Placing tenant data inside the runtime's address space requires
  isolation. The authors suggest extending existing mechanisms such as V8 isolates, memory
  enclaves, or memory protection at the virtual machine layer.

## 8. Questions this repository investigates

Each item should end with code, a measurement, or a written explanation in this README.

1. What is the smallest bytecode VM that reproduces the `Async` / `Await` instruction split?
2. What exactly must the pager and B-tree return so that `SQLITE_IO` can propagate cleanly to
   `step()`?
3. What is the minimum interface for the virtual I/O module (open, read page, write page, sync)?
4. Can the paper's tail-latency result be reproduced: one thread per tenant against coroutines
   over a single event loop?
5. What changes once writes are included: write-ahead log, `fsync`, and group commit under
   asynchronous I/O?
6. How should completions be scheduled fairly across tenants in the runtime's event loop?

## 9. Repository layout

```
limbo/
  Cargo.toml
  src/
    main.rs
  README.md
```

Modules will be added as each question in Section 8 is worked on. Each module's documentation
must cite the section of the paper it implements.

## 10. Building

Requires a Rust toolchain with edition 2024 support (Rust 1.85 or later).

```bash
cargo build
cargo run
```

On macOS, if linking fails with "You have not agreed to the Xcode license agreements", run
`sudo xcodebuild -license accept`. The toolchain, not this repository, is the cause.

## 11. Glossary

| Term | Meaning |
|------|---------|
| Edge-native application | An application that requires edge deployment for latency, bandwidth, privacy, availability or data-residency reasons |
| Serverless runtime | A platform that runs short-lived functions for many tenants and manages resource allocation for them |
| Multi-tenancy | Many independent customers' workloads sharing one process or machine |
| VDBE | SQLite's virtual database engine, the bytecode interpreter that executes compiled SQL |
| Pager | The SQLite component that moves pages between disk and the in-memory page cache |
| `io_uring` | A Linux asynchronous I/O interface built on submission and completion ring buffers shared between user space and the kernel |
| Tail latency | High-percentile latency (p99, p99.9): the latency experienced by the slowest requests |
| Co-design | Designing two layers together, with an explicit interface, instead of composing independent black boxes |

## 12. References

Primary source:

- P. Enberg, S. Tarkoma, J. Crowcroft, A. Rao. *Serverless Runtime / Database Co-Design With
  Asynchronous I/O.* EdgeSys '24. https://doi.org/10.1145/3642968.3654821

```bibtex
@inproceedings{enberg2024limbo,
  author    = {Enberg, Pekka and Tarkoma, Sasu and Crowcroft, Jon and Rao, Ashwin},
  title     = {Serverless Runtime / Database Co-Design With Asynchronous I/O},
  booktitle = {Proceedings of the 7th International Workshop on Edge Systems,
               Analytics and Networking},
  series    = {EdgeSys '24},
  year      = {2024},
  pages     = {49--54},
  publisher = {ACM},
  address   = {Athens, Greece},
  doi       = {10.1145/3642968.3654821}
}
```

Background cited by the paper:

- J. Axboe. *Efficient IO with io_uring.* 2019. https://kernel.dk/io_uring.pdf
- K. P. Gaffney et al. *SQLite: Past, Present, and Future.* PVLDB 2022.
- A. Verbitski et al. *Amazon Aurora: Design Considerations for High Throughput Cloud-Native
  Relational Databases.* SIGMOD 2017.
- E. Jonas et al. *Cloud Programming Simplified: A Berkeley View on Serverless Computing.* 2019.
- J. Schleier-Smith et al. *A FaaS File System for Serverless Computing.* arXiv:2009.09845, 2020.
- T. Zhang et al. *Narrowing the Gap Between Serverless and its State with Storage Functions.*
  SoCC 2019.
- G. Tene. *HdrHistogram: A High Dynamic Range Histogram.* http://hdrhistogram.org/
