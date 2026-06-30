# Project Loom — Build Plan

**Platform:** Cross-platform (Linux + Windows)  
**Structure:** Multi-project solution  
**Runtime:** .NET 10 · Native AOT

---

## Solution layout

```
Loom.sln
├── Loom.SSH/        ← Phase 1: SSH server, ANSI parser
├── Loom.TUI/        ← Phase 2: Spectre.Console render engine
├── Loom.Core/       ← Phase 3: SIMD math, telemetry search
├── Loom.Storage/    ← Phase 4 + 5: binary cache, RAG ingestor
└── Loom.Host/       ← Entry point: wires all four together
```

`Loom.Host` is a thin bootstrap project — starts the SSH listener, hands connections to the TUI, and owns the `IHostedService` lifetime. All other assemblies are independently AOT-compilable and testable.

---

## Phase sequence

Phases must be built in order — each one is a dependency of the next.

---

### Phase 1 — SSH infrastructure
**Project:** `Loom.SSH`

**Goal:** Establish a secure, high-throughput network tunnel with a zero-allocation terminal input parser.

**Key tech:**
- `System.Net.Sockets` — `SocketAsyncEventArgs` for async non-blocking I/O
- `System.Formats.Asn1` + `.NET Cryptography` — embedded SSH state machine
- `ReadOnlySpan<byte>` · `SequenceReader<byte>` — in-place byte stream parsing
- Native AOT — single self-contained binary < 15 MB, < 20 MB RAM footprint

**Deliverable:** Functional SSH server on port `2222` that accepts remote connections and streams parsed, allocation-free terminal input events.

---

### Phase 2 — TUI render engine
**Project:** `Loom.TUI`

**Goal:** Immersive, low-latency display system with smooth differential frame updates over SSH.

**Key tech:**
- `Spectre.Console.Live` — real-time nested tables, panels, and diagnostic gauges
- `AnsiConsole.Live` / `Layout` / `Panel` — composable TUI layout
- SIGWINCH resize detection — instant layout reflow on terminal resize
- Differential frame push — compressed updates over the SSH channel

**Deliverable:** Responsive dashboard with real-time CPU meters, thread panels, and "Digital Surge" transition animations running over SSH.

---

### Phase 3 — SIMD math engine
**Project:** `Loom.Core`

**Goal:** Sub-microsecond search and scoring over telemetry vectors and thread activity data.

**Key tech:**
- `System.Numerics.Vector<float>` — portable SIMD abstraction
- `Vector128<float>` / `Vector256<float>` — AVX2/SSE hardware intrinsics
- Contiguous aligned memory blocks — cache-friendly data layout
- Scalar tail loop — correct handling of remainder elements

**Deliverable:** Telemetry search subsystem that indexes thread behaviours and filters millions of metric records in microseconds (10–20× speedup over scalar loops).

---

### Phase 4 — Binary embedding cache
**Project:** `Loom.Storage`

**Goal:** Eliminate startup delays by memory-mapping pre-processed diagnostic indices.

**Key tech:**
- `System.IO.MemoryMappedFiles.MemoryMappedFile`
- `StructLayout(LayoutKind.Sequential, Pack = 1)` — layout-aligned binary records
- `ReadOnlySpan<T>` cast over native pointer — zero-copy file access
- `Unsafe` / `Marshal.SizeOf<T>` — struct size computation

**Deliverable:** Startup sequence that loads thousands of pre-processed symbol mappings and metric vectors in milliseconds, with no JSON decode cost.

---

### Phase 5 — RAG telemetry ingestor
**Project:** `Loom.Storage`

**Goal:** Transform nested JSON diagnostic outputs into context-rich sentences for vector search.

**Key tech:**
- `Utf8JsonReader` — streaming, zero-allocation JSON traversal
- Recursive path-context concatenation — retains parent–child structural relationships
- Output feeds Phase 3's vectorizer for semantic diagnostic search

**Deliverable:** Ingestion engine that converts raw diagnostic outputs (thread dumps, system profiles, work logs) into descriptive sentences optimised for search accuracy.

---

## Cross-platform notes

- `SocketAsyncEventArgs` works identically on Linux and Windows.
- Phase 4's `AcquirePointer` + `unsafe` cast requires `<AllowUnsafeBlocks>true</AllowUnsafeBlocks>` in each affected `.csproj`.
- `/proc` telemetry reads (Linux) need a `RuntimeInformation.IsOSPlatform(OSPlatform.Linux)` branch; use .NET EventPipe APIs as the Windows fallback.

---

## Production security notes

- Loom runs in a separate network namespace from the target application.
- SSH channel dispatchers permit only predefined Loom telemetry command keys — no interactive shell.
- Telemetry queries automatically back off when target CPU exceeds 85% to prevent service degradation.
