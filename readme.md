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


Hi Team,
As I am joining the GIFT City initiative from the existing Web Banking Payment Transfer team, I would like to understand and clarify the Payment Transfer scope for MVP1.
From the current requirements, I understand that GIFT City digital banking is being enabled on NITRO 7.0, with MVP1 supporting existing IN SCB customers and currencies USD, EUR and GBP. Payment Transfer is one of the key journeys under this scope.
From a Payment Transfer perspective, I would like to clarify the following areas before we proceed with the detailed solution/design.
1. Payment Transfer Scope
Could we please confirm the payment/transfer types planned for GIFT City MVP1?
Own Account Transfer
Book Transfer / SCB-to-SCB transfer
Local/India bank transfer
International transfer
Future-dated transfer
Scheduled/recurring transfer
Beneficiary payments
Cross-currency transfer
It would be helpful to have a clear MVP1 payment journey catalogue, including what is in scope and out of scope.
2. Source and Destination Accounts
We need to understand where a GIFT customer can transfer funds:
GIFT account → Own GIFT account
GIFT account → Own India SCB account
GIFT account → Other SCB account
GIFT account → Indian bank account
GIFT account → International bank account
GIFT account → GIFT account
Also, should transfers between GIFT and India accounts be treated as book transfers, domestic payments or a different payment type?
3. Currency Support
MVP1 mentions USD, EUR and GBP.
Could we clarify whether Payment Transfer supports:
Same-currency transfers — USD → USD, EUR → EUR, GBP → GBP
Cross-currency transfers — USD → EUR, USD → GBP, etc.
GIFT → India transfers involving FX
India → GIFT transfers involving FX
If cross-currency payments are supported, we need to understand where FX rate calculation, rate locking and converted amount calculation will happen.
4. Existing Web Banking Capability Reuse
Since the existing Web Banking Payment Transfer capability already supports several payment journeys, could we clarify:
Which existing Payment Transfer APIs can be reused?
Are we extending the existing Experience/Process APIs for GIFT?
Are GIFT-specific APIs required?
Which existing business validations can be reused?
Which GIFT-specific validations/rules need to be introduced?
A Web Banking vs GIFT Payment Transfer gap analysis would help us identify the required changes.
5. CPH Integration
The project documentation refers to the GIFT City – CPH journey.
Could we clarify whether:
Existing CPH payment services will be reused for GIFT
GIFT-specific CPH services are being introduced
Existing payment contracts need modification
New payment types/currencies are supported by CPH
CPH will provide the same transaction status and error model as existing Web Banking
6. Payment Limits and Eligibility
We need to understand whether GIFT will use the existing Web Banking payment limits or have separate GIFT-specific limits.
Please clarify:
Per-transaction limits
Daily limits
Currency-specific limits
Customer/account-level limits
Payment-type-specific limits
How limits are evaluated for cross-currency transactions
Which system owns the limit validation
7. Authentication / Transaction Authorisation
Could we also confirm whether the existing Web Banking authentication and transaction authorisation framework will be reused?
In particular:
2FA / 3FA requirements
Step-up authentication
Transaction signing
High-value transaction rules
Any GIFT-specific authentication policies
8. Payment Status & Lifecycle
We should confirm whether the existing Web Banking payment lifecycle can be reused.
For example:
Initiated → Authenticated → Submitted → Processing → Completed
and handling of:
Pending / Failed / Rejected / Cancelled / Unknown Status
It would also be important to establish the source of truth for payment status, particularly if CPH processing is asynchronous.
9. Error Handling
Could we align on whether the existing Web Banking Payment Transfer error standard will be reused?
Ideally, downstream/CPH errors should be mapped to a standard Experience API error contract rather than exposing downstream-specific error codes directly to the UI.
We should identify any new GIFT-specific:
Validation errors
Business errors
Compliance/rejection errors
Payment rejection errors
Timeout/downstream errors
Unknown transaction status errors
10. Fees, FX and Payment Confirmation
For each payment type, we also need to understand:
Fee calculation
FX rate and spread
Charges displayed to the customer
Final debit amount
Value date
Payment reference/transaction reference
Confirmation/receipt requirements
Proposed next step
It would be useful to first agree on the Payment Transfer scope and payment/currency/corridor matrix for MVP1.
Once this is confirmed, the Payment Transfer team can perform a Web Banking → GIFT gap analysis covering:
Business Journey → API → Authentication → Validation/Limits → CPH → Payment Status → Error Handling → UI Response
This will help us clearly identify what can be reused from the existing Web Banking implementation and what needs to be changed or newly developed for GIFT City.