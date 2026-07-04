# Project Loom: Architectural Blueprint & Phase-by-Phase Plan
## The "X-Ray" Telemetry Platform for Running Apps (.NET 10 / Spectre.Console / Custom SSH)

Project **Loom** is a lightweight, low-overhead, real-time diagnostic terminal companion designed to attach directly to production applications. Built for developers and site reliability engineers (SREs), Loom provides instant "stethoscope-like" insight into CPU hotpaths, memory allocations, and thread blockages through a secure, remote terminal interface (TUI) over an embedded, custom SSH server.

---

## Why Modern C# & .NET 10?

For a secure, production-grade diagnostic agent that must execute with absolute minimal impact on the target system, **C# on .NET environment** represents the state-of-the-art:

1. **AOT (Ahead-Of-Time) Compilation**: .NET environment supports Native AOT compilation, allowing Loom to compile into a single self-contained, dependency-free binary (< 15MB) with sub-millisecond startup times and ultra-low background memory footprints (< 20MB).
2. **Memory Safety & Low-Allocation Primaries**: Through `Span<T>`, `ReadOnlySpan<T>`, and stack-only `ref struct`s, we can process high-throughput network streams and telemetry metrics with virtually zero allocations, avoiding Garbage Collector (GC) pauses.
3. **SIMD & Hardware Intrinsics**: Direct access to SIMD vectors (via `System.Numerics.Register`) provides hardware-accelerated numeric math, crucial for low-latency searches or telemetry log processing.
4. **Secure Core**: Strongly typed memory, lack of raw pointers by default, safe array bounds-checking, and cross-platform native cryptographies make it hardened against remote code execution vulnerabilities common in native wrappers.

---

## Core System Architecture

```
                                  [ SSH Client Terminal ]
                                             │
                       SSH Channel           ▼  (TCP/Port 2222)
               ┌──────────────────────────────────────────────────────────┐
               │  Embedded Custom SSH Server                              │
               │                                                          │
               │  ┌──────────────────────┐      ┌──────────────────────┐  │
               │  │  Zero-Allocation     │ ───► │  Spectre.Console TUI │  │
               │  │  ANSI Parsing Input  │      │  "Digital Surge" UX  │  │
               │  └──────────────────────┘      └──────────────────────┘  │
               └──────────────────────────────────────────────────────────┘
                                             ▲
                                             │ Core Telemetry Engine
                                             │
               ┌──────────────────────────────────────────────────────────┐
               │  Diagnostic Host Agent (Target Application Sandbox)      │
               │                                                          │
               │  ┌──────────────────────┐      ┌──────────────────────┐  │
               │  │  SIMD Math Engine    │      │  High-Context RAG    │  │
               │  │  (Vector Search)     │      │  Telemetry Ingestor  │  │
               │  └──────────────────────┘      └──────────────────────┘  │
               │                                           ▲              │
               │                                           │ (Disk/Cache) │
               │                                ┌──────────────────────┐  │
               │                                │  Binary Embedding    │  │
               │                                │  Cache (MMap)        │  │
               │                                └──────────────────────┘  │
               └──────────────────────────────────────────────────────────┘
```

---

## Phase-by-Phase Methodological Implementation Plan

### Phase 1: Embedded SSH Infrastructure & Zero-Allocation Input Parser
*Goal: Establish a secure, high-throughput network tunnel capable of parsing raw keystrokes directly into memory buffers without memory pressure.*

* **Objective**: Replace standard SSH shell processes with a streamlined, memory-pooled SSH listener. Establish a zero-allocation pipeline for terminal ANSI keyboard event codes.
* **Mechanism**:
  - Implement a raw Socket-based listener using `SocketAsyncEventArgs` for asynchronous, non-blocking network I/O.
  - Integrate an embedded SSH state machine utilizing `System.Formats.Asn1` and `.NET Cryptography` to authenticate clients and manage SSH channels.
  - Parse byte-streams in place using `ReadOnlySpan<byte>` and `SequenceReader<byte>`. This avoids allocating `string` objects for arrow keys, window resizes (`SIGWINCH`), or escape sequences.
* **Key Deliverable**: A functional SSH server that accepts remote connections on port `2222` and streams parsed, allocation-free terminal inputs.

---

### Phase 2: High-Energy UX & Console Live Engine
*Goal: Create an immersive, low-latency display system that updates smoothly over remote terminal connections.*

