# General Performance FAQ

Backend-independent performance and diagnostics guidance for running **Senzing v4** under
high-throughput load. This is the common material that applies no matter which database you use.
For backend-specific tuning, see the companion repositories:

- [mssql-performance-v4](https://github.com/Senzing/mssql-performance-v4) — Microsoft SQL Server
- [postgresql-performance-v4](https://github.com/Senzing/postgresql-performance-v4) — PostgreSQL

Most of what follows is about **finding the bottleneck** and **what to collect for Senzing Support**,
because at scale the answer is almost always in your database, your network, or your data — not in a
knob on the engine.

## How does Senzing scale?

In your application/service you initialize **one `SzEngine` per process**, then share that one engine
across all the threads in the process. Senzing automatically creates a "context" — including database
connections — for each thread that calls engine functions. The engine has no internal thread pool of
its own.

What this means:

1. **You need real OS threads.** Cooperative concurrency — Python `gevent`/`asyncio`, Go goroutines,
   Java virtual threads, etc. — will not scale here. Those multiplex work onto a small number of OS
   threads, so you cannot parallelize past the number of OS threads actually running. Use real threads.
2. **Don't constantly create and destroy threads.** Because each thread lazily creates its own context
   (and database connections), thread churn is expensive. Use a thread pool. A pattern like a web
   framework that spins up and tears down a thread per request is a poor fit.
3. **Scale out with more processes, not just more threads.** The engine is a "share nothing but the
   database" architecture, so it scales until your database or network can't take more work. But there
   is a second reason to prefer processes: the **database client driver-manager can serialize calls
   within a single process**. Under a large thread count, threads can end up queuing on a
   process-global lock inside the client library rather than doing real database work. Each *process*
   gets its own client environment (its own driver-manager, its own connection pool), so adding
   processes adds real concurrency that adding threads to one process may not. The practical rule:
   **one engine per process; to go faster, add processes.**

The one thing that breaks the "scales until the database does" rule is **data contention** — many
records trying to resolve into the *same* entity at the same time. No amount of hardware fixes that;
it is a property of the data arriving together. The engine's own statistics (below) will tell you if
you have it.

## Validate the basics of the setup

Before chasing anything subtle, confirm the datastore itself performs. The SDK exposes a diagnostic
that runs a pure single-thread insert-latency test against the repository:

```text
# From the SDK — SzDiagnostic.checkRepositoryPerformance (verified v4 API):
#   Python:      env.get_diagnostic().check_repository_performance(3)
#   Java/C#/TS:  env.getDiagnostic().checkRepositoryPerformance(3)
diagnostic = env.getDiagnostic()
result = diagnostic.checkRepositoryPerformance(3)   # run the latency test for 3 seconds
```

It reports how many inserts a single thread completed in the interval — a pure latency test, with no
concurrency and no entity-resolution work. As a rough sanity baseline, a reasonably configured
environment should manage on the order of several thousand inserts in a 3-second window (a well-tuned
setup comfortably exceeds ~9000). If this number is low, stop here — the problem is database/network
latency, and no engine tuning will paper over it.

## High-level stats — the first thing to collect (and send to Support)

The single most useful thing you can do while developing your own application or service is to
**periodically capture the JSON document returned by `SzEngine.getStats()`** (say, every 5 minutes)
and keep it. It returns the engine's internal accounting of how much work it is doing — with **no
clear-text data** — accumulated across all threads in the process, and it resets each time you call it.

**Send the whole document to Senzing Support.** Eight times out of ten, when there is a performance
issue, Support can tell you what is going on just from this.

> [!IMPORTANT]
> These internal fields are **not generally documented and can change without notice** — do not build
> tooling that depends on their exact names or shape. Capture the **entire** `workload` object over a
> window and send it. The annotations below illustrate the *kinds* of signals to look for; treat the
> field names as a guide, not a contract, and confirm against your own version's output.

A few signals in the `workload` object are commonly worth a glance yourself:

```text
stats = json.loads(engine.getStats())   # accumulates all threads in the process; resets on each call

{ "workload": {
    "apiVersion": "4.x...",
    ...
    "retries": 27,             // general contention — the smaller the better
    ...
    // --- these three being non-empty means records that resolve into the SAME entity are arriving
    //     together; shuffling / randomizing the input order is the usual fix ---
    "latchContention":      [ ],   // feature types where contention was caught before hitting the DB — empty is ideal
    "highContentionFeat":   [ ],   // specific hot feature hashes that are problematic — should be empty
    "highContentionResEnt": [ ],   // specific resolved-entity IDs causing contention — should be empty
    ...
    // --- threadState is a SNAPSHOT at the instant getStats() was called; look for TRENDS across samples ---
    "threadState": {
      "active":       126,     // threads currently inside an engine call (e.g. addRecord)
      "idle":           3,     // threads NOT in a call — should be near zero under heavy load; if it isn't,
                               //   you aren't feeding your threads fast enough (bottleneck is UPSTREAM:
                               //   input, queue, network — not the engine or database)
      "sqlExecuting": 117      // threads executing SQL / fetching results = waiting on the database. Points
                               //   you at the DB and network — but read the holistic caution below before
                               //   concluding it's the database *server*.
    }
} }
```

## When you're database- or contention-bound: what to collect

> [!IMPORTANT]
> **Senzing v4 ships stripped libraries with no build ID.** Unlike earlier versions, you **cannot** get
> a meaningful symbolized stack trace out of a running v4 engine — a debugger or profiler attached to
> the process shows unresolved `?? ()` frames that cannot be attributed to a function or a source line.
> So the old "attach gdb and read the Senzing call stack" technique **does not apply to v4**. Diagnose
> with the signals below instead — the engine's own workload counters, the database's wait/contention
> statistics, and host telemetry. None of those need symbols, and together they tell you the same story
> the stack used to, more reliably.
>
> **You can still send the raw stacks to Senzing Support.** You can't resolve the `?? ()` frames
> yourself, but Support can — provided you include the exact **Senzing version and platform**
> (OS/architecture) the process was running, so they can match the addresses against that build's
> symbols. A `gdb -p <pid> -batch -ex "thread apply all bt"` dump from a stuck/slow process is still
> worth capturing; just hand it to Support together with the version and platform.

Work the signals in this order and **correlate everything by wall-clock time** across the app host(s)
and the database host.

### 1. Host telemetry (app hosts *and* the database host)

Collect basic OS telemetry on every host in the run — the application/loader hosts and the database
server — over the same time window (`sar`, `mpstat`, `iostat`, run-queue length, `iowait`):

- **CPU idle vs. busy**, per host. Which tier is actually saturated — the app hosts doing engine work,
  or the database server?
- **iowait and disk latency/utilization** on the database host.
- **Run-queue length** (are there more runnable threads than CPUs?).

The decisive triage rule: **if the database CPU is mostly idle, disk latency is low, and the top waits
are locks or latches, then the bottleneck is contention, not resources** — adding CPU, memory, or IO
will not help; you have to reduce the contention (or the work that causes it).

> [!NOTE]
> A busy CPU is not proof of useful work. A fully saturated user-CPU percentage cannot, by itself,
> distinguish "doing real work" from "spinning on backpressure or a lock." Always read CPU **together
> with throughput and idle** — the two in combination are what reveal whether the host is productive.

### 2. Database wait / contention statistics

This is the primary triage once you know a tier is database-bound. Capture the database's wait
statistics as a **delta over a short window** (snapshot → wait ~30-60s → snapshot → subtract), not as a
cumulative total, and rank the waits by the time spent in them. The backend-specific queries live in
the companion repos; the cross-backend shape is:

- **PostgreSQL** — sample the live wait-event distribution from `pg_stat_activity` (active client
  backends, grouped by wait event) and read IO efficiency/latency from `pg_stat_io`. A wait
  distribution dominated by **data-file reads** means cache-miss random reads — the working set has
  outgrown the buffer pool; the answer is fewer reads per record and more RAM, not a faster CPU.
- **Microsoft SQL Server** — rank `sys.dm_os_wait_stats` by delta wait time, and use
  `sys.dm_db_index_operational_stats` to attribute a lock or latch wait to a **specific index** (a
  per-statement view cannot do that). Page-latch waits with a huge task count are last-page insert
  contention on an ascending key; `LCK_*` waits are row/page lock contention.

Whatever the backend, the goal is the same: identify **which resource** the engine's threads are
waiting on, then decide whether it's a resource limit (add hardware) or a contention hotspot (change
configuration, index, or data ordering). Follow the companion repo for that backend to turn the wait
you found into a fix.

### 3. Cross-check against the engine's workload stats

Line the database picture up against the `getStats()` workload document from the same window. If the
engine reports most threads **executing SQL** while the database server is **idle**, the limit is in
the round trip — the client library, the network, or per-record serialization — not the database
server's capacity. That is exactly the case where **adding processes** (each with its own client
environment) buys you concurrency that adding threads does not.

