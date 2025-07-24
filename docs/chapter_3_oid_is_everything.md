Title: Chapter 3 — OID is Everything — Semantics at CPU Speed

---

### 3.1 The Problem with Names

In traditional enterprise systems, we often rely on human-readable identifiers like `PartNumber`, `DocumentName`, or `TNR`. These identifiers are intuitive — but not performant. Behind every name lies an expensive lookup, a database index, a join, or worse — a cache miss.

MatrixOne takes a radically different stance. It treats identity not as a user construct, but as a system primitive — and encodes it directly as an opaque 4-long structure called **OID**.

---

### 3.2 What is an OID?

```
OID = vault_hi.vault_lo.seq_hi.seq_lo
        ↓       ↓       ↓       ↓
        │       │       │       └── sequenceLow  (object ID low bits)
        │       │       └────────── sequenceHigh (object ID high bits)
        │       └────────────────── vaultLow     (vault ID low bits)
        └────────────────────────── vaultHigh    (vault ID high bits)
```

An OID in MatrixOne is composed of four long integers, separated by dots — for example:

```
3968.54152.63573.4155
```

These four integers are interpreted as two tightly packed 32-bit integers:

- The first two segments represent the **vault ID**, calculated as: `(vault_hi << 16) | vault_lo`
- The last two segments represent the **object ID**, calculated as: `(seq_hi << 16) | seq_lo`

These values are used as follows:

- The **vault ID** is formatted as a hexadecimal suffix to identify the vault-specific table name:
  ```sql
  lxbo_0f80d388
  ```
- The **object ID** becomes the `lxoid` primary key within that table. In Java-style signed 32-bit math, this value may appear negative:
  ```sql
  id = -128643013
  ```

Full example:

```yaml
OID:         3968.54152.63573.4155
vault_hi:    3968
vault_lo:    54152
→ vault_id = (3968 << 16) + 54152 = 260101000 = 0x0f80d388

seq_hi:      63573
seq_lo:      4155
→ object_id = (63573 << 16) + 4155 = -128643013  (signed 32-bit view)
```

This enables the engine to perform fully deterministic resolution with no lookup tables or name-based joins:

```sql
SELECT * FROM lxbo_0f80d388 WHERE id = -128643013;
```

This mechanism supports high-throughput, index-resident lookups at CPU speed — fast, locality-aware, and cache-friendly.

---

### 3.2.1 Encode / Decode Utilities (Python)

The following logic illustrates how to encode and decode a real-world OID like:

```
OID = 3968.54152.63573.4155
```

This is interpreted as:

- Vault ID = `(3968 << 16) | 54152` = `260101000` = `0x0f80d388`
- Object ID = `(63573 << 16) | 4155` = -128643013
- In signed 32-bit view (Java/C-style): `-128643013`

Corresponding SQL lookup:

```sql
SELECT * FROM lxbo_0f80d388 WHERE id = -128643013;
```

When using Java or C, you must use long-compatible types carefully — but the correct result for MatrixOne should **match the signed 32-bit integer** as stored in the database:

```c
// In C or Java (signed 32-bit view)
int oid = (63573 << 16) | 4155;   // ✅ yields -128643013 — matches database

// In Java, using long avoids overflow, but gives unsigned value
long oid = ((long) 63573 << 16) | 4155; // ⚠️ yields 4166324283 — does not match database record
```

MatrixOne’s MQL engine is implemented in C/C++, and follows signed 32-bit logic when interpreting OID components. Therefore, **the version yielding `-128643013` is correct** for all runtime queries and comparisons.
`

Python implementation:

```python
def encode_oid(vault_hi, vault_lo, seq_hi, seq_lo):
    vault_id = (vault_hi << 16) | vault_lo
    object_id = (seq_hi << 16) | seq_lo
    return vault_id, object_id

def decode_oid(vault_id, object_id):
    vault_hi = (vault_id >> 16) & 0xFFFF
    vault_lo = vault_id & 0xFFFF
    seq_hi = (object_id >> 16) & 0xFFFF
    seq_lo = object_id & 0xFFFF
    return vault_hi, vault_lo, seq_hi, seq_lo
````

Use `long` types (not `int`) when implementing in Java or C to avoid misinterpreting IDs as negative numbers. In Java:

```java
long oid = ((long) hi << 16) | lo;
```

Avoid:

```java
int oid = (hi << 16) | lo; // ❌ May produce negative numbers!
```

---

### 3.3 Why OID is Semantically Powerful

Unlike a database primary key, OID does more than identify:

- It conveys **physical location** (but not type) at runtime
- It allows for **direct CPU-level computation** of object presence
- It eliminates the need for joins to determine basic object identity

This means that the entire resolution process becomes:

```
predicate → OID list → memory-resident fetch
```

It’s not just fast — it’s structure-aware.

> **Note:** While OIDs encode physical access paths, they do **not** directly capture user-level semantics — such as `Type`, `Policy`, or `Classification`. These are resolved separately via vault-local tables like `lxbo_XXXX`.
>
> However, OID **does** embody a form of structural semantics — it tells the system where the object lives and how to reach it without further interpretation. In this sense, OID is not a semantic label, but a semantic accelerator.