* **Objective**: Implement a high-refresh rendering loop that pushes compressed differential frame updates to the client console.
* **Mechanism**:
  - Utilize `Spectre.Console.Live` to coordinate real-time updates of nested tables, layout panels, and diagnostic gauges.
  - Develop a **"Digital Surge" transition system**: on user actions, render elegant transition waves across the terminal canvas using procedural text modifications.
  - Track terminal screen sizes dynamically to adjust layouts instantly without throwing rendering exceptions or corrupting terminal character grids.
* **Key Deliverable**: A responsive dashboard containing real-time CPU meters, active thread panels, and smooth frame-refresh animations running directly over the SSH session.

---

### Phase 3: Hardware-Accelerated Diagnostic Search(SIMD Math)
*Goal: Ensure search queries over massive thread lists, performance logs, or diagnostic metadata occur with sub-microsecond latency.*

* **Objective**: Use hardware vectorization (SIMD) to scan, filter, and score telemetry data or diagnostic log signatures instantly.
* **Mechanism**:
  - Store thread activity scores and telemetry vector representations in aligned contiguous blocks of memory.
  - Implement Dot Product and Cosine Similarity computations using `System.Numerics.Vector<float>` or `Vector128<float>` / `Vector256<float>`.
  - Process 8 to 16 floating-point values in a single CPU instruction, achieving a 10x-20x speedup over standard scalar loops.
* **Key Deliverable**: A telemetry search sub-system that indexes thread behaviors and filters millions of metric records in microseconds.

---

### Phase 4: Binary Embedding Cache for Fast Initialization
*Goal: Remove startup delays by persisting heavy indexing databases or configuration trees into highly optimized, native memory-mapped structures.*

* **Objective**: Implement a custom binary file format that can be loaded straight into memory-mapped structures, allowing instant access to diagnostic indices.
* **Mechanism**:
  - Serialize vector embedding structures, application symbols, and diagnostic rules directly as raw, layout-aligned binary records.
  - At startup, load this cache file using `System.IO.MemoryMappedFiles.MemoryMappedFile`.
  - Treat the file contents directly as raw native structures (`ReadOnlySpan<byte>`), mapping the pointers directly to managed types. This reduces startup times from seconds of JSON decoding down to absolute millisecond disk map speeds.
* **Key Deliverable**: A startup sequence that instantly loads thousands of pre-processed symbol mappings and metric vectors without parsing files.

---

### Phase 5: High-Context RAG Telemetry Ingestion
*Goal: Transform complex nested JSON structures (application configs, thread dumps, environmental logs) into highly structured, context-rich text ready for diagnostics.*

* **Objective**: Create a fast, recursive flattener that processes complex execution details into plain-text sentences containing the structural context of the call.
* **Mechanism**:
  - Write a recursive `Utf8JsonReader` loop that walks through nested JSON outputs (e.g., system profiles, dependency lists, work logs) without building complete JSON object models in memory.
  - Concatenate paths into descriptive sentences (for example: `"The background queue service is in an Active state with 15 blocked threads."`).
  - Feed these context-rich sentences to the diagnostic vectorizer, ensuring the semantic relationships of parent-child structures are fully retained.
* **Key Deliverable**: An ingestion engine that turns raw diagnostic outputs into descriptive sentences, optimizing the search accuracy of your diagnostics.

---

## Technical Specifications & Deep Dives

