# Prompt: RDM Legacy → New Service Parity & Optimization Review

> Fill in the `<< >>` placeholders before running. Attach/point to both folders.

---

## PROMPT STARTS HERE

You are a senior backend architect and code auditor specializing in banking reference data platforms.

### Context

I own a **Reference Data Management (RDM)** service that serves bank reference data — banks, branches, locations, countries, currencies, and similar master data.

- **Legacy service**: built on the **Crank framework + Spring**. Path: `<<LEGACY_FOLDER_PATH>>`
- **New service**: independently rebuilt by the team as a **Spring Boot REST service** with its own endpoints, validations, and error handling. Path: `<<NEW_FOLDER_PATH>>`

**Critical constraint:** There is **no formal requirements or BRD document** for this rebuild. The only stated requirement is *"the new service must behave exactly as the current service behaves."* Therefore **the legacy codebase IS the specification.** Every behavior in the legacy code is a requirement until proven otherwise.

### Your task

Perform an exhaustive, evidence-based parity audit of the new service against the legacy service, then recommend technical improvements and a caching design. Produce the result as a **single self-contained HTML report**.

### Method — do this in order

1. **Read the legacy folder completely first.** Build an inventory of every capability before you look at the new code. Do not let the new code's structure bias what you look for.
2. **Then read the new folder completely.**
3. **Then compare**, bidirectionally: legacy features missing in new, AND new behavior with no legacy counterpart (unintended additions are also risk).
4. Cite **file path + line/method reference** as evidence for every finding. If something cannot be determined from the code, mark it explicitly as `UNKNOWN — needs confirmation` rather than guessing.

### Analysis dimensions — cover every one

**1. Surface inventory & endpoint mapping**
- Every legacy operation/action/handler → corresponding new REST endpoint.
- Coverage matrix: Covered / Partially covered / Missing / New-only.
- HTTP verb, path, and semantic correctness (is a legacy read now a POST? is a bulk op now single-item?).

**2. Request/response contract parity**
- Field-by-field mapping for each entity (bank, branch, location, country, etc.).
- Field names, data types, nullability, defaults, date/time formats, number precision, string trimming/casing.
- Wrapper/envelope differences, nesting changes, renamed or dropped fields.
- Pagination, sorting, filtering, search semantics (exact vs partial match, case sensitivity, wildcards).

**3. Business rules & validation parity**
- Mandatory vs optional fields; length, format, regex, and range rules.
- Cross-field and conditional rules; code/lookup validations; referential integrity checks (e.g. branch must belong to an existing bank).
- Uniqueness constraints, duplicate detection.
- Status/lifecycle transitions, active/inactive handling, soft delete vs hard delete.
- Effective dating / temporal validity, versioning, maker-checker or approval workflow if present.
- Default value assignment and derived/computed fields.

**4. Data & persistence parity**
- Entity ↔ table mapping, column-level differences, missing columns.
- Query semantics: joins, filters, ordering, null handling, implicit DB-level defaults.
- Transaction boundaries, isolation, rollback behavior, batch/bulk semantics.
- Any legacy logic that lived in stored procedures, SQL, or the Crank framework itself and may have been silently lost in the rewrite. **Flag framework-provided behavior specifically** — this is the highest-risk category in a framework migration.

**5. Error handling & fault behavior parity**
- Every legacy error condition → new equivalent.
- Error codes, error messages, message keys/i18n, HTTP status mapping, error response shape.
- Exception hierarchy and global handler coverage; unmapped exception → 500 leakage.
- Partial-failure behavior on bulk operations (fail-fast vs collect-all-errors).
- Validation error aggregation: does the new service return all errors or just the first?

**6. Cross-cutting / non-functional parity**
- AuthN/AuthZ, role and entitlement checks, field-level or record-level access control.
- Audit trail, who/when stamps, change history.
- Logging: levels, correlation/trace IDs, PII exposure in logs.
- Concurrency control (optimistic locking, version columns), idempotency of writes.
- Timeouts, retries, downstream integrations, event/message publishing.
- Configuration and environment-specific behavior.