## Don't jump to conclusions — take a holistic view

When the signals say your threads are "waiting on the database," that only means the delay is somewhere
on the path **to** the database — it could be the vendor's database client library, the network, or the
database server itself. It does **not** by itself mean the database server is out of capacity, and it
does **not** mean the database can't handle more load from more threads or processes. For example, some
prebuilt database client driver-managers are compiled with a process-global lock and cannot scale
across threads within one process at all — in that case a "waiting on the database" reading has nothing
to do with your network or your database server, and the fix is more *processes*, not a bigger server.
Always corroborate across all three layers (host telemetry, database waits, engine workload stats)
before spending money on hardware.

## A note on counters: cost-weight before you chase

It is easy to find a counter that is changing a lot and assume it's the problem. A counter delta is
only a verdict if it is **cost-weighted** — establish what share of the contended resource's *time* the
thing actually accounts for before you act on it. A round-trip count that falls by 90% but was only a
percent or two of total CPU will not move throughput. Measure the share first; chase the counter
second.

## What to send to Senzing Support

When you open a performance case, collect these over the **same wall-clock window** and send them
together — it is the fastest path to an answer:

1. The **`getStats()` workload JSON**, sampled periodically across the run (the whole document each
   time — do not trim it).
2. A **database wait-statistics** snapshot over a short window (delta, ranked by time) — see the
   companion repo for the exact query for your backend.
3. **Host telemetry** (CPU idle/busy, iowait, run-queue) from the application host(s) **and** the
   database host, same window.
4. Your **throughput** number and how you measured it (measure sustained end-to-end throughput at the
   point work is acknowledged as done, not an instantaneous peak).
5. Your **configuration**: the exact **Senzing version and platform** (OS/architecture — Support needs
   these to symbolize any stack dumps you include), which database backend, the relevant engine
   settings, and how many **processes and threads** you are running.
6. **Optional, but valuable when a process is stuck or slow:** a `gdb` all-thread backtrace dump
   (`gdb -p <pid> -batch -ex "thread apply all bt"`). The frames will be unsymbolized `?? ()` — send it
   anyway, with the version and platform above; Support symbolizes it against the matching build.

## Reference

- [mssql-performance-v4](https://github.com/Senzing/mssql-performance-v4) — SQL Server tuning
- [postgresql-performance-v4](https://github.com/Senzing/postgresql-performance-v4) — PostgreSQL tuning
