# Systems Design Roadmap → Multi-Strategy Hedge Fund Platform

**Stack:** Python (research/data plane) · SQL (storage/analytics) · Rust (execution/latency plane, with C++ as a conceptual bridge) · AWS (infrastructure)
**Target system:** a platform with market data ingestion, research/backtesting, risk management, and order execution — the four pillars of a real multi-strategy fund's tech stack.

Starting point assumed: solid-ish Python/SQL, new to Rust, relearning C++ fundamentals.

---

## How to use this roadmap

Each phase has: **concepts** (systems design theory), **resources**, and a **demo project** that forces you to apply the concepts immediately. Don't finish all the theory before starting a project — read enough to be dangerous, build, then go back and read deeper once you hit a wall. That's how systems design is actually learned in industry.

Total timeline: ~9-12 months at a serious part-time pace (10-15 hrs/week). Compress or stretch as needed.

---

## Phase 0 — Foundations Refresh (2-3 weeks)

**Goal:** get SQL and Python to "systems" level (not just scripting), and get just enough C++/Rust vocabulary to not be lost later.

### Concepts
- Python: async/await (`asyncio`), multiprocessing vs threading and the GIL, memory model, `dataclasses`/`pydantic` for schema discipline
- SQL: indexes (B-tree vs hash), query planning/`EXPLAIN`, transactions & isolation levels, window functions, normalization vs denormalization tradeoffs
- C++/Rust bridge: stack vs heap, ownership (Rust's borrow checker is a formalized version of what disciplined C++ RAII does manually), why these languages exist for latency-sensitive systems (no GC pause)

### Resources
- *Fluent Python* (Ramalho) — for the async/concurrency chapters specifically
- *Use The Index, Luke* (free, use-the-index-luke.com) — best short resource on how indexes/query planners actually work
- *The Rust Programming Language* ("the book," free at doc.rust-lang.org/book) — chapters 1-6 (ownership, structs, enums) only for now
- PostgreSQL official docs — "Query Planning" section

### Demo project
**P0: Tick data replay tool.** Write a Python script that reads a CSV of historical OHLCV data, loads it into Postgres with a proper schema (symbol, timestamp, OHLCV, partitioned by date), and replays it at configurable speed via an `asyncio` generator that yields ticks. This becomes the data source for later projects. Add indexes and use `EXPLAIN ANALYZE` to verify your queries by symbol+date range are actually using them.

---

## Phase 1 — Systems Design Fundamentals (4-5 weeks)

**Goal:** the vocabulary and mental models used in every real distributed system design — latency, throughput, consistency, replication, partitioning, caching, queuing.

### Concepts
- Latency numbers every engineer should know; percentiles (p50/p99/p999) vs averages
- CAP theorem and PACELC (more useful in practice)
- Consistency models: strong, eventual, causal
- Replication (leader-follower, quorum), partitioning/sharding strategies
- Caching: write-through/write-back, cache invalidation, TTL strategies
- Message queues vs event streams (queue semantics vs log semantics — this distinction matters a lot for market data)
- Load balancing, rate limiting, backpressure
- Idempotency and exactly-once vs at-least-once delivery (**critical for anything touching order execution**)

### Resources
- *Designing Data-Intensive Applications* (Kleppmann) — the single best book for this phase; read Ch 1-9 carefully, they're the ones that matter most here
- ByteByteGo (Alex Xu) — *System Design Interview Vol 1 & 2* for pattern recall, and the free YouTube channel for visual explanations
- "Latency Numbers Every Programmer Should Know" — search for the current gist, it's a living reference
- AWS Well-Architected Framework (free, aws.amazon.com/architecture/well-architected) — Reliability and Performance Efficiency pillars

### Demo project
**P1: Event-driven order book simulator (single machine).** Build an in-memory limit order book in Python that consumes a stream of synthetic order events (add/cancel/match) from a queue (use `asyncio.Queue` first, then swap to Redis Streams). Implement idempotent event processing (dedupe by event ID) and measure p50/p99 latency of processing one event end-to-end. This is your first taste of the exactly-once problem that will haunt every later project.

---

## Phase 2 — SQL & Data Engineering at Scale (3-4 weeks)

**Goal:** move from "SQL user" to "person who designs the storage layer of a trading system." Time-series data is the core dataset of a fund, and it has very specific access patterns.

### Concepts
- Time-series data modeling: wide vs narrow tables, downsampling, retention policies
- OLTP vs OLAP; when you need a columnar store (for backtesting/analytics) vs row store (for order state)
- Partitioning strategies for time-series (by day/symbol), and why this matters for query performance and vacuum/maintenance cost
- Write amplification, and why naive high-frequency tick inserts kill a normal Postgres instance
- CDC (change data capture) as a pattern for keeping multiple stores in sync

### Resources
- TimescaleDB docs (time-series Postgres extension) — genuinely excellent docs, read the "best practices" section
- *SQL Performance Explained* (Winand) — short, dense, worth the read
- AWS Timestream docs, and AWS re:Invent talks on time-series workloads (search "AWS re:Invent Timestream" for current talks)
- QuestDB blog — they write very good technical content on tick data storage specifically, even if you don't use their DB

### Demo project
**P2: Tick data warehouse.** Stand up TimescaleDB (or AWS Timestream) and migrate your P0 replay tool to write into it at realistic tick volume (simulate 10k+ events/sec). Design a schema with proper partitioning/chunking. Build a set of analytical queries a quant would actually run: VWAP by symbol/day, rolling volatility, top-N movers. Compare query performance against your naive Postgres table from Phase 0.

---

## Phase 3 — Python for the Research & Backtesting Plane (4-5 weeks)

**Goal:** the system that lets you test strategies before they touch real capital. This is where correctness matters more than speed.

### Concepts
- Event-driven vs vectorized backtesting architectures (and why event-driven avoids lookahead bias)
- Point-in-time correctness — the single most important and most-violated principle in backtesting (never let a strategy see data it wouldn't have had at that timestamp)
- Distributed task execution for parameter sweeps (this is your first real "distributed systems" application)
- Reproducibility: seeding, versioning data snapshots, versioning strategy code alongside results

### Resources
- *Advances in Financial Machine Learning* (López de Prado) — the chapters on backtesting pitfalls (esp. backtest overfitting) are essential reading regardless of whether you use ML
- *Algorithmic Trading: Winning Strategies and Their Rationale* (Ernest Chan) — practical, not academic
- Backtrader / Zipline-reloaded source code — read, don't necessarily use as-is; understanding how a real event-driven backtester is architected is the point
- Ray or Dask docs — for distributing backtests across workers

### Demo project
**P3: Event-driven backtesting engine.** Build your own (don't just import one — you learn the systems design by building it) event-driven backtester in Python that consumes your Phase 2 tick warehouse, simulates order fills with realistic slippage/latency assumptions, and enforces point-in-time correctness (strategy code physically cannot query future data). Then use Ray/Dask to parallelize a parameter sweep across AWS EC2 spot instances — this is your first cloud-distributed compute project.

---

## Phase 4 — Rust for the Execution & Latency Plane (5-6 weeks)

**Goal:** the part of the system where microseconds matter — order management and execution. This is also where your C++ relearning pays off conceptually even though you're writing Rust.

### Concepts
- Ownership/borrowing deeply (not just syntax — understand *why* it eliminates data races at compile time)
- Zero-cost abstractions, and why Rust can match C++ performance without a GC
- Lock-free/wait-free data structures basics (ring buffers, SPSC/MPSC queues) — foundational for order books and market data handling
- Async Rust (`tokio`) vs OS threads — when each is appropriate for I/O-bound vs CPU-bound work
- FFI: calling Rust from Python (this is how real quant shops actually ship Rust — as a fast core under a Python interface)

### Resources
- *The Rust Programming Language* (finish it this time, especially concurrency chapter)
- *Rust for Rustaceans* (Jon Gjengset) — once past basics, this is the best "systems-level Rust" book
- Jon Gjengset's YouTube channel — live-codes a lock-free data structure and an actor system, extremely relevant
- `PyO3` docs — for Rust↔Python bindings
- `tokio` docs and the "Tokio tutorial"

### Demo project
**P4a: Rust order book engine.** Port your Phase 1 order book simulator to Rust. Use a lock-free ring buffer for the event queue. Benchmark p50/p99/p999 latency against the Python version — this comparison is the whole point of the exercise; you should see 10-100x tail latency improvement.

**P4b: Python↔Rust bridge.** Wrap the Rust order book as a Python extension module with PyO3, so your Phase 3 backtester can call into it as the execution simulator. This mirrors how real quant infra is layered: Python for research velocity, Rust for the hot path.

---

## Phase 5 — AWS Cloud Architecture (4-5 weeks)

**Goal:** deploy the system as a real distributed, fault-tolerant platform, not a laptop demo.

### Concepts
- VPC design, security groups, private subnets for anything touching order flow
- Managed streaming: Kinesis (or MSK/Kafka) for market data ingestion at scale
- Compute choices: EC2 (for latency-predictable execution), Lambda (for event-driven/bursty work), ECS/EKS (for the backtesting fleet)
- Storage tiering: hot (Timestream/RDS), warm (S3 + Athena for historical analytics), cold (S3 Glacier)
- IAM least-privilege design — non-negotiable once you're handling anything resembling real capital
- Observability: CloudWatch metrics/alarms, distributed tracing (X-Ray), structured logging
- Disaster recovery basics: multi-AZ, RTO/RPO concepts (a fund's execution system going down mid-position is a real financial risk, design for it)

### Resources
- AWS Well-Architected Framework (all 6 pillars now, not just the two from Phase 1)
- *AWS Certified Solutions Architect* study guide (even if you don't sit the exam, the curriculum is a very good forced tour of the services you need) — check current guide via AWS's own training pages
- "AWS Streaming Data Solutions for Financial Services" — search AWS's whitepapers page, they publish reference architectures specifically for this
- Kinesis Data Streams docs, and the MSK (managed Kafka) docs for comparison

### Demo project
**P5: Cloud-deployed data + backtest pipeline.** Move Phase 2/3 to AWS: market data ingestion through Kinesis → Lambda consumers writing to Timestream/S3; backtesting fleet as an ECS or Batch job pool triggered on-demand; results and logs to S3 + CloudWatch. Add basic IAM roles per component (ingestion service cannot write orders; backtester cannot read production credentials, etc.). This is the project that teaches you cost/latency/reliability tradeoffs under a real cloud bill.

---

## Phase 6 — Capstone: Integrated Multi-Strategy Platform (8-12 weeks)

**Goal:** tie every phase together into one coherent system with the four pillars of a real fund's stack.

### Architecture to build
```
[Market Data Feed] → Kinesis → [Rust ingestion/normalization] → Timestream/S3
                                        ↓
                          [Python research/backtest cluster]  (Ray on ECS/Batch)
                                        ↓
                          [Strategy signals] → [Risk engine (Python, position limits, VaR)]
                                        ↓
                          [Rust order management + execution engine]
                                        ↓
                          [Paper-trading broker API / simulated exchange]
                                        ↓
                          [Postgres/Timescale: positions, fills, PnL] → dashboards
```

### Concepts to layer in specifically for the capstone
- Risk management as a system: position limits, kill switches, pre-trade risk checks (these must be synchronous and fast — a systems design problem in themselves)
- Saga pattern / compensating transactions for multi-step order workflows that can partially fail
- Circuit breakers between components (if risk engine is down, does execution fail open or closed? — this is a real design decision with financial consequences)
- Chaos testing basics: kill a component mid-flow and verify the system recovers into a consistent state

### Resources
- *Building Reliable and Scalable Systems* — search for current O'Reilly/free equivalents; also revisit DDIA Ch 12 (the future of data systems / correctness) now that you have a real system to apply it to
- FIX protocol docs (quickfixengine.org) — even a paper system benefits from modeling order messages on the real industry standard
- Interactive Brokers or Alpaca paper trading API docs — for a realistic "broker" to execute against without real capital

### Deliverable
A running system, deployed on AWS, that: ingests simulated or real delayed market data, runs a strategy through backtest, promotes it to paper trading, routes orders through the Rust execution engine with pre-trade risk checks, and reconciles positions/PnL into a dashboard. This is genuinely close to a v0 of real fund infrastructure — and a very strong portfolio piece if you ever need to demonstrate technical credibility to investors or hires.

---

## Ongoing / parallel track: fund-specific knowledge

Systems design is necessary but not sufficient — worth running in parallel, not sequentially:
- Regulatory/compliance basics for the jurisdiction you'd launch in (this shapes system requirements — audit logging, trade reporting, recordkeeping — more than any technical resource will)
- Market microstructure (Harris, *Trading and Exchanges*) — informs your order book and execution engine design choices directly
- Operational due diligence expectations — institutional allocators will eventually ask about your infrastructure's reliability, DR, and audit trail; building Phase 6 with that lens in mind pays off later

---

## Suggested pacing summary

| Phase | Weeks | Focus |
|---|---|---|
| 0 | 1-3 | Python/SQL/Rust-C++ foundations refresh |
| 1 | 4-8 | Systems design fundamentals |
| 2 | 9-12 | SQL/time-series storage |
| 3 | 13-17 | Python backtesting engine |
| 4 | 18-23 | Rust execution engine |
| 5 | 24-28 | AWS deployment |
| 6 | 29-40 | Integrated capstone |

Adjust freely — the projects are cumulative by design, so slipping a phase just shifts the whole line, it doesn't break anything downstream.
