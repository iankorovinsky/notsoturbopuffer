# Turbopuffer Blogs

## Continuous recall measurement

`https://turbopuffer.com/blog/continuous-recall`

**Key takeaways:**

1. **ANN trade-off**: Approximate indexes sacrifice accuracy for speed. Exhaustive search on 100M vectors = 3.5 min, ANN = 2s.
2. **1% live sampling**: turbopuffer automatically measures recall on 1% of production queries by comparing ANN vs exhaustive results. Unique to them.
3. **90-95% recall@10 target**: They monitor and alert on this threshold for all queries including filtered ones (harder than vanilla vector search).
4. **Why it matters**: Customers couldn't measure recall before turbopuffer, unknowingly got poor results. What's not measured isn't guaranteed.
5. **Performance gains passed through**: As they optimize, they increase recall/lower latency rather than pocket efficiency gains.
6. **Recall formula**: `recall@k = (# vectors in both ANN and exact top-k) / k`

**Performance napkin math** (1536-dim f32 vectors, ~3 GiB/s SSD):


| Vectors | Exhaustive | ANN   |
| ------- | ---------- | ----- |
| 1M      | 2s         | 60ms  |
| 10M     | 20s        | 350ms |
| 100M    | 3.5min     | 2s    |
| 1B      | 0.5h       | 10s   |


## Native filtering for high-recall vector search

`https://turbopuffer.com/blog/native-filtering`

**The problem:**

Most production queries have filters (e.g., `WHERE path LIKE 'foo/src/*'`), but traditional approaches fail:

- **Pre-filter**: Find matching docs → compute distances → return nearest k
  - Result: 100% recall, but O(dimensions × matches) - too slow (~10s)
- **Post-filter**: Find k nearest neighbors → filter results
  - Result: Fast (~20ms), but 0% recall if none of top-k match filter

**The solution: Native filtering**

Attribute indexes that are **aware of the vector index clustering hierarchy**:

1. Scan attribute index to identify which clusters contain matches
2. Only fetch and search those relevant clusters
3. Achieves ~90% recall with ~25ms latency

**Implementation:**

- **Two-level indexes**: Cluster-level (fast, tells which clusters have matches) + row-level (exact bitmaps)
- **LSM tree storage**: Uses `(attribute_value, cluster_id)` as key, `Set<local_id>` as value to avoid full rewrites
- **Compressed bitmaps**: Keeps index size small for fast cold queries from S3
- **SPFresh integration**: Attribute indexes react to cluster splits/merges, stay in sync with vector index

**Result:**


| Approach    | Recall  | Latency  |
| ----------- | ------- | -------- |
| Post-filter | 0%      | 20ms     |
| Pre-filter  | 100%    | 10s      |
| **Native**  | **90%** | **25ms** |


## FTS v2: up to 20x faster full-text search

`https://turbopuffer.com/blog/fts-v2`

**Two major improvements:**

1. **New index structure**: 10x size reduction on-disk, includes metadata to skip large chunks of irrelevant postings
2. **MAXSCORE dynamic pruning**: Same algorithm as Apache Lucene, scales better for long queries (agents write longer queries than humans)

**Performance gains** (5M Wikipedia documents, k=100):


| Query                                | v1    | v2   | Improvement |
| ------------------------------------ | ----- | ---- | ----------- |
| "san francisco"                      | 8ms   | 3ms  | 2.7x        |
| "the who"                            | 57ms  | 7ms  | 8x          |
| "united states constitution"         | 20ms  | 5ms  | 4x          |
| "lord of the rings"                  | 75ms  | 6ms  | 12.5x       |
| "pop singer songwriter born 1989..." | 174ms | 20ms | 8.7x        |


**Best speedups for:**

- Large datasets
- Low top_k values
- Queries with frequent terms

**New features:**

- `word_v3` tokenizer: Unicode-aware text segmentation
- Rank by filter: Conditionally boost matching docs
- Prefix queries: Search-as-you-type
- Regex filtering

**Performance comparison**: Now comparable to Tantivy and Apache Lucene.

