# Task API - Architecture Principles & Technical Decisions

**Document Purpose:** This document maps architectural principles to the technical decisions they drive in the Task API specification.

---

## Principle 1: Simplicity Over Cleverness

**Core Belief:** The simplest solution that solves the problem completely. No fancy algorithms, no "smart" optimizations that add complexity. Boring is good.

### Technical Decisions Driven by This Principle:

1. **Resource-Oriented REST Design**
   - Standard HTTP verbs (POST, GET, PATCH, DELETE)
   - Well-understood patterns
   - No custom protocols or RPC-style endpoints

2. **Last-Write-Wins Concurrency**
   - No locks, no optimistic locking, no version checking
   - Most recent update wins
   - Trade-off: Can lose edits in race conditions, but vastly simpler

3. **Opaque ID Generation**
   - Simple prefixed strings (`task_123`)
   - No complex encoding schemes
   - Server has full flexibility in implementation

4. **UTF-8 Text Encoding**
   - One text encoding standard throughout
   - Modern, widely supported
   - No encoding negotiation needed

---

## Principle 2: Consistency Over Flexibility

**Core Belief:** Same patterns everywhere, even if it means saying "no" to some use cases. One way to do things. Predictable beats powerful.

### Technical Decisions Driven by This Principle:

1. **Single Pagination Model**
   - Cursor-based only (no offset/limit, no page numbers)
   - Same mechanism for all list endpoints
   - `starting_after` / `ending_before` pattern everywhere

2. **Deterministic Sorting**
   - Always `created_at DESC`, then `id DESC`
   - No sorting options, no customization
   - Same query = same order, always

3. **Uniform Concurrency Model**
   - Last-write-wins on ALL endpoints
   - Not different per resource or operation
   - One rule to learn and remember

4. **Structured Error Format**
   - Same JSON shape for all errors
   - Always includes: `type`, `code`, `message`, `param`, `doc_url`
   - Machine and human readable

5. **Unix Timestamps Only**
   - All timestamps are Unix seconds
   - Not ISO strings, not mixed formats
   - One format everywhere

---

## Principle 3: Explicit Over Implicit

**Core Belief:** No magic. No "smart defaults" that surprise you. Everything is stated clearly.

### Technical Decisions Driven by This Principle:

1. **Explicit API Versioning**
   - `Task-Version: 2025-01-01` header required or uses account default
   - Response includes applied version
   - No guessing which behavior you got

2. **Explicit Idempotency Keys**
   - Client must provide `Idempotency-Key` header
   - Not auto-generated from request hash
   - Client controls retry semantics

3. **Timestamp Change Rules Documented**
   - `updated_at` changes only when values change
   - `completed_at` rules explicitly stated (null vs timestamp)
   - No "updated_at changes on every write" surprise

4. **Unknown Parameters Rejected**
   - Typos in field names = 400 error
   - Not silently ignored
   - Forces client to be explicit about what they're sending

5. **Required vs Optional Fields**
   - `task_name` required on POST, optional on PATCH
   - Documented clearly
   - No "we'll guess what you meant"

---

## Principle 4: Design for Failure

**Core Belief:** Assume networks fail, clients retry, users make mistakes. The system must handle all of this gracefully.

### Technical Decisions Driven by This Principle:

1. **Idempotency Support**
   - All non-GET endpoints accept idempotency keys
   - Keys stored for 24 hours
   - Same key + same params = same response
   - Same key + different params = 409 Conflict

2. **Cursor-Based Pagination**
   - Page numbers break when data changes mid-pagination
   - Cursors are stable references to specific records
   - Handles data mutations during pagination

3. **Cursor Validation**
   - Malformed cursors = 400 with specific error code
   - Missing cursors = 400 with clear message
   - Cursors outside filter range = 400
   - Client knows exactly what went wrong

4. **Rate Limiting**
   - 100 requests/minute per API key
   - Protects system from runaway clients
   - 429 response with clear error

5. **Defensive Validation**
   - All inputs validated at boundary
   - `task_name` max 255 chars
   - `query` max 512 chars
   - `limit` must be 1-100 integer
   - No garbage data enters system

6. **Separate Test/Live Environments**
   - `sk_test_` vs `sk_live_` keys
   - Separate rate limits
   - Can't accidentally affect production

---

## Principle 5: Evolvability Without Breakage

**Core Belief:** The API will need to change, but existing clients must keep working. Backwards compatibility is sacred.

### Technical Decisions Driven by This Principle:

