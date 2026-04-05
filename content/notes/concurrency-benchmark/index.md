---
title: "Designing PostgreSQL Architectures That Scale"
date: 2026-04-05
description: "Benchmarking five concurrency architectures in C++ and Go against PostgreSQL, with TCP latency injection via macOS dummynet"
tags: ["C++", "Go", "Concurrency", "PostgreSQL", "Systems Programming", "Benchmarks"]
showTableOfContents: true
showReadingTime: true
showWordCount: true
---

## Overview

StudentDbPostgress is a multi-level concurrency benchmark comparing five architectures in C++ and Go against PostgreSQL. The project measures how different concurrency models behave under controlled network latency, injected via macOS dummynet on an M2 MacBook Pro (16GB).

I coded Level 1 entirely myself. For subsequent levels, I used Claude for architectural design while writing the implementation code.

## Why This Project Exists

Most concurrency benchmarks measure throughput in ideal conditions. Real systems operate over networks with non-trivial latency. This project injects controlled TCP latency using macOS dummynet to observe how concurrency models degrade under realistic conditions, not just how fast they run in a vacuum.

## Architecture Levels

### Level 1: Single-Threaded Baseline

Sequential request processing. One connection, one query at a time. This establishes the floor for all comparisons. Every millisecond of added latency maps directly to throughput loss with no amortization.

### Level 2: Thread-Per-Connection

Each incoming request spawns a dedicated thread with its own PostgreSQL connection. Simple to implement, expensive to scale. Thread creation overhead and context switching costs become measurable above ~100 concurrent connections.

### Level 3: Connection Pool with Worker Threads

A fixed pool of database connections shared across worker threads. Requests queue when all connections are busy. This isolates the cost of connection management from the cost of concurrency.

### Level 4: Event-Driven (epoll/kqueue)

Non-blocking I/O with an event loop. A single thread multiplexes across many connections using kqueue (macOS). This measures whether eliminating thread overhead compensates for the complexity of async state management.

### Level 5: Hybrid Architecture

Combines an event loop for I/O multiplexing with a thread pool for CPU-bound work. Requests are accepted asynchronously but dispatched to workers for database operations.

## Latency Injection

TCP latency is injected at the OS level using macOS dummynet (via `dnctl` and `pfctl`). This operates below the application layer, so the benchmark code sees the same latency characteristics as a real network hop.
```bash
# Example: inject 5ms latency on loopback traffic to PostgreSQL port
sudo dnctl pipe 1 config delay 5ms
sudo pfctl -e
```

Latency levels tested: 0ms (baseline), 1ms, 5ms, 10ms, 50ms.

## Measurement Methodology

Each level runs a fixed workload of student record CRUD operations against PostgreSQL. Metrics collected per run:

- Throughput (operations/second)
- p50, p95, p99 latency
- CPU utilization
- Memory footprint
- Connection pool saturation (where applicable)

All measurements are repeated across 5 runs per configuration. Results report median values with interquartile ranges.

## Go vs C++ Comparison

Both languages implement the same five levels against the same PostgreSQL instance. The comparison isolates runtime overhead: Go's goroutine scheduler and garbage collector vs C++'s manual memory management and OS threads (or custom thread pools).

Go's goroutines are expected to outperform C++ raw threads at high concurrency counts due to M:N scheduling, but C++ connection pool implementations should close the gap at the architectures where pooling dominates.

## Current Status

Level 1 (single-threaded baseline) is complete for both C++ and Go. Levels 2 through 5 are in design/implementation. Results and Excalidraw architecture diagrams will be added as each level is benchmarked.

## Repository

Source code: [github.com/zarouz/StudentDbPostgress](https://github.com/zarouz/StudentDbPostgress)