## Vectorized MAXSCORE over WAND for long queries

`https://turbopuffer.com/blog/fts-v2-maxscore`

**The problem:**

Agents write longer queries (tens of terms) than humans. Need an algorithm that scales well with query length.

**Two algorithms for skipping documents:**

**MAXSCORE (term-centric)**:

- Sort terms by score contribution
- Identify "non-essential" terms that can't affect top-k anymore
- Use only essential terms to find candidates, all terms to compute scores
- Better throughput, decent skipping

**WAND (document-centric)**:

- Find next doc ID that could possibly qualify for top-k
- Uses iterator positions to skip more aggressively  
- Better skipping, lower throughput (more overhead per doc)


| Algorithm                        | Skipping  | Throughput    |
| -------------------------------- | --------- | ------------- |
| Exhaustive                       | None      | Very good     |
| WAND                             | Very good | Average       |
| MAXSCORE                         | Good      | Good          |
| **Lucene's vectorized MAXSCORE** | **Good**  | **Very good** |


**Why MAXSCORE wins for long queries:**

WAND's overhead scales with # of terms. On long queries, skipping power is outweighed by poor throughput. Apache Lucene initially used WAND, but switched to MAXSCORE after finding WAND was slower than exhaustive search on certain queries.

**turbopuffer's MAXSCORE optimizations:**

**Batched iterator advancement**: Instead of alternating between iterators, process many docs from same iterator in a row.

**CPU-level benefits:**

- **Memory locality**: Cache prefetcher predicts upcoming data, higher hit rates
- **Branch prediction**: CPU learns pattern, keeps instruction pipeline full
- **SIMD vectorization**: Sequential data enables parallel processing

**Trade-off parallels:**

- WAND vs MAXSCORE ≈ HNSW vs SPFresh
- WAND/HNSW: optimize skipping first, throughput second
- MAXSCORE/SPFresh: optimize throughput first, skipping second

**Result**: "Serial and dumb" beats "smart and random" on modern CPUs with AVX-512.

## Why BM25 queries with more terms can be faster

`https://turbopuffer.com/blog/bm25-scaling`

**Counterintuitive findings:**

BM25 scaling is different from vector search. Key surprises:
1. **More terms can make queries FASTER**
2. Best query at top_k=10 might not be best at top_k=10,000

**Key concept: Essential vs Non-essential terms**

**Essential terms**: Must evaluate as candidates (low posting count)
**Non-essential terms**: Can be skipped (won't affect top-k)

**Example** (200M docs, top_k=20):

| Query | Total postings | Essential terms | Latency |
|-------|---------------|-----------------|---------|
| "pop singer" | 5.8M | singer (850k) | 9.6ms |
| "pop singer songwriter" | 6.1M | songwriter (286k) | 5.5ms ✓ |

Query 3 is **faster** despite more terms because:
- "songwriter" (286k postings) is essential
- "pop" and "singer" become non-essential
- 3x fewer candidates to evaluate

**Scaling with document count:**

Modeled as `latency = C · n^K` where n = doc count, K = scaling factor

- Short queries: K = 0.44-0.70 (sub-linear)
- Long queries: K = 0.84-0.92 (near-linear)
- Longer queries scale worse as datasets grow

**Scaling with top_k:**

Modeled as `latency = C · top_k^K`

- Multiplying top_k by 10 increases latency by ~65% (very efficient)
- Lines cross: best at small top_k ≠ best at large top_k
- No correlation between # terms and how well query scales with top_k

**Takeaways:**

1. Query latency proportional to **essential term postings**, not total terms
2. Longer queries scale more linearly with doc count (worse at scale)
3. BM25 scales efficiently with top_k
4. Total term count is misleading - essential term count matters more

## Designing inverted indexes in a KV-store on object storage

`https://turbopuffer.com/blog/fts-v2-postings`

**The problem: FTS v1 cluster-based partitions**

v1 aligned posting list boundaries with vector cluster boundaries. Simple, but broke down at scale:

**Zipfian distribution**: Most terms appear in few docs
- Median partition: **1.5 postings**
- p90 partition: **11 postings**
- Result: Millions of tiny KV pairs, massive metadata overhead, poor compression

**The solution: FTS v2 fixed-size blocks**

Partition posting lists into **~256 posting blocks**, independent of vector clusters.

**Why 256?**

1. **Compression efficiency**: Uses 128-posting frames, blocks split at 512/merge at 128 (guarantees 1-4 frames per block)
2. **Query performance**: Large enough to skip meaningful work, small enough that block-max scores stay tight (more frequent skips)
3. **Storage overhead**: Amortizes metadata across enough postings to be negligible

**Implementation:**

Uses Quickwit's bitpacking (port of Daniel Lemire's simdcomp):
- **741M postings/sec** decoded throughput
- **~4.5 GB/s** (assuming 6 bytes/posting)
- **3-4x faster** than Zstd (~1 GB/s)