### 1. Zero-Allocation ANSI Input Parser
```csharp
using System;

namespace Loom.Core.Network;

public readonly ref struct TerminalInputEvent
{
    public TerminalKey Key { get; }
    public ReadOnlySpan<char> RawBuffer { get; }

    public TerminalInputEvent(TerminalKey key, ReadOnlySpan<char> rawBuffer)
    {
        Key = key;
        RawBuffer = rawBuffer;
    }
}

public enum TerminalKey
{
    Unknown,
    ArrowUp,
    ArrowDown,
    ArrowLeft,
    ArrowRight,
    Escape,
    Enter,
    Backspace
}

public static class AnsiParser
{
    // High-performance parser that reads directly from the raw SSH input stream span
    public static TerminalInputEvent Parse(ReadOnlySpan<byte> source)
    {
        if (source.Length == 0) return new TerminalInputEvent(TerminalKey.Unknown, ReadOnlySpan<char>.Empty);

        // Map escape sequences directly without string allocations
        if (source[0] == 0x1B) // ESC character
        {
            if (source.Length >= 3 && source[1] == 0x5B) // ESC [
            {
                switch (source[2])
                {
                    case 0x41: return new TerminalInputEvent(TerminalKey.ArrowUp, ReadOnlySpan<char>.Empty);
                    case 0x42: return new TerminalInputEvent(TerminalKey.ArrowDown, ReadOnlySpan<char>.Empty);
                    case 0x43: return new TerminalInputEvent(TerminalKey.ArrowRight, ReadOnlySpan<char>.Empty);
                    case 0x44: return new TerminalInputEvent(TerminalKey.ArrowLeft, ReadOnlySpan<char>.Empty);
                }
            }
            return new TerminalInputEvent(TerminalKey.Escape, ReadOnlySpan<char>.Empty);
        }

        if (source[0] == 0x0D || source[0] == 0x0A) return new TerminalInputEvent(TerminalKey.Enter, ReadOnlySpan<char>.Empty);
        if (source[0] == 0x7F || source[0] == 0x08) return new TerminalInputEvent(TerminalKey.Backspace, ReadOnlySpan<char>.Empty);

        return new TerminalInputEvent(TerminalKey.Unknown, ReadOnlySpan<char>.Empty);
    }
}
```

### 2. SIMD-Accelerated Math Engine
```csharp
using System;
using System.Numerics;
using System.Runtime.Intrinsics;

namespace Loom.Core.Math;

public static class SimdMathEngine
{
    // SIMD-accelerated Dot Product calculation for float vectors
    public static float DotProduct(ReadOnlySpan<float> vectorA, ReadOnlySpan<float> vectorB)
    {
        if (vectorA.Length != vectorB.Length)
            throw new ArgumentException("Vector lengths must be identical.");

        float sum = 0f;
        int i = 0;
        int simdLength = Vector<float>.Count;

        // Process blocks in parallel using SIMD hardware registers
        for (; i <= vectorA.Length - simdLength; i += simdLength)
        {
            var va = new Vector<float>(vectorA.Slice(i));
            var vb = new Vector<float>(vectorB.Slice(i));
            sum += Vector.Dot(va, vb);
        }

        // Handle remaining items (scalar math loop)
        for (; i < vectorA.Length; i++)
        {
            sum += vectorA[i] * vectorB[i];
        }

        return sum;
    }
}
```

### 3. Binary Embedding Cache (Memory-Mapped Store)
```csharp
using System;
using System.IO;
using System.IO.MemoryMappedFiles;
using System.Runtime.InteropServices;

namespace Loom.Core.Storage;

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct EmbeddingRecord
{
    public int VectorId;
    public float MetricScore;
    // Fast binary aligned structure
}

public sealed class BinaryCacheReader : IDisposable
{
    private MemoryMappedFile? _mmf;
    private MemoryMappedViewAccessor? _accessor;

    public unsafe ReadOnlySpan<EmbeddingRecord> LoadCache(string filepath)
    {
        var fileInfo = new FileInfo(filepath);
        if (!fileInfo.Exists) return ReadOnlySpan<EmbeddingRecord>.Empty;

        _mmf = MemoryMappedFile.CreateFromFile(filepath, FileMode.Open, "LoomCacheMap");
        _accessor = _mmf.CreateViewAccessor(0, fileInfo.Length, MemoryMappedFileAccess.Read);

        byte* pointer = null;
        _accessor.SafeMemoryMappedViewHandle.AcquirePointer(ref pointer);

        int structSize = Marshal.SizeOf<EmbeddingRecord>();
        int recordCount = (int)(fileInfo.Length / structSize);

        // Directly cast the raw file handle pointer into an efficient Span
        return new ReadOnlySpan<EmbeddingRecord>(pointer, recordCount);
    }

    public void Dispose()
    {
        _accessor?.Dispose();
        _mmf?.Dispose();
    }
}
```

---

## Production Security Matrix

To run in high-security enterprise containers or secure network rings:
* **Host Decoupling**: Loom runs as a separate network namespace from the target application, securely querying runtime telemetry over OS diagnostic ports (`/proc` file systems on Linux, EventPipes in .NET runtime).
* **SSH Sandbox Protection**: Standard interactive terminal processes are completely disabled. Custom channel dispatchers only permit predefined Loom telemetry command keys.
* **Low Impact Boundary**: Telemetry queries automatically back off if target application CPU levels exceed 85%, ensuring Loom never causes service degradation.