1. **API Versioning System**
   - Breaking changes only across versions
   - Old versions continue to work
   - Clients can upgrade on their schedule

2. **Opaque ID Format**
   - Clients can't parse or make assumptions about IDs
   - Server can change ID generation algorithm
   - Clients won't break if we change internal implementation

3. **Separation of Concerns**
   - Authentication, rate limiting, validation are independent
   - Can change rate limits without touching validation
   - Can add new authentication methods without breaking existing

4. **Additive Changes Only Within Version**
   - New optional fields can be added
   - New error codes can be added
   - Clients ignore what they don't understand

---

## Principle 6: Trust Through Predictability

**Core Belief:** Developers trust what they can predict. Same input = same output. Documented behavior = actual behavior.

### Technical Decisions Driven by This Principle:

1. **Deterministic Sorting**
   - Same query always returns same order
   - `created_at DESC`, `id DESC`
   - No random or unpredictable ordering

2. **Idempotency Guarantees**
   - Same idempotency key = identical response
   - Predictable retry behavior
   - No surprises on network failures

3. **Immutable `created_at`**
   - Never changes after creation
   - Reliable for time-based queries
   - Can depend on it for audit trails

4. **No-Op Updates Don't Change `updated_at`**
   - Setting same value = no change
   - `updated_at` only changes when data actually changes
   - Predictable timestamp behavior

5. **Documented `completed_at` Rules**
   - `false → true` = set to current timestamp
   - `true → false` = set to null
   - `true → true` = unchanged
   - `false → false` = unchanged
   - No ambiguity

6. **Empty Results Return List Object**
   - Filter with no matches = `{"object": "list", "data": [], "has_more": false}`
   - Never 404 for empty results
   - Consistent structure

---

## Principle 7: Fail Fast, Fail Clear

**Core Belief:** Bad requests are rejected immediately with useful errors. No silent failures. No "we'll try to figure out what you meant."

### Technical Decisions Driven by This Principle:

1. **Comprehensive Input Validation**
   - Missing `task_name` on POST = 400 `missing_param`
   - `task_name` with control characters = 400 `invalid_param`
   - `limit` not integer = 400 `invalid_request_error`
   - Invalid cursor format = 400 `invalid_cursor`

2. **Structured Error Responses**
   - Every error has: type, code, message, param, doc_url
   - Machine-readable codes for programmatic handling
   - Human-readable messages for debugging
   - Links to documentation

3. **Unknown Parameters Rejected**
   - Typo in field name = 400 error immediately
   - Not silently ignored
   - Catches client bugs early

4. **Authentication Failures**
   - Missing API key = 401 immediately
   - Invalid API key = 401 immediately
   - No attempt to proceed without auth

5. **Cursor Validation Errors**
   - Malformed cursor = 400 `invalid_cursor`
   - Non-existent cursor = 400 `resource_missing`
   - Cursor outside filter range = 400 `invalid_cursor`
   - Client knows exactly what to fix

6. **Date Range Validation**
   - `created_at[gte] > created_at[lte]` = 400 `invalid_request_error`
   - Invalid ranges caught immediately
   - Clear error message

---

## Principle 8: Stateless Operation

**Core Belief:** Each request stands alone. Server doesn't remember you between requests. No sessions.

### Technical Decisions Driven by This Principle:

1. **API Key in Every Request**
   - `Authorization: Bearer sk_test_xxx` on every call
   - No login session to maintain
   - No session cookies

2. **All Context in Request**
   - Pagination cursor in query params
   - Filters in query params
   - Version in header
   - Request is self-contained

3. **No Server-Side State**
   - Last-write-wins (no version tracking)
   - No locks held between requests
   - Horizontally scalable by design

4. **Idempotency Keys Stored, Not Sessions**
   - Keys stored for 24 hours
   - But each request is still independent
   - No session required

---

## Summary: How Principles Drive Decisions

Each technical decision in the Task API is driven by one or more architectural principles:

- **When we chose cursor-based pagination**, it was driven by: Consistency (one pagination model), Design for Failure (stable during data changes), and Fail Fast (clear cursor errors)

- **When we chose last-write-wins concurrency**, it was driven by: Simplicity (easiest model), Consistency (same everywhere), and Stateless (no locking state)

- **When we chose to reject unknown parameters**, it was driven by: Explicit (no guessing), Fail Fast (catch typos early)

- **When we chose opaque IDs**, it was driven by: Simplicity (no complex schemes), Evolvability (can change implementation), Fail Fast (clients can't make wrong assumptions)

Every decision traces back to these fundamental principles.
