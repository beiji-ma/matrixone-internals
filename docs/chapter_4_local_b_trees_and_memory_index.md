Title: Chapter 4 — Local B-Trees and the Memory-Resident Index

### 4.1 What is a Local B-Tree?

Most deployments of MatrixOne rely on the `rip` mode of the ematrix server, where the local B-Tree is embedded directly within the same process space as the application server — typically a Java servlet container such as Tomee Plus. This co-location allows the B-Tree index to reside in shared memory, avoiding the performance overhead of remote access (as seen in older `rmi` setups).

Though ephemeral, this memory-resident structure may optionally support local persistence (e.g., memory dumps) for diagnostic or warm-start purposes. However, its primary role is **runtime acceleration**, not durability.

In MatrixOne, a **Local B-Tree** refers to a memory-resident index structure scoped to the session or user context. It tracks already-resolved OIDs (Object Identifiers) and maps them to lightweight metadata — such as object headers, vault location, or select attributes.

Unlike persistent database indexes, Local B-Trees are ephemeral: they exist only for the duration of a session or process. Yet they are crucial for enabling **low-latency object resolution**, avoiding repeated lookups to the underlying `lxbo_XXXX` and related tables.

These trees are populated on demand — or prefilled based on query results, expansion traversals, or scripted preload. In essence, they act as a runtime “cache-with-index”, offering:

- Constant-time membership tests
- Fast iteration over known OIDs
- In-memory filtering before triggering DB roundtrips

MatrixOne's B-Tree is not merely indexing full object records. Instead, it orchestrates memory-resident routing to specific database tables based on attribute type. For instance:

- `lxbo_XXXX` holds object core records (headers)
- `lxstring_XXXX` stores string attributes
- `lxdate_XXXX` holds date values
- `lxint_XXXX`, `lxreal_XXXX`, etc. for numerical types

This reflects a **columnar storage layout**, where attributes are decoupled by type for space and I/O efficiency.

When resolving an OID, the engine consults the B-Tree to determine what attributes are already in memory and which tables must be consulted. If an attribute is missing (e.g. due to lazy loading or filtering), the corresponding column-table (e.g. `lxstring_XXXX`) is queried selectively, often by OID.

This design enables very efficient memory usage and supports fine-grained caching at the attribute level.

Next, we’ll explore how this index powers the OID-first execution model.

### 4.2 OID-first Execution and Local Resolution

MatrixOne employs a distinct query execution model: **predicate first, data later**. Rather than navigating object graphs or issuing joined queries to retrieve both structure and data in one sweep, it first identifies the relevant OIDs based on filters and relations — and only then pulls full data into memory if needed.

This "OID-first" pattern offers several key advantages:

- **Selective I/O**: By narrowing down the target OIDs early, the system avoids scanning unrelated records or fetching full rows prematurely.
- **Predictable access pattern**: OID resolution occurs via local B-Tree lookup first. If the object is already indexed locally, resolution is instant.
- **Decoupled phases**: Planning (identify what to fetch) is cleanly separated from evaluation (actually fetch it), which improves modularity and caching.

#### Example:

```mql
list bus * * * select id,policy where policy == "Engineering"
```

This query first resolves all matching object headers using the OID filter. If the result is large, only object IDs and policies are loaded. Further commands (e.g., `print`, `expand`, or scripting operations) can trigger deeper object loads **from memory**, not DB.

This is why even “list” and “print” in MatrixOne are not mere output tools — they participate in staged resolution of objects, leveraging the B-Tree to avoid redundant retrievals.

The result is a highly efficient, stepwise execution model — very different from ORM-based object graph traversal.

### 4.3 When Logs Say "Full Scan"

It's common for newcomers to panic when trace logs say: `Executing full scan on table lxbo_XXXX`.

But this is **not** what it seems. In many cases, this message simply indicates a bulk load or prefetch operation — triggered by a query that intentionally lacks filtering (e.g., `list bus * * *`).

The goal isn’t to “scan” in the SQL sense, but to populate the **local B-Tree index** for subsequent in-memory evaluation.

#### Analogy:

- Think of it like an OS page fault: a segment of memory is loaded because it's needed soon.
- Or like Oracle’s buffer cache: reads from disk fill memory so that future reads are cheap.

What really matters is:

- How well the filtering predicates narrow down candidate OIDs
- Whether follow-up stages benefit from locality

