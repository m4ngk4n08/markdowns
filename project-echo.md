# Project Echo
**System Architecture & Roadmap Specification**  
**Target Stack:** .NET 10, Angular 22, Local RAG, SIMD-Accelerated Vector Match  

---

## 1. Project Overview & Objective
**Project Echo** is a local, stateful, retrieval-augmented tutoring system designed to resolve the comprehension gap in AI-assisted development. By ingesting an AI-generated codebase and parsing it into structured architectural concepts (known as "Bricks"), Project Echo guides the developer through a strict, turn-based Socratic verification loop. 

The system's name represents the ultimate goal of the platform: the developer must *echo* back the architectural decisions in their own words. The system performs a friction audit on this "echo" before unlocking subsequent phases of the codebase.

---

## 2. SDLC Stage A: Requirements Analysis

### A. Functional Requirements (FR)
1.  **Codebase Ingestion Engine:**
    *   Must crawl a designated directory, filtering out non-source files.
    *   Must divide source code (.cs, .ts) and markdown documents (.md) into context-aware chunks aligned to syntax boundaries (classes, methods, file sections).
2.  **Semantic Vector generator:**
    *   Must interface with an embedding generator (local or cloud) via unified interfaces to translate text chunks into floating-point vectors.
3.  **The Echo Vector Store:**
    *   Must store chunk vectors in a flat, contiguous, in-memory array to optimize CPU cache-line efficiency.
    *   Must implement parallel, linear-scanning vector queries using hardware-level SIMD operations.
4.  **Stateful Verification Machine:**
    *   Must maintain a strict 5-stage turn loop per architectural concept.
    *   Must evaluate user explanations for conceptual gaps (the Friction Audit) during the active recall step.
5.  **State-Anchored Chat Client:**
    *   Must track conversation logs and current learning states.
    *   Must prepends the current state token (e.g., `[Current State: Brick #X, Turn Y]`) to all outgoing client messages.

### B. Non-Functional Requirements (NFR)
1.  **Search Latency:** The SIMD comparison search over 100,000 vector chunks must execute in under 10ms on a standard server CPU.
2.  **Memory Footprint:** The backend runtime must compile to Native AOT, aiming for an idle memory footprint of approximately 30MB.
3.  **Frontend Performance:** The frontend must execute in a fully Zoneless state, relying purely on Signals to trigger view refreshes.
4.  **Determinism:** The AI model's response patterns must be constrained through prompt-based sampling guidelines to prevent deviation from the state machine.

### C. Threat Modeling (STRIDE)
*   **Spoofing & Tampering:** The risk of malicious source code injecting prompt instructions during ingestion. *Mitigation:* Sanitize parsed text chunks and encapsulate retrieved codebase context inside distinct system-defined XML boundaries within the model's prompt.
*   **Information Disclosure:** Transmission of sensitive source code files to third-party endpoints. *Mitigation:* Abstract the embedding generator layer to allow a seamless swap between cloud APIs and local, offline ONNX or Ollama-based models.
*   **Denial of Service (DoS):** Oversized repositories exhausting system memory during parsing. *Mitigation:* Enforce hard limits on the maximum repository directory depth, maximum file sizes, and the maximum number of in-memory vector allocations.

---

## 3. SDLC Stage B: Architecture & System Design

### A. System Architecture & Data Flow
```text
+-------------------------------------------------------------+
|                     ANGULAR 22 FRONTEND                     |
|  - Zoneless Runtime            - Signal-based Forms         |
|  - Reactive State Store        - Automated State Anchoring  |
+------------------------------+------------------------------+
                               | HTTPS / SSE
                               v
+-------------------------------------------------------------+
|                       .NET 10 BACKEND                       |
|  - Web API Host               - Microsoft.Extensions.AI     |
|  - InMemory Flat Vector Store - Parallel LINQ Query Router   |
+------------------------------+------------------------------+
                               | Native CPU Intrinsics
                               v
               +------------------------------+
               |     SIMD VECTOR ENGINE       |
               |  - System.Numerics.Tensors   |
               |  - AVX-10.2 / AVX-512 / Neon |
               +------------------------------+
```
### B. The Echo RAG Pipeline
1.  **Semantic Chunking Strategy:** Use abstract syntax tree (AST) parsers for programming languages to split code into logical class and method boundaries, ensuring semantic blocks remain intact. Use heading-based sliding windows for markdown documents.
2.  **Memory Layout Optimization:** Align float vector arrays contiguously in memory. This contiguous allocation allows the CPU to load multiple vectors into L1/L2 cache concurrently, preparing the data for vectorized SIMD operations.
3.  **SIMD Search Execution:** When the user sends a message, the API vectorizes the input. The Echo Vector Engine performs a parallelized linear scan across the contiguous memory blocks. The CPU uses specialized hardware registers (AVX-512, AVX-10.2, or ARM Neon) to compare vector spaces in parallel, calculating Cosine Similarity under a single instruction multiple data pathway.
4.  **Prompt Assembly Oracle:** The top similarity results are fetched and formatted. These are injected into the System Prompt under clear delimitations, giving the generative AI model the exact codebase context it needs to mentor the user.

