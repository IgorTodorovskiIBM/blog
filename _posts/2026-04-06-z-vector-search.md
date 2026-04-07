---
layout:       post
title:        "From Porting to RAG: Building a Vector Search Engine for z/OS"
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

In a [previous blog post](https://igortodorovskiibm.github.io/blog/2023/08/22/llama.cpp/), we demonstrated that porting LLaMa.cpp to z/OS was not only possible but practical, you really can run a 7B parameter LLM on a mainframe. It was a bit slow, but it worked. After that initial port landed, I started wondering what else we could build on top of it.

The scenario I had in mind was simple. Picture a z/OS system programmer staring at a console flooded with messages, ABENDs, RACF violations, dataset allocation errors, CICS abends, and trying to figure out which ones matter, what they mean, and whether the system has seen anything like this before. Today, that involves flipping between IBM message manuals, internal runbooks, ticket histories, and tribal knowledge. What if you could just *ask*? "What does this message mean? Has it happened before? What did we do about it last time?"

And critically, I wanted that experience to work **directly on z/OS**, not by shipping log data off to a cloud LLM, not by running a Python notebook on a Linux VM somewhere, but right there on the LPAR where the data already lives. For the air-gapped, data-sensitive workloads that run on mainframes, anything else is a non-starter.

A chatbot is fun, but the right tool for that scenario is **Retrieval-Augmented Generation (RAG)**, using an LLM not as a know-it-all encyclopedia, but as a reasoning layer over your own data. And the foundation of every RAG system is the same thing: **embeddings**. If we could get embedding models running on z/OS, we could build a proper semantic search engine, locally, on the mainframe.

The result is **z-vector-search**, a RAG-powered semantic search engine running natively on z/OS, along with **z-console**, an example prototype built on top of it that enriches operator console messages with relevant context.

## Getting Embeddings Working on z/OS

The idea actually came from a [llama.cpp discussion thread](https://github.com/ggml-org/llama.cpp/discussions/7712) about adding embedding model support. Reading through it, I realized that all the pieces I needed to build a z/OS RAG system were already on the table, I just had to wire them up.

### What's an embedding, anyway?

If you've never worked with them, embeddings are the trick that makes "semantic search" possible. An embedding model takes a piece of text and turns it into a list of numbers, a **vector**, that captures its meaning. The clever part is that two pieces of text with similar meanings produce vectors that are mathematically close to each other in space, even if they share no words in common.

That means a search for `"dataset allocation failure"` can find a document that says `"IEC070I"`, because both phrases live near each other in vector space. No keyword matching, no synonyms list, no manual rules. The model has already learned what things *mean*.

To do search with embeddings, you embed every document once and store the vectors. At query time, you embed the query the same way and find the documents whose vectors are nearest yours. That's the whole game.

### The Model

The model I chose was **Nomic Embed Text v1.5**. Quantized to Q4_K_M, it's just ~84 MB, small enough to run comfortably on z/OS, and well-regarded for retrieval tasks. It's an **encoder-only** model (think BERT-style), which means it's purpose-built for turning text into vectors rather than generating new text.

### What It Took to Get Working

llama.cpp's embedding support is newer than its text generation support, so a few things needed attention to make it behave on z/OS:

- **Encoder model code path.** Encoder-only models like Nomic take a different route through llama.cpp than decoder models like LLaMa. They produce one vector per input rather than streaming tokens, which means a different API (`llama_encode()` instead of `llama_decode()`) and slightly different batch handling.

- **Pooling.** The model produces a vector for every token, but you want a single vector per document. Nomic expects MEAN pooling, averaging the per-token vectors together. Getting this wrong produces embeddings that *look* fine but retrieve nonsense.

- **Document and query prefixes.** Nomic uses a clever convention where you prepend `search_document:` to text you're indexing and `search_query:` to text you're searching for. This subtly nudges the model to put documents and queries in slightly different regions of the embedding space, which measurably improves retrieval quality. A simple trick, but it makes a real difference.

- **The endianness problem, again!** Just like with the original llama.cpp port, endianness came back to haunt me. Embedding vectors are arrays of 32-bit floats, and a database built on x86 (little-endian) needs every float byte-swapped before z/OS (big-endian) can read them. I added automatic endianness detection and a `--convert-endian` flag so you can build a knowledge base on a fast Linux box and ship the `.db` file over to z/OS.

After working through these, I had embeddings producing sensible vectors on z/OS, and that was enough to start building something real.

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

At query time, sqlite-vec performs KNN (k-nearest-neighbor) search using cosine distance. Building sqlite-vec on z/OS took a couple of small patches, guarding BSD `u_int*_t` typedefs behind `__MVS__` and resolving a macro conflict with `sqlite3ext.h`, but nothing dramatic.

### Chunking

Large documents can't be embedded as a single unit: encoder models have a token limit, and long texts lose detail when compressed into one vector. So documents are split into overlapping chunks, 256 tokens each, with 64 tokens of overlap between adjacent chunks. The overlap ensures context at chunk boundaries isn't lost, and each chunk is independently embedded and stored.

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

The query returns the most semantically relevant chunks, ranked by similarity. No keyword matching needed, if your runbook says "dataset allocation failure" and you search for "IEC070I error," it still finds the right answer.

All tools support `--json` output for scripting, so you can pipe results into `jq`:

```bash
z-query --json "dataset allocation failure" | jq '.results[0].snippet'
```

## Hybrid Search

Pure semantic search is powerful, but sometimes you know exactly what you're looking for. If an operator sees `ICH408I` and wants to look it up, they don't need semantic similarity, they need an exact match.

So z-query automatically classifies each query:

- `ICH408I` → keyword search (exact message ID via SQL LIKE)
- `DFH*` → keyword search (wildcard)
- `MSGID:IEF JOB:PAYROLL` → keyword search (structured prefix)
- `why is my CICS transaction failing` → semantic search (natural language)
- `ICH408I unauthorized access` → hybrid (both, merged)

When both modes run, results are merged using **Reciprocal Rank Fusion (RRF)**, a clever technique for combining ranked lists without needing to normalize scores across different methods. The formula is simple:

```
score = Σ 1/(k + rank)    where k = 60
```

This gives you the precision of keyword search with the recall of semantic search, and it's the standard approach in modern hybrid retrieval systems.

## The IBM z/OS Messages Knowledge Base

A semantic search engine is only as good as its data. To make the tool immediately useful for z/OS operators, I built a pre-packaged knowledge base of **24,565 IBM z/OS messages**, covering MVS, RACF, CICS, DB2, MQ, and system abend/wait codes. Each entry includes the message ID, explanation, system action, and operator response.

The knowledge base ships as a ready-to-use SQLite database, so `z-query` can answer questions about IBM messages out of the box:

```bash
z-query "what does abend S0C4 mean"
```

This returns the relevant system code documentation explaining that S0C4 is a protection exception, typically caused by a program accessing storage it doesn't own.

## z-console: RAG for the Operator Console

This is where everything came together.

The z/OS operator console is the nerve center of a mainframe system. Messages stream in constantly, job completions, security events, storage allocations, errors, abends. Experienced operators know what to look for, but the volume is overwhelming, and critical messages can be buried in noise.

**z-console** reads your console messages and enriches each one with relevant context from both IBM documentation and your system's own history.

### How It Works

1. **Read**, pulls messages from the z/OS SYSLOG via `pcon` (a z/OS utility that reads the system log)
2. **Filter**, picks out high-value messages: abends (`IEF*`), data errors (`IEC*`), RACF violations (`ICH*`), CICS (`DFH*`), DB2 (`DSN*`), MQ (`CSQ*`), and anything with action/error severity
3. **Look up**, for each interesting message, runs a two-phase search:
   - **Keyword** against the IBM messages knowledge base, what does this message ID mean?
   - **Semantic** against your operational history, have we seen something like this before?
4. **Display**, presents everything with color-coded severity and ranked context

### Input Modes

z-console has three ways to feed it messages:

```bash
# Single message, the simplest starting point
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
     [2026-04-03 14:22] ICH408I,ICH409I, BATCH1 revoked on SYS1,
     resolved by security team reset. Related: RACF password policy
     change ticket INC-4421.
     (distance: 0.31)
```

In a single glance, the operator knows what the message means *and* that it's happened before, with a pointer to how it was resolved last time. That's the whole pitch for RAG on the console.

### Summary Mode

Sometimes you don't need full RAG enrichment, just a quick health check. `--summary` groups messages by severity and category without loading the embedding model at all:

```bash
z-console --summary --pcon -l
```

```
=== Console Summary (last hour) ===
Total messages: 847 | Interesting: 23

  CRITICAL/ERROR (3):
    ICH408I  ×2  USER(BATCH1) LOGON/JOB INITIATION - ACCESS REVOKED
    IEC030I  ×1  I/O ERROR, DATASET SYS1.LINKLIB

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

Messages are grouped into 5-minute time windows and stored with structured metadata, message IDs, highest severity, jobname, system name, timestamps. The longer it runs, the more historical context z-console can draw on.

## Measuring Performance

How fast is all of this? Rather than hardcoding numbers into this post, where they'd go stale the moment you run on different hardware, both `z-query` and `z-console` support a `--metrics` flag that outputs timing data as JSON on stderr.

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
   Embed (Nomic Embed v1.5)
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

Everything runs locally on z/OS. No external API calls, no cloud dependencies, no data leaving the LPAR. For the air-gapped environments common in finance and healthcare, that's not a nice-to-have, it's a hard requirement.

The entire stack is pure C++17 with vendored SQLite and sqlite-vec, linked against llama.cpp. No Python runtime, no Java, no external dependencies beyond what `zopen` provides.

## Bonus Round: Making llama.cpp Faster on z/OS

Once z-vector-search was working end-to-end, one thing was painfully obvious: it was slow. Not broken, just slow. Embedding a single chunk took longer than it had any right to.

A bit of profiling pointed at the obvious culprits: the quantized matrix-vector multiplies that dominate every forward pass, and the elementwise float helpers (`ggml_vec_add_f32`, `ggml_vec_mul_f32`, and friends) that get called millions of times per query. On x86, llama.cpp vectorizes all of this with AVX2/AVX-512 intrinsics. On ARM, it uses NEON. On z/OS? **Nothing.** The s390x backend was running scalar code through the entire hot path.

This was a great opportunity for some IBM Z SIMD work. IBM Z processors from z13 onwards include the **Vector Facility for z/Architecture (VXE)**, a 128-bit SIMD instruction set conceptually similar to SSE/AVX or NEON, and the IBM C/C++ compiler exposes it via vector intrinsics.

To accelerate this work, I leaned heavily on **[IBM Bob](https://bob.ibm.com/)**, IBM's internal AI assistant, to help me navigate VXE intrinsics, generate first-pass implementations of the s390x vectorized routines, and cross-check the bit-level details of llama.cpp's quantized formats. Pair-programming with Bob turned what would have been weeks of intrinsic spelunking into a much shorter loop of "draft → verify → tune."

The result was a set of new s390x implementations:

- **Vector helpers**, `ggml_vec_add_f32`, `ggml_vec_sub_f32`, `ggml_vec_mul_f32`, `ggml_vec_scale_f32`, `ggml_vec_mad_f32`, and the FP16 variants, now process 8 floats per loop iteration using `vec_xl`/`vec_add`/`vec_xst`.
- **Q4_K × Q8_K matrix-vector multiply**, a brand new `ggml-cpu/arch/s390/repack.cpp` implementing `ggml_gemv_q4_K_8x4_q8_K` with VXE intrinsics. The core trick is using `vec_mule`/`vec_mulo` (multiply even/odd lanes) to widen int8 → int16 cleanly, then a second pair to horizontally reduce into int32, the whole sequence retires in a handful of cycles on z15.
- **Q8_K row quantization**, a vectorized `quantize_row_q8_K` that uses `__builtin_s390_vfisb` to round-and-convert in a single instruction.
- **CMake plumbing**, a new `OS390` branch in `ggml/src/ggml-cpu/CMakeLists.txt` that turns on `-fzvector -m64 -march=z15` and pulls in the s390x sources.
- **Optional MASS/MASSV linkage**, IBM's hand-tuned vector math library, gated behind `-DGGML_USE_MASSV=ON`, for faster transcendentals like `exp` and `log`.

Roughly 900 lines of new code across five files. Forward passes on a z15 LPAR feel noticeably snappier as a result, and the `--metrics` numbers above reflect the post-vectorization world. I plan to clean these patches up and submit them upstream so every z/OS llama.cpp user benefits.

## What's Next

A few threads I'd like to pull on:

- **Timeline correlation**, "what else was happening on the system when this error occurred?"
- **Proactive alerting**, have the daemon watch for patterns and alert operators before problems escalate
- **Upstreaming**, cleaning up the llama.cpp s390x patches and submitting them to the community so everyone benefits

## Getting Started

**Prerequisites:** you'll need the [zopen package manager](https://zopen.community) set up on your z/OS system. If you haven't used zopen before, the [QuickStart Guide](https://zopen.community/#/Guides/QuickStart) takes about five minutes.

The simplest way to get going is straight from zopen:

```bash
zopen install z-vector-search
```

That pulls in llama.cpp and the z-vector-search tools in one shot. If you'd rather build from source:

```bash
# 1. Install llama.cpp via zopen
zopen install llamacpp

# 2. Build z-vector-search
cmake -B build -DLLAMA_ROOT=$ZOPEN_PKGINSTALL/llamacpp
cmake --build build
```

Then, regardless of how you installed:

```bash
# 3. Run setup, downloads the model and unpacks the IBM messages DB
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

What started as "can we get embeddings working on z/OS?" turned into a full RAG-powered operational assistant. Each step revealed the next problem worth solving. Embeddings gave us semantic understanding. A vector store made it persistent. Hybrid search made it practical for operators who think in message IDs, not natural language. z-console tied it all together. And along the way, a round of SIMD vectorization made the whole thing fast enough to actually use.

The mainframe has always been about running critical workloads reliably. Now it can understand them too.