**Results:**

**10x smaller indexes** (40M MSMARCO docs):
- v1: 51.6 GiB
- v2: 5.22 GiB

**Better gains for less frequent terms**:

| Term | v1 size | v2 size | Reduction |
|------|---------|---------|-----------|
| "©" (most common) | 833 MiB | 340 MiB | 2.5x |
| "裏" (10,000th) | 17.8 MiB | 2.9 MiB | 6.2x |

Less common terms have fewer postings per cluster in v1 → smaller partitions → proportionally more metadata overhead.

**Combined with MAXSCORE: up to 20x faster queries**

**Future:** Will apply to infix search (%puf%), search-as-you-type, and filtering. Posting lists are generic over weight type, can substitute zero-sized type and reuse same code.

**Design philosophy:** "Earned complexity" - built simple first, observed production performance, then optimized based on customer workloads.

## ANN v3: 200ms p99 over 100 billion vectors

`https://turbopuffer.com/blog/ann-v3`

**Goal:** 100B vectors, 1024D f16, >1k QPS, 200ms p99 latency = **200 TiB of dense vector data**

**Key insight: Vector search is bandwidth-bound**

Arithmetic intensity analysis:
- Vector dot product uses each byte once
- Low arithmetic intensity → bandwidth-bound, not compute-bound
- Bottleneck is fetching data vectors, not computing distances

**Memory hierarchy bottlenecks:**

```
CPU Registers  < 1 KB      >10 TB/s
L1/L2/L3       KBs-MBs     1-10s TB/s
DRAM           GBs-TBs     100-500 GB/s
NVMe SSD       TBs-10s TB  1-30 GB/s
Object Storage PBs-EBs     1-10 GB/s
```

**Technique 1: Hierarchical clustering (SPFresh)**

Multi-level tree with 100x branching factor:
- Bounds object storage round-trips to tree height
- **Spatial locality**: Nearby vectors stored contiguously
- **Temporal locality**: Upper levels stay in DRAM naturally
- Scan ~500 clusters × 100 vectors = 100MB per level

**Without quantization:**
- Centroid vectors (upper): DRAM, 300 GB/s → **1,000 qps max**
- Data vectors (lowest): NVMe, 10 GB/s → **100 qps max** ← bottleneck

**Technique 2: Binary quantization (RaBitQ)**

1 bit per dimension = **16-32x compression**:

```
[ 0.94, -0.01,  0.39, -0.72, ... ]
              ↓
         [ 1, 0, 1, 0, ... ]
```

**RaBitQ properties:**
- Exploits concentration of measure in high dimensions
- Returns distance estimate ranges (e.g., [0.69, 0.83])
- <1% of vectors need full-precision reranking
- Each bit reused 4 times → **64x higher arithmetic intensity**

**With quantization:**
- Quantized centroids (upper 3 levels): **L3 cache**, 600 GB/s → **33,000 qps**
- Quantized data (lowest): **DRAM**, 300 GB/s → **50,000 qps**
- Full precision rerank (1%): NVMe, 10 GB/s → **10,000 qps** ← new bottleneck