### C. Conversational State Machine & Drift Prevention
To guarantee that the AI model does not diverge from the V-model structure over long-running sessions, the system uses dual-state validation:
*   **Active State Anchor (Client-Side):** The Angular client holds the active state (e.g., Brick #1, Turn 2) in a reactive Signal. The form submission logic automatically intercepts the user's raw message and prefixes it with the state token before it reaches the network boundary.
*   **Template Verification (Server-Side):** The .NET 10 backend parses this incoming token, matches it against its internal state table, pulls the corresponding system prompt template, and strips the anchor from the payload before routing it to the generative model.

---

## 4. SDLC Stage C: Implementation Roadmap

### Milestone 1: Core Ingestion & Memory Storage (Weeks 1-2)
*   Develop directory traversing algorithms and AST-based source code parsers.
*   Implement flat, contiguous memory storage classes to house vector coordinates.
*   Integrate unified `Microsoft.Extensions.AI` interfaces to abstract embedding generation.

### Milestone 2: Echo SIMD Search Engine (Weeks 3-4)
*   Build the linear vector similarity scan using .NET 10 `System.Numerics.Tensors`.
*   Verify that JIT/AOT compilation paths successfully target native hardware registers (AVX-512, AVX-10.2, Neon).
*   Add multi-threaded vector comparison handling using Parallel LINQ.

### Milestone 3: Socratic Dialogue Controller (Weeks 5-6)
*   Write state validation interceptors in the ASP.NET Core API layer.
*   Implement the 5-turn pacing logic (Analogy, Deep Dive, Application, Verification, Integration) into system-level templates.
*   Develop the hard-stop correction handler to address user deviation or model hallucinations.

### Milestone 4: Zoneless Client Development (Weeks 7-8)
*   Set up the Angular 22 app workspace, disabling Zone.js entirely to run in a Zoneless state.
*   Build Signal-based stores to manage chat history, active Brick, and active Turn.
*   Implement reactive Signal Forms to handle user input validation and automated state-anchor prefixing.

### Milestone 5: System Integration & RAG Endpoints (Weeks 9-10)
*   Connect the Angular client to the ASP.NET Core Web API endpoints.
*   Integrate the retrieval step so that semantically relevant codebase chunks are injected into the system prompt on every API turn.

---

## 5. SDLC Stage D: Verification & Native AOT Safety

### A. Performance Audits
*   **Memory Fragmentation Analysis:** Benchmark memory allocations during ingestion to confirm that codebase chunks are successfully allocated in flat arrays without causing heap fragmentation.
*   **SIMD Scaling Analysis:** Run performance benchmarks on similarity searches at scales of 10k, 50k, 100k, and 500k chunks to verify linear latency scaling.

### B. Native AOT & Trim Verification
*   Enable compile-time static analyzers in the .csproj configuration to flag dynamic reflection or unreferenced code dependencies.
*   Ensure all JSON serialization pipelines utilize compile-time source generation instead of runtime reflection.
*   Execute test builds targeting Native AOT (`PublishAot=true`) to output highly secure, fast-starting native binaries.

---

## 6. SDLC Stage E: Deployment & Release Engineering

### A. Chiseled Container Strategy
*   Establish multi-stage Docker builds.
*   Perform the .NET 10 compilation and Native AOT assembly inside an SDK container with native compilation dependencies (Clang/zlib).
*   Deploy the compiled native binary inside a hardened, minimal chiseled runtime environment to minimize the image footprint and attack surface.

### B. Host Virtualization & Core Optimization
*   **Instruction Set Pass-Through:** Configure host VM orchestration profiles to guarantee that SIMD instruction flags are passed down to the container runtime.
*   **Thread Allocation Mapping:** Align Parallel LINQ thread allocations with physical host cores to prevent execution bottlenecks under concurrent user traffic.

---

## 7. Strategic Trade-offs Matrix

| Architectural Choice | Advantages | Strategic Trade-offs |
| :--- | :--- | :--- |
| **Flat In-Memory SIMD Scan** | Sub-5ms search latency for datasets under 100k chunks. Completely removes dependencies on third-party vector databases. | As the database scales past 500k records, linear scanning will consume more CPU cycles. Large-scale deployments will require moving to indexed vector systems (HNSW). |
| **Zoneless Angular 22** | Minimizes change detection passes, lowers bundle sizes, and provides precise rendering performance driven strictly by Signal mutations. | Restricts the use of older third-party Angular libraries that still rely on Zone.js execution loops, necessitating custom Signal wrappers for legacy code. |
| **Native AOT Compilation** | Hardens application security, minimizes the attack surface, and reduces memory footprints down to ~30MB with sub-second container cold starts. | Prevents the use of dynamic runtime libraries, requiring compile-time source generation for all serialization, dependency injection, and data models. |
| **CPU SIMD vs Dedicated GPU** | Simplifies infrastructure setup. The application can run on standard, cost-effective VMs or local development machines without complex GPU drivers. | Offloads similarity scanning to the CPU. While fast, the process of generating the initial vector embeddings on the CPU will remain slow unless external APIs are used. |

---

## 8. Mandatory Response Format Templates

The following templates represent the strict structures Project Echo must use to communicate with the user throughout each active milestone state.

### Template for Turn 1 (Stages 1 & 2)
> ## 🔲 Brick #[Number]: [Concept Name]
> 
> ### 📖 Stage 1: Foundation (V-Model Step A)
> [One-sentence summary + Analogies + Document citation]
> 
> [METACOGNITIVE CHECK - Pre-Mortem]: [Your question]
> 
> ### 📖 Stage 2: Deep Dive (V-Model Step B)
> [Numbered steps + Technical jargon + Supporting citations]
> 
> [METACOGNITIVE CHECK - Prediction]: [Your question]
> 
> ### ✅ Turn Audit
> - [ ] Checked the user's State Anchor?
> - [ ] Stage 1 analogy and summary included?
> - [ ] Stage 2 sequential steps included?
> - [ ] Exactly one citation for Stage 1?
> - [ ] Pre-mortem and Prediction checks ready?
> - [ ] Stopped and waiting for response?

### Template for Turn 2 (Stage 3)
> ### 📖 Stage 3: Application (V-Model Step C)
> [Briefly look at the user's prediction]
> 
> [How we do this in Loom + Citations to sub-stage/risk]
> 
> [METACOGNITIVE CHECK - Confidence Gauge]: [Your question]
> 
> ### ✅ Turn Audit
> - [ ] Checked the user's State Anchor?
> - [ ] Evaluated the user's prediction?
> - [ ] Explained Loom implementation with citations?
> - [ ] Confidence gauge check ready?
> - [ ] Stopped and waiting for response?

### Template for Turn 3 (Stage 4)
> ### 📖 Stage 4: Verification (V-Model Step D)
> I'm stepping back now. Take a stab at explaining this concept to me in your own words. How does it work, and why does Project Loom require it?

### Template for Turn 4 (Stage 5)
> ### 🔍 Friction Audit & Stage 5: Integration (V-Model Step E)
> [Encouraging analysis of their explanation, pointing out any fuzzy spots]
> 
> [Integration details connecting to previous bricks + Citations]
> 
> [METACOGNITIVE CHECK - If-Then Projection]: [Your question, using actual names of completed bricks]
> 
> ### ✅ Turn Audit
> - [ ] Checked the user's State Anchor?
> - [ ] Handled gaps in the user's explanation?
> - [ ] Connected this to previous bricks?
> - [ ] Exactly one citation for Stage 5?
> - [ ] If-Then projection ready?
> - [ ] Stopped and waiting for response?
