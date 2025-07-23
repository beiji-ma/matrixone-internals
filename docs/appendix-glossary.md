# Appendix — Glossary of Terms (Shared with Series A and B)

This glossary consolidates recurring terms, abbreviations, and concepts used throughout both **Series A** and **Series B** in the *matrixone-internals* project. It provides semantic clarity and helps reduce repetition across chapters.

---

### A
- **AEF (Application Exchange Framework):** MatrixOne's client-side framework for building enterprise UIs.
- **API (Application Programming Interface):** A set of programmatic interfaces, often contrasted here with declarative DSL usage.

### B
- **BOM (Bill of Materials):** A structured representation of components in an assembly; used heavily in PLM.
- **BTREE (Binary Tree):** Memory-resident structure for indexing and filtering OIDs in MatrixOne.

### C
- **CRUD (Create, Read, Update, Delete):** The four basic operations of persistent storage systems, often seen as inadequate for PLM use cases.
- **Cue:** A component in thick-client UI configuration defining how query results are shown.

### D
- **DSL (Domain-Specific Language):** A language tailored to a narrow set of operations, such as MatrixOne’s MQL.
- **DTM (Data Transaction Manager):** Component responsible for coordinating data-level operations across modules.

### E
- **Expand:** A MatrixOne operation for traversing object relationships.
- **EMatrix Server:** The core server executable in MatrixOne that runs as either RMI or RIP.

### F
- **Filter:** A term for result refinement applied after OID resolution, often in memory, not at the DB level.
- **Full Scan:** A misunderstood trace label that can indicate cache warming, not inefficiency.

### G
- **GUI (Graphical User Interface):** Used here in contrast to DSL/command-line interfaces.

### I
- **Indexing:** Performed in-memory via B-Trees in MatrixOne; distinct from DB indexes.

### J
- **JPO (Java Program Object):** A server-side Java extension mechanism in MatrixOne.
- **Join (SQL-style):** Explicit linking of tables, often discouraged in MQL design.

### L
- **Lxbo_XXXX:** Vault-specific logical object table.
- **Lxro_XXXX:** Vault-specific relationship object table.

### M
- **MQL (Matrix Query Language):** The embedded DSL used for data manipulation in MatrixOne.
- **MatrixOne:** Dassault Systèmes' PLM platform formerly known as ENOVIA MatrixOne.

### O
- **OID (Object ID):** Bit-packed physical identity used as a pointer and access handle.
- **ORM (Object-Relational Mapping):** A common source of design mismatch in PLM when overused.

### P
- **Predicate:** A filter condition applied before or during OID resolution.
- **Policy:** Access control mechanism for types and actions.

### Q
- **Query Pipeline:** The structured execution model from predicate to result.

### R
- **RIP (RMI Integrated Process):** A deployment mode where MatrixOne server shares memory space with servlet engine.
- **RMI (Remote Method Invocation):** Java’s traditional inter-process communication method.
- **ROWID:** Oracle’s physical record identifier, a conceptual ancestor to MatrixOne’s OID.

### S
- **SOV (Secondary Ownership Vector):** Mechanism for scoped permission delegation.
- **Snowflake ID:** Widely used distributed UID format; compared to MatrixOne’s OID in concept.

### T
- **TNR (Type-Name-Revision):** Human-readable object identity; less performant than OID.
- **Trace:** Execution-level logging used for diagnosis.

### V
- **Vault:** Logical shard in MatrixOne; each with its own set of physical tables.

---

> ✳️ Suggest additions or refinements via issues or PRs — this glossary evolves with the project.