In short, "scan" in MatrixOne often means **prefetch**, not inefficiency.

### 4.4 Volcano Model Analogy

MatrixOne’s staged evaluation and OID-first resolution may feel unfamiliar at first, but they align remarkably well with the classic **Volcano Model** of query execution.

In the Volcano Model (Selinger et al.), each operator pulls data from its child via an iterator-like interface. This lazy, demand-driven execution avoids materializing intermediate results.

MatrixOne doesn’t implement iterators per se, but conceptually it behaves similarly:

- OID resolution acts as an access-path operator
- Local B-Trees serve as memory-level filters
- Object materialization happens only when explicitly needed (e.g. `print`, script, or downstream `expand`)

This staged approach makes MatrixOne:

- Modular — each phase (filtering, resolving, fetching) is separable
- Optimizable — future designs can intercept stages with smart caching or rewriting
- Predictable — no hidden joins or graph walks

In Chapter 5, we’ll zoom into how this conceptual model is implemented via a real execution pipeline.

### 4.5 Misuse Patterns: When B-Trees Get Polluted

A well-structured local B-Tree can be a performance multiplier — but a poorly used one can just as easily degrade the session.

#### ❌ Example: Misuse of `expand` with unbounded filters

```mql
expand bus * * * to relationship * select name,policy
```

This query is deceptively simple, but introduces multiple anti-patterns:

- Wildcards (`* * *`) match too many objects — potentially thousands or millions
- `relationship *` traverses all connected objects without constraint
- Though `select` is used, it still triggers materialization for all expanded objects

What follows is excessive OID loading, mass object lookup, and memory bloat. Worse, the B-Tree cache is now polluted with data unlikely to be reused — breaking cache locality.

#### ✅ Better pattern:

```mql
expand bus Product A 1 to relationship EBOM select name,policy
```

Restricting source objects and relationship names ensures the B-Tree grows predictably.

#### OLAP vs OLTP Mindset

In data migration or reporting (OLAP), full scans and mass loading may be expected. But in transactional flows (OLTP), preloading too much — especially for queries expected to return **only a few matching objects** — breaks interactivity and bloats memory needlessly.

#### Takeaway:

Don't fear `full scan` in trace logs — fear **meaningless scans that destroy locality**.

### 4.6 Summary & Design Implications

While the benefits of local B-Trees are clear, they also raise deployment concerns in multi-instance setups.

### 4.7 Reflection: Local Caches in Multi-Instance Deployments

In real-world environments, MatrixOne is often deployed with multiple application server instances — either on a single machine or across a cluster — to support load balancing, failover, or scalability.

However, in RIP mode (where the ematrix server runs within the application container), each instance maintains its **own independent B-Tree**, scoped to its local process.

#### Implications:

- **No Shared Cache:** Each instance must warm up its own B-Tree. There's no inter-process memory sharing, which leads to redundancy.
- **Uneven User Experience:** Queries might be fast on one server (with a hot cache) and slow on another (cold cache), confusing users.
- **Monitoring Complexity:** Cache hit/miss rates vary by instance, making performance tuning difficult.
- **Loss of Warmth:** In distributed environments, frequent re-routing means B-Trees are always “cold”, undermining one of their core advantages.

#### Experimental Suggestion:

This behavior can be verified through concurrent load testing on two separate RIP instances: observe whether their B-Tree states diverge over time despite receiving similar requests.

#### Room for Modernization:

- Introduce optional B-Tree state persistence (memory snapshots or lazy journals)
- Explore lightweight shared-index services (e.g., gRPC microservice for B-Tree sync)
- Apply sticky sessions to preserve locality where applicable

MatrixOne’s use of Local B-Trees embodies a design philosophy that separates **structure resolution from data loading**, allowing for tight control over what is fetched, when, and why.

Key takeaways from this chapter:

- Local B-Trees are **session-resident indexes**, optimized for resolving object identities (OIDs) quickly
- Queries follow a **staged execution model**, where filtering and expansion are separated from materialization
- Apparent “full scans” are often **controlled prefetches**, not performance bottlenecks
- The architecture aligns conceptually with **Volcano-style query execution**, where each stage pulls data on demand

This architecture supports high interactivity and makes runtime optimization feasible — provided the user avoids patterns that pollute the cache.

In the next chapter, we’ll dive into how this design manifests in the core engine: the MatrixOne Execution Pipeline.

