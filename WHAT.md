# Task API: How It Achieves Its Goals

## Executive Summary

This document explains the **mechanisms** the Task API specification uses to achieve three core goals: Predictability, Professionalism, and Developer Experience. Each mechanism is explained with concrete examples showing why it matters and how it works.

---

## The Three Core Goals (Recap)

1. **Predictability** - Same request always gets same result, no surprises
2. **Professionalism** - Handles security, errors, rate limits, and edge cases
3. **Developer Experience** - Clear patterns, helpful errors, easy to integrate

---

## The Complete Toolbox

### Mechanisms for Predictability

#### 1. Idempotency Keys

**What it is:**
A unique identifier sent with requests to ensure the same operation isn't performed twice.

**How it works:**
```
POST /tasks
Idempotency-Key: abc-123
{"task_name": "buy milk"}

→ Creates task_001

(Network glitch, user retries)

POST /tasks  
Idempotency-Key: abc-123
{"task_name": "buy milk"}

→ Returns task_001 (no duplicate created)
```

**Why it matters:**
- Network issues happen constantly in mobile apps
- Users click buttons multiple times when things are slow
- Without this: duplicate tasks, duplicate charges, data corruption
- With this: Safe to retry any request

**The spec's rules:**
- Keys stored for 24 hours
- Same key + same params = same response (safe)
- Same key + different params = 409 Conflict (catches bugs)
- Scoped per API key (test and live don't interfere)

**Real-world impact:**
Prevents the "I clicked checkout once but got charged three times" scenario that plagues poorly designed systems.

---

#### 2. Deterministic Sorting

**What it is:**
Task lists always come back in exactly the same order: newest first, with ties broken by ID.

**How it works:**
```
Sort order: 
1. created_at descending (newest first)
2. id descending (if timestamps match)

Example tasks:
- task_125 created at 1731347000
- task_124 created at 1731347000  
- task_123 created at 1731346000

Result order: task_125, task_124, task_123
```

**Why it matters:**
Without deterministic sorting, pagination breaks:
- Page 1: [A, B, C]
- (New task D is created)
- Page 2: [C, E, F] ← C appears twice!

With deterministic sorting:
- Order is stable and predictable
- Pagination works correctly
- Users don't see duplicates or miss items

**The spec's rules:**
- Primary sort: created_at descending
- Secondary sort: id descending (tie-breaker)
- This order applies to ALL list operations
- Cannot be changed (would break pagination)

---

#### 3. Clear Timestamp Semantics

**What it is:**
Explicit rules about when timestamps change and what they mean.

**How it works:**

`updated_at` rules:
- Changes ONLY when field values actually change
- No-op updates don't change it
- Lets clients detect actual changes

`completed_at` rules:
```
false → true:  set to current timestamp
true → false:  set to null
true → true:   unchanged
false → false: unchanged (stays null)
```

**Why it matters:**

Without clear rules:
```
Developer: "Did the task change since I last checked?"
API: "updated_at changed"
Developer: "What changed?"
API: "Uh... nothing actually changed, we just touched it"
```

With clear rules:
```
Developer: "If updated_at changed, I know data changed"
API: "Correct. You can cache reliably."
```

**Real-world impact:**
- Enables efficient caching strategies
- Reduces unnecessary UI updates
- Makes synchronization logic simple and correct

---

#### 4. Last-Write-Wins Concurrency

**What it is:**
When two requests try to update the same task simultaneously, the last one processed wins.

**How it works:**
```
Time: 10:00:00.000 - Request A: {task_name: "buy milk"}
Time: 10:00:00.001 - Request B: {task_name: "buy bread"}

Result: task_name = "buy bread" (B was last)
```

**Why it matters:**

Alternative (optimistic locking):
```
Client must send: {version: 5, task_name: "new"}
If version mismatch → Error, client must retry
```
More complex but prevents overwrites.

Last-write-wins:
```
Client sends: {task_name: "new"}
Always succeeds
Simpler but last write wins
```

**The spec's choice:**
- Started with last-write-wins (Stripe does this)
- Simpler for v1 of API
- Note says "May be expanded in future versions"
- Trade-off: simplicity now vs. safety later

**Real-world impact:**
For a task API, losing a concurrent edit is acceptable. For a payment API, you'd need optimistic locking. The spec acknowledges this trade-off explicitly.

---

### Mechanisms for Professionalism

#### 5. API Key Prefixes

**What it is:**
API keys have meaningful prefixes that identify their type and environment.

**How it works:**
```
sk_test_4eC39HqLyjWDarjtT1zdp7dc  ← Test mode key
sk_live_4eC39HqLyjWDarjtT1zdp7dc  ← Live mode key

sk = secret key
test/live = environment
rest = opaque random data
```

**Why it matters:**

Scenario without prefixes:
```
Developer copies key from production config
Pastes into test environment
Accidentally deletes real customer data
😱
```

Scenario with prefixes:
```
Test environment sees sk_live_xxx
Refuses to start: "Live key detected in test environment"
Developer catches mistake immediately
✓
```

**The spec's rules:**
- Keys are opaque (don't parse the random part)
- Test and live have separate rate limits
- Test keys can't access live data (enforced server-side)

**Real-world impact:**
- Prevents the #1 cause of production incidents (wrong environment)
- Enables safe testing without fear
- Makes leaked keys easier to identify and revoke

---

#### 6. Rate Limiting

**What it is:**
Maximum number of requests allowed per time period.

**How it works:**
```
Default: 100 requests per minute per API key

Request 1-100: ✓ (200 OK)
Request 101: ✗ (429 rate_limit_error)

Response headers:
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1731345060 (when limit resets)
```

**Why it matters:**

Without rate limiting:
- Buggy client in infinite loop → 1 million requests/second
- Your server crashes
- All users affected
- Massive AWS bill

With rate limiting:
- Buggy client hits limit
- Gets clear error message
- Other clients unaffected
- Your server stays up

**The spec's rules:**
- Per API key (not per IP)
- Separate limits for test vs live keys
- Returns structured error with retry timing
- Configurable per account (implied)

**Real-world impact:**
Essential for production stability. Also protects against both accidents and attacks.

---

#### 7. API Versioning

**What it is:**
Clients can specify which version of the API they're using.

**How it works:**
```
Request:
Task-Version: 2025-01-01

Response:
Task-Version: 2025-01-01
(confirms which version was used)
```

**Why it matters:**

Without versioning:
```
Year 1: API returns {is_done: boolean}
Year 2: You want to change to {status: "pending"|"done"|"archived"}
Problem: Breaks all existing clients
Solution: Can't make the change 😞
```

With versioning:
```
2025-01-01: {is_done: boolean}
2026-01-01: {status: "pending"|"done"|"archived"}

Old clients: Send Task-Version: 2025-01-01 → get old format
New clients: Send Task-Version: 2026-01-01 → get new format

Everyone happy ✓
```

**The spec's rules:**
- Version specified in header (not URL)
- Breaking changes require new version
- If no version sent, use account's default
- Response echoes version used

**Real-world impact:**
Lets you improve the API without breaking existing integrations. Critical for long-term API health.

---

#### 8. Structured Error Objects

**What it is:**
Errors return machine-readable JSON with specific fields for debugging.

**How it works:**
```json
{
  "error": {
    "type": "invalid_request_error",
    "message": "Missing required param: task_name.",
    "code": "missing_param",
    "param": "task_name",
    "doc_url": "https://docs.example.com/errors#missing_param"
  }
}
```

**Why it matters:**

Bad error (many APIs):
```
Status: 400
Body: "Bad Request"
```
Developer thinks: "WHAT is bad? WHERE is the problem?"

Good error (this spec):
```
Status: 400
Body: {
  type: "invalid_request_error",  ← Category
  code: "missing_param",           ← Machine-readable
  param: "task_name",              ← Exact field
  message: "Missing required...",  ← Human-readable
  doc_url: "..."                   ← Learn more
}
```

**The spec's rules:**
- Every error has consistent structure
- `type` for broad category
- `code` for specific error (for programmatic handling)
- `param` identifies problematic field
- `message` explains in plain English
- `doc_url` links to documentation

**Real-world impact:**
- Developers fix bugs 10x faster
- Fewer support tickets
- Better monitoring (can alert on specific error codes)
- Clients can handle errors programmatically

---

### Mechanisms for Developer Experience

#### 9. Object Envelopes

**What it is:**
Every response wraps data in a consistent object structure with metadata.

**How it works:**

Single task:
```json
{
  "id": "task_123",
  "object": "task",
  "task_name": "buy milk",
  ...
}
```

List of tasks:
```json
{
  "object": "list",
  "data": [ /* array of tasks */ ],
  "has_more": true
}
```

**Why it matters:**

Without envelopes:
```
GET /tasks/123 → {"task_name": "milk", ...}
GET /tasks → [{"task_name": "milk"}, ...]

Problem: Different shapes!
Client code: if (Array.isArray(response)) { ... } else { ... }
```

With envelopes:
```
Every response has "object" field
Client code: 
  if (response.object === "list") { use response.data }
  if (response.object === "task") { use response }
  
Consistent and predictable!
```

**The spec's rules:**
- Every object has an `object` field
- Value indicates type: "task", "list", etc.
- Lists always have: `object: "list"`, `data: []`, `has_more: bool`
- Makes polymorphic handling easy

**Real-world impact:**
- Client code is simpler and less buggy
- Easy to add new object types later
- Follows Stripe's proven pattern
- Makes auto-generated client libraries easier

---

#### 10. Cursor-Based Pagination

**What it is:**
Navigate through large result sets using opaque cursors instead of page numbers.

**How it works:**

Offset pagination (old way):
```
GET /tasks?page=1&per_page=10  → items 1-10
GET /tasks?page=2&per_page=10  → items 11-20

Problem: If item added between requests, page 2 shows duplicates
```

Cursor pagination (this spec):
```
GET /tasks?limit=10
→ Returns 10 items, last one is task_120

GET /tasks?limit=10&starting_after=task_120
→ Returns next 10 items after task_120
→ Stable, no duplicates even if data changes
```

**Why it matters:**

Real scenario:
- User on page 1 of their tasks
- They create a new task
- They click "next page"
- Offset pagination: might see duplicate or skip items
- Cursor pagination: correctly shows next batch

**The spec's rules:**
- Use `starting_after` OR `ending_before` (not both)
- Cursor must be valid task ID
- Cursor must match current filters
- Returns `has_more` to indicate more pages exist
- Limit: 1-100, default 10

**Cursor validation:**
- Malformed cursor → 400 invalid_cursor
- Cursor task deleted → 400 invalid_cursor
- Cursor excluded by filters → 400 invalid_cursor

**Real-world impact:**
- Handles millions of tasks gracefully
- No duplicate or skipped items during pagination
- Industry best practice for large datasets

---

#### 11. Comprehensive Validation Rules

**What it is:**
Explicit rules for every input, with clear error messages when violated.

**How it works:**

`task_name` validation:
```
✓ "buy milk" 
✓ "buy milk 🥛" (emoji allowed)
✓ "  buy milk  " → trimmed to "buy milk"
✗ "" (empty after trim)
✗ "buy\nmilk" (newlines not allowed)
✗ (256 character string) (too long)
```

`limit` validation:
```
✓ 1, 2, 3, ... 100
✗ 0 → "limit must be between 1 and 100"
✗ 101 → "limit must be between 1 and 100"
✗ "ten" → "limit must be an integer"
```

**Why it matters:**

Without clear validation:
```
Client sends: {"task_name": ""}
Server might: 
- Accept it (now have empty tasks)
- Reject it (with unclear error)
- Crash
Who knows? 🤷
```

With clear validation:
```
Client sends: {"task_name": ""}
Server rejects: {
  "error": {
    "type": "invalid_request_error",
    "code": "invalid_param",
    "param": "task_name",
    "message": "task_name cannot be empty after trimming whitespace"
  }
}
Client fixes immediately ✓
```

**The spec's rules:**
- Every parameter has explicit constraints
- UTF-8 supported (international users)
- No control characters (security)
- Maximum lengths specified
- Trimming behavior defined
- Unknown parameters rejected

**Real-world impact:**
- Prevents garbage data in database
- Makes client-side validation easier
- Catches bugs during development, not production
- Security: prevents injection attacks

---

#### 12. Semantic HTTP Status Codes

**What it is:**
Use HTTP's built-in status codes correctly and consistently.

**How it works:**

```
200 OK          → Successful GET/PATCH
201 Created     → Successful POST (new resource)
400 Bad Request → Client sent invalid data
401 Unauthorized → Missing/invalid API key
404 Not Found   → Resource doesn't exist
409 Conflict    → Idempotency key mismatch
429 Too Many    → Rate limit exceeded
500 Server Error → Something broke on our end
```

**Why it matters:**

Good use:
```
Client checks: if (response.status === 404)
Client knows: Resource missing, stop trying
Client can: Show "task not found" message
```

Bad use (some APIs):
```
Everything returns 200, even errors:
{
  "status": 200,
  "success": false,
  "error": "not found"
}

Now client must parse body to know what happened 😞
```

**The spec's rules:**
- 2xx = success (200, 201)
- 4xx = client error (400, 401, 404, 409, 429)
- 5xx = server error (500, 502, 503, 504)
- 402 = logical failure (unusual, Stripe uses this)
- 424 = external dependency failed

**Real-world impact:**
- Standard HTTP libraries work correctly
- Monitoring tools work out-of-box
- Follows web standards (principle of least surprise)
- Makes debugging easier (status code tells the story)

---

## How Multiple Mechanisms Work Together

### Example: Creating a Task with Network Retry

**Scenario:** User creates a task, network glitches, client retries.

**Mechanisms engaged:**

1. **Authentication (API Keys)**
   ```
   Authorization: Bearer sk_test_xxx
   → Validates user has permission
   ```

2. **Versioning**
   ```
   Task-Version: 2025-01-01
   → Uses correct API behavior
   ```

3. **Idempotency**
   ```
   Idempotency-Key: user-click-789
   → First request: creates task_123
   → Retry: returns same task_123 (no duplicate)
   ```

4. **Validation**
   ```
   {"task_name": "  buy milk  "}
   → Validates: trim, check length, check characters
   → Accepts and trims to "buy milk"
   ```

5. **Rate Limiting**
   ```
   → Checks: Has this key exceeded 100/min?
   → No: proceed
   → Yes: return 429
   ```

6. **Object Envelope**
   ```
   → Response wrapped as:
   {
     "object": "task",
     "id": "task_123",
     ...
   }
   ```

7. **HTTP Status**
   ```
   → First request: 201 Created
   → Retry: 200 OK
   ```

8. **Structured Error** (if something failed)
   ```
   → Clear error object with code, param, message
   ```

**Result:** A simple "create task" request is protected by 8 different mechanisms working in concert.

---

### Example: Paginating Through Tasks

**Scenario:** Developer wants to display all 10,000 tasks in batches.

**Mechanisms engaged:**

1. **Deterministic Sorting**
   ```
   → Results always ordered by: created_at desc, id desc
   → Stable ordering enables reliable pagination
   ```

2. **Cursor Pagination**
   ```
   GET /tasks?limit=100
   → Returns 100 tasks + has_more: true
   
   GET /tasks?limit=100&starting_after=task_500
   → Returns next 100 tasks after task_500
   ```

3. **Object Envelope**
   ```
   {
     "object": "list",
     "data": [ /* 100 tasks */ ],
     "has_more": true
   }
   → Consistent structure every request
   ```

4. **Validation**
   ```
   limit=1000 → Error: "limit must be between 1 and 100"
   → Prevents accidentally requesting millions of tasks
   ```

5. **Cursor Validation**
   ```
   starting_after=task_999 (doesn't exist)
   → Error: "invalid_cursor"
   → Clear feedback on what's wrong
   ```

6. **Rate Limiting**
   ```
   → If client tries to fetch too fast
   → Returns 429 with retry-after timing
   ```

**Result:** Can safely iterate through millions of tasks without duplicates, skips, or crashes.

---

## The Bigger Picture: Why This Level of Detail?

You might ask: "Why specify all this? Seems like overkill for a task API."

**The answer:** This spec isn't really about tasks. It's about:

1. **Preventing known problems**
   - Every rule prevents a real issue that happens in production
   - Stripe learned these lessons over years
   - This spec captures that wisdom upfront

2. **Enabling confident development**
   - Developers know exactly what to expect
   - No ambiguity = fewer bugs
   - Can write code without constantly testing edge cases

3. **Scaling gracefully**
   - These patterns work for 100 users or 100 million
   - Don't need to rewrite when you grow
   - Professional from day one

4. **Being a good platform**
   - If others build on your API, you have responsibility
   - Breaking changes hurt your users
   - Good design = long-term trust

---

## Summary: The Complete Picture

**For Predictability:**
- Idempotency Keys → safe retries
- Deterministic Sorting → stable pagination
- Clear Timestamps → reliable change detection
- Last-Write-Wins → simple concurrency

**For Professionalism:**
- API Key Prefixes → environment safety
- Rate Limiting → stability and security
- Versioning → evolution without breaking
- Structured Errors → debuggable failures

**For Developer Experience:**
- Object Envelopes → consistent responses
- Cursor Pagination → scalable iteration
- Validation Rules → clear feedback
- HTTP Status Codes → standard semantics

**All 12 mechanisms working together create an API that:**
- Works correctly under stress
- Handles edge cases gracefully
- Makes developers productive
- Scales to production loads
- Maintains backwards compatibility
- Provides excellent debugging experience

This is what "enterprise-grade" really means.