**Plot twist: Became compute-bound**

64x arithmetic intensity shift → saturates at **~1,000 qps (compute-bound)**

**Optimizations:**
- Switch to AVX-512 for VPOPCNTDQ (popcount instruction)
- 3 CPU cycles latency, 1 instruction/cycle throughput
- 30% microbenchmark improvement → 5% production gain

**Distribution: Random sharding**

Storage-dense VMs (GCP z3, AWS I7i, Azure Lsv4):
- 10-40 TiB per machine
- Random shard assignment during ingestion
- Broadcast queries, stitch global top-k
- Scales linearly with machines

**Architecture decisions:**
1. Hierarchical clustering matches memory hierarchy tiers
2. Binary quantization enables 16-32x compression
3. RaBitQ maintains recall with minimal reranking
4. Maximize single-machine efficiency before distributing

**Result:** 200ms p99 at 1k+ QPS over 100B vectors in production

## Building a distributed queue in a single JSON file on object storage

`https://turbopuffer.com/blog/object-storage-queue`

**Use case:** Indexing job queue that notifies indexing nodes after WAL writes. Not on write path, purely async notification system.

**Result:** 10x lower tail latency vs previous sharded-per-node design (slow nodes no longer block jobs).

**Evolution (earned complexity):**

**Step 1: Simple queue.json**
- Single file with full queue contents
- Pushers/workers use **CAS (compare-and-set)** to atomically modify
- CAS: write only succeeds if file hasn't changed since read
- Works great up to **1 RPS** (GCS limit)

**Step 2: Add group commit**
- Object storage writes: ~200ms latency
- **Buffer requests in memory** while write is in flight
- Flush buffer as next CAS write when current completes
- **Decouples write rate from request rate**
- Bottleneck shifts: ~200ms/write → network bandwidth (~10 GB/s)
- Problem: Still ~5 writes/sec max (CAS forces non-overlapping writes)

**Step 3: Add stateless broker**
- Single broker handles ALL interactions with object storage
- Eliminates contention (no more multiple writers)
- Runs group commit loop on behalf of all clients
- Doesn't ack until durably committed to object storage
- Broker bottleneck: easily serves hundreds/thousands of clients (just buffering + I/O)

**Step 4: Add HA (high availability)**

**Broker failover:**
- Write broker address to `queue.json`
- If request to broker times out → start new broker
- Stateless → easy to move
- CAS ensures correctness even with 2 brokers (old one gets CAS failure)

**Job heartbeats:**
- Workers periodically send timestamp to broker
- Broker writes heartbeat to `queue.json` for each claimed job
- If heartbeat > timeout → assume worker dead, next worker takes over
- **At-least-once delivery guarantee**

**Key primitives:**
- **CAS (compare-and-set)**: Atomic updates without locks
- **Group commit**: Batch writes to amortize latency
- **Stateless broker**: Single writer eliminates contention
- **Heartbeats**: Detect failed workers

**Philosophy:** "Simple, predictable, easy to be on-call for, and extremely scalable. We know how it behaves."

## Rust zero-cost abstractions vs. SIMD

`https://turbopuffer.com/blog/zero-cost`

**Problem:** Customer query with thousands of `ContainsAny` filter values taking 220ms, but only 10ms on BM25 ranking → 200ms+ on filter evaluation.

**Napkin math:** 67MB of filter bitmaps at 6,240 MB/s NVMe → should take 10-20ms, not 200ms+.

**Root cause: LSM merge iterator**

`ContainsAny` with thousands of values → thousands of disjoint key ranges → merge iterator consuming **60% of runtime**.

**The "zero-cost" trap:**

```rust
// Original: next() returns one value at a time
while let Some(v) = merge_tree.next() {
    let x = v + 1;
    if x % 2 == 0 { sum += x; }
}
```

**Expected:** 100K values × 4 instructions ÷ 3 GHz = **130μs**
**Actual:** **6.5ms** (50x slower!)