---

### 3.4 Comparisons: Oracle ROWID and Snowflake ID

\$1

MatrixOne’s OID stands in that lineage — but applies it to a multi-table, semantically structured PLM world.

However, one should be cautious when interpreting OID beyond its intended scope.
Because it reflects **physical deployment structure**, OID can change across migrations, replications, or refactoring.

Newcomers to MatrixOne may mistakenly persist OIDs as attributes to establish relationships — instead of using `connect` relationships.
This can become catastrophic if vaults are reassigned or data is reshuffled.

Such misuse often stems from a fundamental misunderstanding of **layered architecture** — especially the distinction between logical relationships and physical identifiers. Without a mental model of MatrixOne's **deployment topology**, engineers may treat OIDs as stable foreign keys, which they are not.

> This is not hypothetical — real-world systems (including some OOTB logic) have exhibited such misuse. — but applies it to a multi-table, semantically structured PLM world.

---

### 3.5 TNR vs OID: Names for Users, Bits for Machines

TNR (Type-Name-Revision) in MatrixOne refers to the fields within the `lxbo` logical object table. Since there may be multiple vaults, each associated with its own set of tables, the `lxbo` table actually appears as suffixed forms like `lxbo_0BCEFD` — where the suffix encodes the vault ID in hexadecimal, as defined in the `mxlattice` vault table. Similar vault-specific suffixing applies to related tables such as `lxro_0BCEFD`, `lxstring_0BCEFD`, `lxdate_0BCEFD`, etc.

Depending on system configuration, TNR values may or may not be globally unique across vaults. This is a configurable constraint, and the system does not enforce global uniqueness by default.

While TNR makes for an excellent external API or human-facing contract, it is not the system’s foundation. MatrixOne treats TNR as a “decorated view” over OID.

By decoupling user naming from physical identity, the platform enables:

- Clean separation of UI and storage
- Index-free resolution (if OID is known)
- Batch operations via sorted OID ranges

---

### 3.6 When OID Enables Performance Patterns

The power of OID-first design reveals itself in:

- **Expansion queries**: where child objects can be filtered by OID prefix

> ⚠️ When parent and child reside in **different vaults**, expansion relations are written into each vault's `lxro_XXXX` table separately. This increases maintenance overhead and risks locality loss. Most performant scenarios assume parent and children share the same vault.

- **Security scoping**: checking access rights via direct OID scan
- **Memory cache population**: indexed by OID for near-zero lookup time

Once predicates resolve to OIDs, the rest of the system simply follows — no SQL, no joins, no joins pretending to be joins.

> “OIDs are not just faster keys — they are design philosophy materialized.”

---

### 3.7 Room for Modernization?

Before we talk about modernization, let’s revisit a subtle but fascinating detail about how MatrixOne assigns OIDs.

When creating a new business object, the engine often sends **four simultaneous insert statements** to the internal `lxoid` table. The purpose is not to store four objects — but to preemptively reserve a usable OID.

This “multi-insert for OID” trick likely reflects a kind of **fault-tolerant design**: instead of waiting for a round-trip retry if one insert fails, the engine sends four at once and uses the first successful result, discarding the rest.

Some might worry about waste — but in PLM systems, object creation is infrequent compared to internet-scale workloads. And unused rows can always be cleaned up during maintenance windows.


While the “multi-insert for OID” approach reflects a solid high-availability mindset, modern distributed systems may favor:

- Background allocation pools
- Dedicated UID generation services (à la Snowflake)
- Hash-based ID semantics in low-conflict domains

That said, the current strategy remains surprisingly effective — not due to technical novelty, but because it honors the context: low write volume, high determinism, and CPU-level access locality.

This is not about upgrading for trend’s sake — but understanding what already works, and when evolution makes sense.

---

Refer to Series Foreword for scope and disclaimer.

---

### ✍️ Author’s Note

My understanding of the OID structure didn’t come from documentation — because none was available. Around 2013–2014, driven by pure curiosity and the need to troubleshoot performance behavior in real systems, I reverse-engineered the logic behind OID composition through observation, trial, and tooling.

This culminated in a short encode/decode note I shared internally at Sony Nordic, where we had been running large-scale MatrixOne deployments.

Later, while exploring Oracle's internal architecture (especially through DSI materials), I began to appreciate this design not just as a convenience, but as a form of **deployment-aware identity encoding** — a structure that blends storage abstraction with runtime locality awareness.

What struck me most was how early this design emerged.

Long before systems like **Snowflake** popularized structured IDs, or **MySQL** users traded tips on `ip2int` and `int2ip`, MatrixOne had already embedded system topology, vault partitioning, and access semantics into a compact, bitwise OID — hiding power in plain sight.

> The beauty of this design lies not in convenience, but in **intentional abstraction**.

This note is not meant as personal display, but as a reminder:

> The goal of tracing back design is not self-validation — but the pursuit of underlying structure.

