---
layout:       post
title:        "From Porting to RAG: Speeding up llama.cpp and Building a Vector Search Engine for z/OS"
author:       "Igor Todorovski"
header-img:   "img/in-post/ai_on_z.jpg"
catalog:      true
hidden:       true
tags:
    - z/OS
    - AI
    - llama.cpp
    - embeddings
    - vector-search
    - RAG
    - zopen
    - SIMD
---

In a [previous blog post](https://igortodorovskiibm.github.io/blog/2023/08/22/llama.cpp/), we demonstrated that porting LLaMa.cpp to z/OS was not only possible but practical — you really can run a 7B parameter LLM on a mainframe. After that initial port landed, two things started nagging at me.

First, it was slow. Correct, but slow. The s390x backend was falling back to scalar code for most of the hot path, leaving real performance on the table. Second, llama.cpp had recently added support for **embedding models** — and if we could get those working on z/OS, it would open the door to something much more interesting than a chatbot demo: a proper semantic search engine, running locally, on the mainframe.

This post is the story of chasing both threads. It ends with **z-vector-search**, a RAG-powered semantic search engine for z/OS, and **z-console**, a tool that enriches live operator console messages with context from IBM documentation and your system's own operational history.

Let's walk through how we got there — the wins, the bugs, and the "aha" moments along the way.

## Making llama.cpp Fast on z/OS

Before getting excited about new features, I wanted to fix what was already there. Profiling the existing port showed the hot path exactly where you'd expect it: the quantized matrix-vector multiplies (`ggml_gemv_q4_K_8x4_q8_K`), the elementwise float helpers (`ggml_vec_add_f32`, `ggml_vec_sub_f32`, `ggml_vec_scale_f32`), and the row quantization routine (`quantize_row_q8_K`).

On x86, llama.cpp vectorizes all of this with AVX2/AVX-512 intrinsics. On ARM, it uses NEON. On z/OS? **Nothing.** Scalar code everywhere.

### z/Architecture Vector Extensions (VXE)

IBM Z processors from z13 onwards include the **Vector Facility for z/Architecture** — a SIMD instruction set with 128-bit vector registers, very similar in spirit to SSE/AVX or NEON. The IBM C/C++ compiler exposes these through intrinsics like `vec_xl` (load), `vec_xst` (store), `vec_add`, `vec_mul`, and the widening `vec_mule`/`vec_mulo` (multiply even/odd elements).

The plan was simple: add vectorized s390x implementations for the hot path.

### Vectorizing the Vector Helpers

The easiest wins were the elementwise float operations in `ggml-cpu/vec.h` — called millions of times per forward pass. Here's the before and after for `ggml_vec_add_f32`:

**Before — scalar:**
```c
for (int i = 0; i < n; ++i) {
    z[i] = x[i] + y[i];
}
```

**After — VXE vectorized:**
```c
int i = 0;
#if defined(__VXE__) || defined(__VXE2__) || defined(__MVS__)
for (; i + 7 < n; i += 8) {
    vec_xst(vec_add(vec_xl(0, x + i),     vec_xl(0, y + i)),     0, z + i);
    vec_xst(vec_add(vec_xl(0, x + i + 4), vec_xl(0, y + i + 4)), 0, z + i + 4);
}
for (; i + 3 < n; i += 4) {
    vec_xst(vec_add(vec_xl(0, x + i), vec_xl(0, y + i)), 0, z + i);
}
#endif
for (; i < n; ++i) {
    z[i] = x[i] + y[i];
}
```

Each VXE register holds four floats, so the unrolled loop processes eight floats per iteration. I applied the same treatment to `ggml_vec_sub_f32`, `ggml_vec_acc_f32`, `ggml_vec_mul_f32`, `ggml_vec_scale_f32`, `ggml_vec_mad_f32`, and friends — plus the FP16 variants, which convert on load and store.

### The Big One: Q4_K × Q8_K GEMV

The real workhorse of quantized inference is the Q4_K × Q8_K matrix-vector multiply. It runs for every weight matrix in every layer, on every forward pass. Adding an s390x-specific implementation had by far the biggest impact.

I wrote a new file, `ggml/src/ggml-cpu/arch/s390/repack.cpp`, implementing `ggml_gemv_q4_K_8x4_q8_K` with VXE intrinsics. The core primitive is a column-wise dot product that produces four dot products of four elements each in a single SIMD pass:

```cpp
static inline int32x4_t ggml_vec_dot_col4(int32x4_t acc, int8x16_t a, int8x16_t b) {
    const int16x8_t ones = vec_splats((int16_t)1);
    const int16x8_t p = vec_add(vec_mule(a, b), vec_mulo(a, b));
    return vec_add(acc, vec_add(vec_mule(p, ones), vec_mulo(p, ones)));
}
```

`vec_mule` and `vec_mulo` multiply the even and odd lanes respectively, widening from int8 to int16 to avoid overflow. Summed together, they give you the full eight-element product. A second pair of mul-even/mul-odd against a broadcast of 1 performs the horizontal reduction into int32. On z15, this entire sequence retires in just a handful of cycles.

The outer loop decodes the Q4_K scales (those `kmask1`/`kmask2`/`kmask3` bit tricks), broadcasts them into vector registers, and feeds the quantized weights through the dot product. Eight columns are processed in parallel — `sumf_lo` and `sumf_hi` — matching the 8×4 block layout that llama.cpp's repacking uses.

### Vectorized Q8_K Quantization

The input to a quantized GEMV is itself freshly quantized every pass: each activation tensor gets converted to Q8_K format before the matmul. I added an s390x-specific `quantize_row_q8_K` using VXE for the per-group scale-and-round loop, with a nice little trick: `__builtin_s390_vfisb(v, 4, 1)` emits a single **vector float round-to-integer** instruction, which is considerably faster than a scalar rint + convert.

### Build System Integration

None of this compiles unless CMake knows where to find it. I added a z/OS branch to `ggml/src/ggml-cpu/CMakeLists.txt`:

```cmake
elseif (CMAKE_SYSTEM_NAME STREQUAL "OS390")
    message(STATUS "z/OS detected")
    list(APPEND ARCH_FLAGS -fzvector -m64 -march=z15)
    list(APPEND GGML_CPU_SOURCES ggml-cpu/arch/s390/quants.c)
    list(APPEND GGML_CPU_SOURCES ggml-cpu/arch/s390/repack.cpp)
    list(APPEND ARCH_DEFINITIONS GGML_VXE)
    ...
```

The `-fzvector` flag is the magic incantation that turns on the VXE intrinsic headers in the IBM C++ compiler for z/OS.

### Optional: IBM MASS Library

IBM's **Mathematical Acceleration Subsystem (MASS)** — specifically the vector flavor, MASSV — provides hand-tuned implementations of transcendental functions (exp, log, sin, cos, pow). I added optional linkage against MASS behind a `GGML_USE_MASSV` CMake flag:

```cmake
if(GGML_USE_MASSV)
    target_link_libraries(... "/usr/lpp/cbclib/lib/libmassv.arch${TARGET_ARCH}.a")
    target_compile_definitions(... GGML_USE_MASS)
endif()
```

When enabled, `ggml_vec_exp_f32` and friends route through MASSV for extra speed on activation functions. It's opt-in because `vsexp` has some edge cases in interactive mode I'm still working through.

### The Result

All told, the vectorization work touched five files and added around 900 lines. Matrix-vector multiplies that had been running scalar now benefit from 4-wide SIMD, and the elementwise helpers run 8-wide. Forward passes on a z15 LPAR feel noticeably snappier — enough that running embedding models interactively starts to feel reasonable, which turns out to be important for what comes next.

I plan to clean these patches up and submit them upstream so that every z/OS llama.cpp user benefits, not just those building from my port.

## Getting Embeddings Working

With a faster llama.cpp in hand, it was time to tackle the second thread: embeddings.

Embedding models are the foundation of semantic search. Instead of generating text, they convert text into dense numerical vectors that capture meaning. Two semantically similar pieces of text produce vectors that are close together in vector space — so "dataset allocation failure" and "IEC070I" end up near each other even though they share no keywords.

The model I chose was **Nomic Embed Text v1.5**. Quantized to Q4_K_M, it's just ~84 MB — small enough to run comfortably on z/OS, and well-regarded for retrieval tasks.

Getting it working took some effort:

1. **Encoder model support.** Embedding models like Nomic are encoder-only (think BERT), which take a different code path in llama.cpp than decoder models like LLaMa. I had to ensure `llama_encode()` was being called correctly, with all tokens marked as outputs, and that the batch handling worked for encoder sequences.

2. **MEAN pooling.** The Nomic model uses MEAN pooling to aggregate per-token embeddings into a single document-level vector. Getting this right was essential — a wrong pooling strategy produces embeddings that *look* valid but retrieve nonsense.

3. **Prefix strategy.** Nomic uses a prefix convention: `search_document:` is prepended when indexing, and `search_query:` when querying. This creates better separation between document and query embeddings and measurably improves retrieval accuracy.

4. **The endianness problem — again!** Just like with inference, endianness came back to haunt me. Embedding vectors are arrays of 32-bit floats; a database created on x86 (little-endian) and moved to z/OS (big-endian) needs every float in every vector byte-swapped. I added automatic endianness detection and a `--convert-endian` flag for cross-platform portability.

The debugging process was... educational. My first attempt produced garbage vectors — the KV cache was being contaminated between chunks, so every embedding after the first was polluted with residual state. The fix was to clear the cache between encode calls. Then I discovered the batch size had to match the context size for encoder models, or llama.cpp would silently crash. Each fix peeled back another layer.

But eventually — embeddings working reliably on z/OS. Terrific!

## Building the Search Engine

With working (and fast) embeddings, the next step was obvious: build a persistent vector store so you could index documents once and query them repeatedly.

### Storage: SQLite + sqlite-vec

I chose **SQLite** as the backend, extended with **sqlite-vec** for vector similarity search. The combination is elegant: no database server to manage, no network dependencies, just a single `.db` file.

The schema stores each text chunk alongside its embedding and metadata:

```
chunks table:
  - filename, snippet (text content)
  - vec_chunks (embedding vector via sqlite-vec)
  - source_type, mtime (metadata)
  - msgid, severity, jobname, sysname (structured fields for console data)
```

At query time, sqlite-vec performs KNN (k-nearest-neighbor) search using cosine distance. Building sqlite-vec on z/OS took a couple of small patches — guarding BSD `u_int*_t` typedefs behind `__MVS__` and resolving a macro conflict with `sqlite3ext.h` — but nothing dramatic.

### Chunking

Large documents can't be embedded as a single unit: encoder models have a token limit, and long texts lose detail when compressed into one vector. So documents are split into overlapping chunks — 256 tokens each, with 64 tokens of overlap between adjacent chunks. The overlap ensures context at chunk boundaries isn't lost, and each chunk is independently embedded and stored.

### The Tools

The project shipped as a suite of command-line tools:

| Tool | Purpose |
|------|---------|
| `z-index` | Index documents into the persistent vector store |
| `z-query` | Search the store with natural language queries |
| `z-vector-search` | One-shot mode: index and query without persistence |
| `z-setup` | First-run setup: download the model, unpack the IBM messages DB |

A typical workflow:

```bash
# Index your runbooks
z-index --store ~/my-store.db /path/to/runbooks/*.txt

# Search with natural language
z-query --store ~/my-store.db "how do I recover from an IEC070I error"
```

The query returns the most semantically relevant chunks, ranked by similarity. No keyword matching needed — if your runbook says "dataset allocation failure" and you search for "IEC070I error," it still finds the right answer.

All tools support `--json` output for scripting, so you can pipe results into `jq`:

```bash
z-query --json "dataset allocation failure" | jq '.results[0].snippet'
```

## Hybrid Search

Pure semantic search is powerful, but sometimes you know exactly what you're looking for. If an operator sees `ICH408I` and wants to look it up, they don't need semantic similarity — they need an exact match.

So z-query automatically classifies each query:

- `ICH408I` → keyword search (exact message ID via SQL LIKE)
- `DFH*` → keyword search (wildcard)
- `MSGID:IEF JOB:PAYROLL` → keyword search (structured prefix)
- `why is my CICS transaction failing` → semantic search (natural language)
- `ICH408I unauthorized access` → hybrid (both, merged)

When both modes run, results are merged using **Reciprocal Rank Fusion (RRF)** — a clever technique for combining ranked lists without needing to normalize scores across different methods. The formula is simple:

```
score = Σ 1/(k + rank)    where k = 60
```

This gives you the precision of keyword search with the recall of semantic search, and it's the standard approach in modern hybrid retrieval systems.

## The IBM z/OS Messages Knowledge Base

A semantic search engine is only as good as its data. To make the tool immediately useful for z/OS operators, I built a pre-packaged knowledge base of **24,565 IBM z/OS messages** — covering MVS, RACF, CICS, DB2, MQ, and system abend/wait codes. Each entry includes the message ID, explanation, system action, and operator response.

The knowledge base ships as a ready-to-use SQLite database, so `z-query` can answer questions about IBM messages out of the box:

```bash
z-query "what does abend S0C4 mean"
```

This returns the relevant system code documentation explaining that S0C4 is a protection exception — typically caused by a program accessing storage it doesn't own.

## z-console: RAG for the Operator Console

This is where everything came together.

The z/OS operator console is the nerve center of a mainframe system. Messages stream in constantly — job completions, security events, storage allocations, errors, abends. Experienced operators know what to look for, but the volume is overwhelming, and critical messages can be buried in noise.

**z-console** reads your console messages and enriches each one with relevant context from both IBM documentation and your system's own history.

### How It Works

1. **Read** — pulls messages from the z/OS SYSLOG via `pcon` (a z/OS utility that reads the system log)
2. **Filter** — picks out high-value messages: abends (`IEF*`), data errors (`IEC*`), RACF violations (`ICH*`), CICS (`DFH*`), DB2 (`DSN*`), MQ (`CSQ*`), and anything with action/error severity
3. **Look up** — for each interesting message, runs a two-phase search:
   - **Keyword** against the IBM messages knowledge base — what does this message ID mean?
   - **Semantic** against your operational history — have we seen something like this before?
4. **Display** — presents everything with color-coded severity and ranked context

### Input Modes

z-console has three ways to feed it messages:

```bash
# Single message — the simplest starting point
z-console "ICH408I USER(BATCH1) GROUP(PROD) LOGON/JOB INITIATION - ACCESS REVOKED"

# Live console via pcon
z-console --pcon -l                    # last hour
z-console --pcon -t 30                 # last 30 minutes
z-console --since 2026-04-06T10:00     # since a specific timestamp

# Pipe from stdin
cat syslog.txt | z-console
```

### Example Output

Running `z-console --pcon -l` on a system with an access violation might produce:

```
Parsed 847 messages, 23 interesting, 14 unique IDs to look up.

━━━ ICH408I (severity: E) ━━━
  ICH408I USER(BATCH1) GROUP(PROD) NAME(BATCH JOB)
    LOGON/JOB INITIATION - ACCESS REVOKED

  IBM Documentation (keyword match):
     ICH408I - A RACF-defined user has been revoked. The user's access
     authority has been removed, typically because consecutive incorrect
     password attempts exceeded the SETROPTS PASSWORD limit.
     System Action: The logon or job is rejected.
     Operator Response: Contact the security administrator to reinstate
     access via ALTUSER userid RESUME.
     (distance: 0.12)

  Operational History (semantic match):
     [2026-04-03 14:22] ICH408I,ICH409I — BATCH1 revoked on SYS1,
     resolved by security team reset. Related: RACF password policy
     change ticket INC-4421.
     (distance: 0.31)
```

In a single glance, the operator knows what the message means *and* that it's happened before — with a pointer to how it was resolved last time. That's the whole pitch for RAG on the console.

### Summary Mode

Sometimes you don't need full RAG enrichment — just a quick health check. `--summary` groups messages by severity and category without loading the embedding model at all:

```bash
z-console --summary --pcon -l
```

```
=== Console Summary (last hour) ===
Total messages: 847 | Interesting: 23

  CRITICAL/ERROR (3):
    ICH408I  ×2  USER(BATCH1) LOGON/JOB INITIATION - ACCESS REVOKED
    IEC030I  ×1  I/O ERROR — DATASET SYS1.LINKLIB

  WARNING (5):
    IEA404W  ×3  REAL STORAGE SHORTAGE
    CSV028W  ×2  MODULE NOT FOUND IN LINKLIST

  INFORMATIONAL (15):
    DFH1501I ×8  CICS TRANSACTION COMPLETED
    DSN9022I ×7  DB2 COMMAND COMPLETED
```

Fast enough to run frequently, and gives operators an at-a-glance view of system health.

### Message Filtering

Not every z/OS message is worth looking up. High-volume, low-value messages like `$HASP` job queue notifications or `IGD103I` storage allocations would drown out the signal. z-console ships with a default skip list, which operators can customize by editing `~/.z-vector-search/skip_msgids.txt`:

```
# Trailing * for prefix match
$HASP*
IEF196I
IEF285I
IGD103I
IRR010I
```

### Building Operational History

To power the "have we seen this before?" lookups, there's a companion tool called `z-ingest-console`. It runs as a background daemon via `z-console-daemon.sh` (every 5 minutes by default) and continuously indexes console messages into the vector store:

```bash
# Start the daemon in the background
nohup ./z-console-daemon.sh &
```

Messages are grouped into 5-minute time windows and stored with structured metadata — message IDs, highest severity, jobname, system name, timestamps. The longer it runs, the more historical context z-console can draw on.

## Measuring Performance

I mentioned "snappier" earlier without showing numbers. Rather than hardcoding them into this post — where they'd go stale the moment you run on different hardware — both `z-query` and `z-console` now support a `--metrics` flag that outputs timing data as JSON on stderr.

For a query:

```bash
z-query --metrics "what does abend S0C4 mean" 2>metrics.json
```

```json
{"mode":"semantic","model_load_ms":2341.5,"embed_ms":287.3,
 "search_ms":42.1,"total_ms":2812.4,"results":5,"store_chunks":34102}
```

For z-console, the metrics break down timing across all enriched messages, with per-message averages:

```bash
z-console --metrics --pcon -l 2>metrics.json
```

```json
{"total_parsed":847,"interesting":23,"skipped":824,"unique_ids":14,
 "cache_hits":3,"enriched":11,"model_load_ms":2341.5,
 "total_enrich_ms":4892.1,"total_embed_ms":3156.7,"total_search_ms":1204.8,
 "avg_enrich_ms":444.7,"avg_embed_ms":286.9,"avg_search_ms":109.5}
```

The key insight from the metrics: model load is a one-time cost of a few seconds, and after that each message enrichment takes well under half a second. Keyword-only modes (`--summary`, pure msgid lookups) skip the model entirely and return in milliseconds. The `--metrics` output lets you measure exactly what matters on your LPAR, under your workload.

## The Full Picture

Pulling it all together, here's the pipeline that runs every time z-console enriches a message:

```
Console Messages / Documents
        ↓
   Tokenize (llama.cpp)
        ↓
   Chunk (256 tokens, 64 overlap)
        ↓
   Embed (Nomic Embed v1.5 + VXE SIMD)
        ↓
   L2 Normalize
        ↓
   Store (SQLite + sqlite-vec)
        ↓
   Query → Classify → Keyword / Semantic / Hybrid
        ↓
   Reciprocal Rank Fusion
        ↓
   Top-K Results with Context
```

Everything runs locally on z/OS. No external API calls, no cloud dependencies, no data leaving the LPAR. For the air-gapped environments common in finance and healthcare, that's not a nice-to-have — it's a hard requirement.

The entire stack is pure C++17 with vendored SQLite and sqlite-vec, linked against llama.cpp. No Python runtime, no Java, no external dependencies beyond what `zopen` provides.

## What's Next

A few threads I'd like to pull on:

- **Timeline correlation** — "what else was happening on the system when this error occurred?"
- **Proactive alerting** — have the daemon watch for patterns and alert operators before problems escalate
- **Upstreaming** — cleaning up the llama.cpp s390x patches and submitting them to the community so everyone benefits

## Getting Started

**Prerequisites:** you'll need the [zopen package manager](https://zopen.community) set up on your z/OS system. If you haven't used zopen before, the [QuickStart Guide](https://zopen.community/#/Guides/QuickStart) takes about five minutes.

```bash
# 1. Install llama.cpp via zopen
zopen install llamacpp

# 2. Build z-vector-search
cmake -B build -DLLAMA_ROOT=$ZOPEN_PKGINSTALL/llamacpp
cmake --build build

# 3. Run setup — downloads the model and unpacks the IBM messages DB
z-setup

# 4. Query the IBM messages knowledge base
z-query "what does abend S0C4 mean"

# 5. Look up a single console message
z-console "ICH408I USER(BATCH1) GROUP(PROD) LOGON/JOB INITIATION - ACCESS REVOKED"

# 6. Or read the last hour of live console
z-console --pcon -l
```

The source code is available on GitHub.

## Conclusion

What started as "let's make llama.cpp faster on z/OS" turned into a full RAG-powered operational assistant. Each step revealed the next problem worth solving. SIMD vectorization made the inference engine fast enough to be useful. Embeddings gave us semantic understanding. A vector store made it persistent. Hybrid search made it practical for operators who think in message IDs, not natural language. And z-console tied it all together into something that makes the mainframe console genuinely more manageable.

The mainframe has always been about running critical workloads reliably. Now it can understand them too.