**Why?**
- Iterator compiles to tight 7-instruction loop (zero-cost per call)
- But recursive `next()` calls prevent compiler from:
  - **Loop unrolling** (can't predict next value until current finishes)
  - **SIMD vectorization** (values arrive one at a time)
- Abstraction boundary between calls hides optimization opportunities

**"Zero-cost" means:** Abstraction compiles away for single call
**"Zero-cost" doesn't mean:** No opportunity cost preventing cross-call optimizations

**Solution: Batched iterators**

```rust
// Batched: next_batch() returns 512 values at once
while let Some(batch) = merge_tree.next_batch() {
    for val in batch {  // tight inner loop over plain array
        let x = val + 1;
        if x % 2 == 0 { sum += x; }
    }
}
```

**Assembly shows SIMD:**
- Original: 7 scalar instructions per value
- Batched: Processes **8 values at once** using SIMD instructions (`.2d`, `.16b`)

**Results:**

**Microbenchmark** (100K values):
- Before: 6.5ms
- After: **110μs** (60x faster, beats napkin math due to SIMD)

**Production query:**
- Before: 220ms
- After: **47ms** (4.7x faster)

**Key insight:** Modern CPUs excel at "dumb and serial work" but need to see the loop shape to apply SIMD. Batching exposes contiguous data to compiler, amortizes merge cost across 512 KV pairs.

**Design philosophy:** "Production profiles trump theory and abstraction. Mechanical sympathy required."

## Ranking by attribute: Mixing numeric attributes into text search

`https://turbopuffer.com/blog/rank-by-attribute`

**The multi-stage search problem:**

```
Corpus (1M-1B) --BM25--> Candidates (100-1K) --Reranker--> Results (10)
```

- **Stage 1 (BM25)**: Fast, indexed, scales to 100M+ docs
- **Stage 2 (Reranker)**: Expensive inference (cross-encoder, LLM), careful reasoning

**Problem:** When relevance depends on non-text attributes (recency, PageRank, sales volume), BM25-only first stage breaks down.

**Example:** Email search for "weekly status meeting"
- User wants THIS week's meeting
- BM25 ranks by text match only → old email with nested threads ranks higher
- Yesterday's email might be 500th (outside candidate set)
- **Reranker cannot rescue what first stage never surfaces**

**Solution: Rank by attribute in first stage**

Based on 2005 paper by **Stephen Robertson** (co-author of BM25): Use **sigmoid functions** to convert attribute values into score contributions comparable to BM25 term scores.

**Example query:**

```javascript
{
  "rank_by": ["Sum", [
    ["body", "BM25", "weekly status meeting"],
    ["Product", 1.5,
      ["Decay",
        ["Dist", ["Attribute", "date"], new Date()],
        { "midpoint": "30d" }
      ]
    ]
  ]]
}
```

**Compiled internally:**

```rust
score = 1.0 * bm25(body, "weekly")
      + 1.0 * bm25(body, "status") 
      + 1.0 * bm25(body, "meeting")
      + 1.5 * attribute(date, f(x) = 30d / (x + 30d))
```

**Key functions:**

- **Decay**: Sigmoid falling with value (e.g., recency decay)
- **Saturate**: Sigmoid rising with value (e.g., popularity boost)
- Both produce bounded output [0, 1]
- `midpoint`: Steepness parameter
- `Product`: Weight (max score contribution)

**How it scales:**

Attribute clauses run through the **same vectorized MAXSCORE** as BM25 terms:
- Attribute clause sits alongside BM25 term clauses
- MAXSCORE heap uses combined scores for pruning
- Aggressive skipping still works
- Scoring function applied at query time (tune without reindexing)

**Benefits:**

1. First stage now tracks full relevance objective (text + attributes)
2. No overfetching to compensate for BM25-only limitations
3. Compact, relevant candidate list for reranker
4. Scales efficiently to 100M+ corpus sizes
5. Latency increase comparable to adding more query terms

**Design philosophy:** Pull key attributes (1-2 per dataset) into first stage to boost their signal at scale while maintaining BM25's efficiency characteristics.