**7. Gap register**
Produce a single consolidated table of every discrepancy:

| ID | Area | Legacy behavior (evidence) | New behavior (evidence) | Gap type | Severity | Business impact | Recommended fix |

- Severity: **Critical** (data corruption / wrong financial reference data / security), **High** (functional regression visible to consumers), **Medium** (behavioral drift, contract change), **Low** (cosmetic/internal).
- Sort by severity descending.

**8. Technical improvement recommendations**
Independent of parity — what should be improved in the new service:
- API design and REST maturity, versioning strategy, backward compatibility for existing consumers.
- Layering, separation of concerns, duplication, testability.
- Validation approach (Bean Validation vs imperative), error model consistency (e.g. RFC 7807 Problem Details).
- Query efficiency, N+1 problems, projections/DTOs, indexing.
- Observability, resilience, connection pooling, payload size, bulk endpoints.
- Test coverage gaps. For each: current state → recommendation → effort (S/M/L) → benefit.

**9. Caching design for reference data**
Reference data is read-heavy and changes rarely — design the caching strategy in detail:
- What to cache: entity-level, list/query results, lookup maps, full dataset preload.
- Cache topology: local (Caffeine) vs distributed (Redis) vs two-tier near-cache; when each is appropriate here.
- Cache key design, including tenant/locale/version dimensions.
- TTL vs explicit invalidation; write-through vs cache-aside.
- **Invalidation on CRUD** — precise rules for which keys to evict when a bank/branch/country is created, updated, or deactivated, including dependent/derived entries.
- Cross-instance invalidation (pub/sub, event-driven) in a multi-pod deployment.
- Startup warm-up / preloading, and behavior on cold cache.
- Cache stampede protection, negative caching, null handling.
- Consistency guarantees and acceptable staleness for banking reference data — call out where staleness is NOT acceptable.
- Memory sizing estimate approach, eviction policy, metrics and hit-ratio monitoring.
- HTTP-level caching (ETag / Last-Modified / Cache-Control) for consumers.
- Concrete Spring implementation guidance (`@Cacheable` / `CacheManager` config) with sample code.
- Also review `<<CACHE_FOLDER_PATH>>` (existing/proposed cache code) and assess it against the above.

**10. Verification & test plan**
- Prioritized parity test cases derived from the gaps found.
- Recommend a shadow/diff-testing approach: replay real requests against both services and diff responses.
- Edge cases and negative cases most likely to break.

**11. Assumptions & open questions**
- Everything you could not determine from code, phrased as a specific question for the team.

### Output format

Produce **one self-contained HTML file** (inline CSS, no external dependencies) containing:

- Title, generation date, and a scope statement listing what was analyzed.
- **Executive summary** at top: overall migration readiness verdict, counts by severity, and the top 5 risks in plain business language.
- Sticky/side navigation linking to each section.
- Color-coded severity badges (Critical=red, High=orange, Medium=amber, Low=grey).
- Sortable/readable tables for the coverage matrix, field mappings, and gap register.
- Collapsible sections for long detail blocks.
- Code snippets in styled `<pre>` blocks with legacy vs new side by side where useful.
- Print/PDF-friendly styling.

### Rules

- **Do not assume parity.** Absence of evidence is a finding, not a pass. If you cannot locate the new equivalent of a legacy behavior, report it as a gap with `UNKNOWN` confidence rather than omitting it.
- **Do not invent** file names, methods, error codes, or behavior. Every claim traces to code.
- Be exhaustive over concise — completeness matters more than brevity here.
- Where the new implementation is genuinely better than legacy, say so and mark it as an intentional improvement rather than a gap.

## PROMPT ENDS HERE

---

## Optional details worth adding before you run it

Adding any of these will noticeably sharpen the output:

- Which cache technology is already available in your stack (Redis? Hazelcast? Caffeine only?)
- Approximate data volumes (how many branches/banks/countries) and read QPS
- How many pods/instances the service runs on
- Whether existing API consumers must keep working unchanged (contract freeze) or can be updated
- Whether reference data updates arrive via API, batch file, or upstream events
- Java/Spring Boot version, and the database in use
