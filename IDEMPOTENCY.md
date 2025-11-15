# Idempotency in API Design: A Deep Dive

**A step-by-step guide to understanding how idempotency works under the hood**

---

## Table of Contents

1. [Core Concept](#core-concept)
2. [The Real-World Problem](#the-real-world-problem)
3. [How It Works (Step by Step)](#how-it-works-step-by-step)
4. [Critical Implementation Details](#critical-implementation-details)
5. [Under-the-Hood Implementation](#under-the-hood-implementation)
6. [Edge Cases & Gotchas](#edge-cases--gotchas)
7. [Best Practices](#best-practices)
8. [Comparison to Alternatives](#comparison-to-alternatives)
9. [Quick Reference](#quick-reference)

---

## Core Concept

### Simple Definition

**Idempotency:** "Do the same thing twice, get the same result twice."

Like a light switch - flip it to ON twice, the light is still just... on. Not "double on."

### In API Terms

Making the same API request multiple times has the same effect as making it once.

```
Request 1: Create task "buy milk" → Creates task_123 ✓
Request 2: (exact same)           → Returns task_123 (no duplicate) ✓
Request 3: (exact same)           → Returns task_123 (no duplicate) ✓
```

**Result:** One task, not three.

---

## The Real-World Problem

### The Nightmare Without Idempotency

**Scenario:** User on subway with flaky internet clicks "Create Task"

```
User clicks button
   ↓
App sends request
   ↓
Network times out (did it work?)
   ↓
App retries (just to be safe)
   ↓
Result: ???
```

**Possible outcomes without idempotency:**
- ❌ Two identical tasks created
- ❌ Error thrown (why?)
- ❌ One task created but which request succeeded?
- ❌ Developer has no idea what happened

**Developer's code becomes:**

```javascript
// Nightmare: handling every possible outcome
try {
    await createTask();
} catch (error) {
    // Did it fail? Or succeed but network dropped?
    // Should I retry? Will that create a duplicate?
    // Help???
}
```

### The Solution With Idempotency

**Same scenario with idempotency:**

```
User clicks button
   ↓
App sends: POST /tasks + Idempotency-Key: "abc-123"
   ↓
Network times out
   ↓
App retries: POST /tasks + Idempotency-Key: "abc-123" (SAME KEY)
   ↓
Result: Guaranteed exactly one task ✓
```

**Developer's code becomes:**

```javascript
// Simple: safe to retry
const response = await createTask({
    idempotencyKey: generateUniqueKey()
});
// If it fails, just retry. No duplicates possible.
```

---

## How It Works (Step by Step)

### Step 1: The Request Arrives

Client sends:
```http
POST /tasks
Authorization: Bearer sk_test_xyz
Idempotency-Key: abc-123
Content-Type: application/json

{
  "task_name": "buy milk"
}
```

### Step 2: Server Checks "Have I Seen This Before?"

The server maintains an **idempotency store** (a database table):

```
Idempotency Store:
┌─────────────┬────────┬──────────┬──────────────┬──────────┬────────────┐
│ Key         │ Method │ Path     │ API Key      │ Response │ Expires At │
├─────────────┼────────┼──────────┼──────────────┼──────────┼────────────┤
│ ...         │ ...    │ ...      │ ...          │ ...      │ ...        │
└─────────────┴────────┴──────────┴──────────────┴──────────┴────────────┘
```

Server searches for a matching row where ALL of these match:
- ✓ Key: `abc-123`
- ✓ Method: `POST`
- ✓ Path: `/tasks`
- ✓ API Key: `sk_test_xyz`

### Step 3A: First Time (NOT FOUND)

**No match found** → This is a new operation.

**Server actions:**

1. **Process the request**
   ```
   Create new task → task_123
   ```

2. **Generate response**
   ```json
   {
     "id": "task_123",
     "object": "task",
     "task_name": "buy milk",
     "is_done": false,
     "completed_at": null,
     "created_at": 1731348000,
     "updated_at": 1731348000
   }
   ```

3. **Store in idempotency table**
   ```
   INSERT INTO idempotency_keys:
   ┌─────────────┬────────┬──────────┬──────────────┬─────────────────┬─────────────────────┐
   │ abc-123     │ POST   │ /tasks   │ sk_test_xyz  │ {task_123 ...}  │ 2024-11-16 12:00:00 │
   └─────────────┴────────┴──────────┴──────────────┴─────────────────┴─────────────────────┘
   ```

4. **Return response to client**
   ```
   HTTP 201 Created
   {task_123 object}
   ```

**Result:** Task created. Response cached for 24 hours.

---

### Step 3B: Retry (FOUND)

**Same request arrives again:**

```http
POST /tasks
Authorization: Bearer sk_test_xyz
Idempotency-Key: abc-123  ← SAME KEY
Content-Type: application/json

{
  "task_name": "buy milk"
}
```

**Server searches idempotency store** → **MATCH FOUND!**

**Server actions:**

1. ~~Create task~~ ← **SKIPPED** (already done)
2. **Retrieve cached response** from idempotency table
3. **Return cached response**
   ```
   HTTP 200 OK  ← Note: 200, not 201 (indicates cached)
   {task_123 object}  ← EXACT same response
   ```

**Critical insight:** 
- No database write happens
- No duplicate task created
- Client gets identical response
- It's as if the first request "replayed"

---

## Critical Implementation Details

### Detail 1: What Makes Requests "The Same"?

The idempotency lookup key is a **composite** of four values:

```
Lookup Key = (Idempotency-Key + HTTP Method + Path + API Key)
```

**Why all four?**

| Component | Why It Matters | Example |
|-----------|----------------|---------|
| **Idempotency-Key** | Client's unique identifier | `abc-123` |
| **HTTP Method** | POST vs PATCH are different operations | `POST` vs `PATCH` |
| **Path** | Different endpoints are different operations | `/tasks` vs `/tasks/123` |
| **API Key** | Different users/accounts are isolated | `sk_test_xxx` vs `sk_live_yyy` |

**Examples of DIFFERENT operations:**

```
Operation A: POST /tasks + "abc-123" + sk_test_xyz
Operation B: POST /tasks + "abc-123" + sk_live_xyz  ← Different API key
Operation C: PATCH /tasks + "abc-123" + sk_test_xyz ← Different method
Operation D: POST /tasks/123 + "abc-123" + sk_test_xyz ← Different path
```

All four are stored and tracked separately.

---

### Detail 2: Parameter Mismatch Detection

**What if the request body changes on retry?**

**Request 1:**
```http
POST /tasks
Idempotency-Key: abc-123

{
  "task_name": "buy milk"
}
```

**Request 2 (same key, DIFFERENT data):**
```http
POST /tasks
Idempotency-Key: abc-123

{
  "task_name": "buy CHEESE"  ← CHANGED
}
```

**Server behavior:**

1. Finds matching idempotency key
2. Compares request body: `"buy milk"` ≠ `"buy CHEESE"`
3. **Detects conflict**

**Response:**
```http
HTTP 409 Conflict

{
  "error": {
    "type": "invalid_request_error",
    "message": "Idempotency key 'abc-123' was previously used with different parameters.",
    "code": "idempotency_conflict",
    "param": "idempotency_key",
    "doc_url": "https://docs.example.com/errors#idempotency_conflict"
  }
}
```

**Why refuse?**

The server cannot determine if this is:
- ✓ A legitimate retry (use cached response), OR
- ✗ A bug in your code (accidentally reusing a key)

So it **fails safe** and forces you to clarify by using a new key.

---

### Detail 3: The 24-Hour Expiration Window

Idempotency records are not permanent - they expire after 24 hours.

**Timeline:**

```
Hour 0:  POST /tasks + "abc-123" 
         → Creates task_123
         → Stores response in cache
         → Expires at Hour 24

Hour 2:  POST /tasks + "abc-123"
         → Cache hit! Returns task_123 ✓

Hour 25: POST /tasks + "abc-123"
         → Cache expired and deleted
         → Treated as NEW request
         → Creates task_456 (different task!)
```

**Why 24 hours?**

| Benefit | Reasoning |
|---------|-----------|
| **Long enough** | Covers any reasonable retry scenario (network issues, crashes, etc.) |
| **Short enough** | Prevents database from growing infinitely |
| **Industry standard** | Stripe, Adyen, and other major APIs use 24 hours |

**Practical impact:**

If your client code accidentally reuses the same idempotency key after 24 hours, it will create a duplicate. Your key generation strategy should ensure keys are unique per operation.

---

## Under-the-Hood Implementation

### Database Schema

The idempotency store is typically a database table:

```sql
CREATE TABLE idempotency_keys (
    -- Composite key components
    idempotency_key VARCHAR(255) NOT NULL,
    http_method VARCHAR(10) NOT NULL,
    request_path VARCHAR(255) NOT NULL,
    api_key VARCHAR(255) NOT NULL,
    
    -- Request/response data
    request_body TEXT,
    response_status INT,
    response_body TEXT,
    
    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NOT NULL,
    
    -- Composite primary key
    PRIMARY KEY (idempotency_key, http_method, request_path, api_key),
    
    -- Index for cleanup jobs
    INDEX idx_expires_at (expires_at)
);
```

### Pseudocode Implementation

Here's what happens when a request arrives:

```python
def handle_request(method, path, api_key, idempotency_key, request_body):
    """
    Process an API request with idempotency support.
    """
    
    # Step 1: Check if idempotency key is provided
    if not idempotency_key:
        # No idempotency - process normally
        return process_request(request_body)
    
    # Step 2: Build composite lookup key
    cache_key = {
        'idempotency_key': idempotency_key,
        'http_method': method,
        'request_path': path,
        'api_key': api_key
    }
    
    # Step 3: Look up in idempotency store
    cached_record = idempotency_store.get(cache_key)
    
    # Step 4: Handle cache hit
    if cached_record and not is_expired(cached_record):
        # Check if request body matches
        if cached_record.request_body == request_body:
            # Perfect match - return cached response
            return Response(
                status=cached_record.response_status,
                body=cached_record.response_body
            )
        else:
            # Same key, different parameters - conflict!
            return Response(
                status=409,
                body={
                    "error": {
                        "type": "invalid_request_error",
                        "message": "Idempotency key already used with different parameters.",
                        "code": "idempotency_conflict"
                    }
                }
            )
    
    # Step 5: Cache miss - process request normally
    response = process_request(request_body)
    
    # Step 6: Store response in cache
    idempotency_store.set(
        key=cache_key,
        request_body=request_body,
        response_status=response.status,
        response_body=response.body,
        expires_at=now() + timedelta(hours=24)
    )
    
    # Step 7: Return response
    return response


def is_expired(record):
    """Check if an idempotency record has expired."""
    return record.expires_at < now()
```

### Query Flow

**First request:**
```sql
-- Lookup (returns nothing)
SELECT * FROM idempotency_keys 
WHERE idempotency_key = 'abc-123'
  AND http_method = 'POST'
  AND request_path = '/tasks'
  AND api_key = 'sk_test_xyz'
  AND expires_at > NOW();
-- Result: 0 rows

-- Process request → creates task_123

-- Store result
INSERT INTO idempotency_keys (
    idempotency_key, http_method, request_path, api_key,
    request_body, response_status, response_body, expires_at
) VALUES (
    'abc-123', 'POST', '/tasks', 'sk_test_xyz',
    '{"task_name":"buy milk"}', 201, '{"id":"task_123",...}',
    NOW() + INTERVAL 24 HOUR
);
```

**Retry request:**
```sql
-- Lookup (returns cached record)
SELECT * FROM idempotency_keys 
WHERE idempotency_key = 'abc-123'
  AND http_method = 'POST'
  AND request_path = '/tasks'
  AND api_key = 'sk_test_xyz'
  AND expires_at > NOW();
-- Result: 1 row with cached response

-- No INSERT or UPDATE needed
-- Just return the cached response_body
```

---

## Edge Cases & Gotchas

### Edge Case 1: Server Crashes Mid-Request

**Scenario:**
```
1. Request arrives with idempotency key "abc-123"
2. Server creates task_123
3. Server crashes BEFORE storing in idempotency table
4. Client retries with same key "abc-123"
```

**What happens?**

The idempotency record was never saved, so:
- Retry looks like a new request
- Creates task_456 (duplicate!)
- **Idempotency temporarily broken**

**Solution:**

Use database transactions:
```python
with database.transaction():
    # Create task and store idempotency record ATOMICALLY
    task = create_task(body)
    store_idempotency_record(key, response)
    commit()
```

Either both succeed or both fail. No partial state.

---

### Edge Case 2: Concurrent Identical Requests

**Scenario:**

Two requests with the same idempotency key arrive at the exact same microsecond.

```
Request A: POST /tasks + "abc-123" → arrives at T=0
Request B: POST /tasks + "abc-123" → arrives at T=0
```

**What happens?**

Race condition! Both might:
1. Check idempotency store (both find nothing)
2. Create tasks (task_123 and task_124)
3. Try to insert into idempotency store

**Solution:**

Database constraint enforces uniqueness:
```sql
PRIMARY KEY (idempotency_key, http_method, request_path, api_key)
```

One INSERT succeeds, the other fails with duplicate key error.
The failing request can then retry and find the cached response.

---

### Edge Case 3: Large Response Bodies

**Problem:** Storing large responses (e.g., 5MB JSON) in every idempotency record bloats the database.

**Solutions:**

1. **Compress responses** before storing
   ```python
   compressed = gzip.compress(response_body)
   store(compressed)
   ```

2. **Store reference** instead of full response
   ```python
   # Instead of storing response body, store:
   {
       "task_id": "task_123",
       "reference": "tasks/task_123"
   }
   # On cache hit, fetch fresh data by reference
   ```

3. **Set size limits**
   ```python
   if len(response_body) > MAX_CACHE_SIZE:
       # Don't cache, but mark idempotency key as "used"
       store_used_marker(key)
   ```

---

### Gotcha 1: Client Must Generate Keys

**Critical understanding:** The server doesn't generate idempotency keys. The client does.

**Bad client code:**
```javascript
// ❌ WRONG: Reusing the same key
const IDEMPOTENCY_KEY = "my-constant-key";

fetch('/tasks', {
    headers: { 'Idempotency-Key': IDEMPOTENCY_KEY },
    body: JSON.stringify({ task_name: "buy milk" })
});

fetch('/tasks', {
    headers: { 'Idempotency-Key': IDEMPOTENCY_KEY },  // ❌ Same key!
    body: JSON.stringify({ task_name: "buy bread" })
});
// Second request returns cached "buy milk" task!
```

**Good client code:**
```javascript
// ✓ CORRECT: Generate unique key per operation
function createTask(taskName) {
    const idempotencyKey = `task-create-${uuidv4()}`;
    
    return fetch('/tasks', {
        headers: { 'Idempotency-Key': idempotencyKey },
        body: JSON.stringify({ task_name: taskName })
    });
}

createTask("buy milk");   // Key: task-create-a1b2c3d4...
createTask("buy bread");  // Key: task-create-e5f6g7h8...
```

---

### Gotcha 2: GET Requests Don't Need Idempotency

**GET requests are naturally idempotent** by HTTP spec.

```
GET /tasks/123  (read)
GET /tasks/123  (read again)
GET /tasks/123  (read again)
```

All return the same data. No side effects. No duplicates possible.

**Idempotency keys are only for:**
- POST (create)
- PATCH (update)
- DELETE (remove)
- Any request that modifies state

---

## Best Practices

### For API Implementers

#### 1. Always Use Transactions

```python
# ✓ GOOD: Atomic operation
with transaction():
    result = modify_resource(data)
    store_idempotency(key, result)
    commit()

# ❌ BAD: Can create inconsistency
result = modify_resource(data)
store_idempotency(key, result)  # If this fails, no retry protection!
```

#### 2. Set Appropriate Cache Size Limits

```python
MAX_IDEMPOTENCY_CACHE_SIZE = 1_000_000  # 1MB

if response_size > MAX_IDEMPOTENCY_CACHE_SIZE:
    # Log warning and skip caching
    logger.warning(f"Response too large for idempotency cache: {response_size}")
    # Still mark key as "used" to detect conflicts
```

#### 3. Clean Up Expired Records

```python
# Scheduled job (runs hourly)
def cleanup_expired_idempotency_records():
    deleted = idempotency_store.delete_where(
        expires_at < now()
    )
    logger.info(f"Cleaned up {deleted} expired idempotency records")
```

#### 4. Monitor Idempotency Hit Rate

```python
# Metrics to track
metrics.increment('idempotency.checked')
if cache_hit:
    metrics.increment('idempotency.hit')
    metrics.increment('idempotency.duplicates_prevented')
else:
    metrics.increment('idempotency.miss')

# Good hit rate: 1-5% (indicates retries are working)
# High hit rate: 20%+ (investigate: buggy clients? network issues?)
```

---

### For API Consumers

#### 1. Generate Unique Keys

```javascript
// ✓ GOOD: UUID-based
const idempotencyKey = `${operation}-${uuidv4()}`;

// ✓ GOOD: Timestamp + random
const idempotencyKey = `${operation}-${Date.now()}-${Math.random()}`;

// ✓ GOOD: Hash of operation data
const idempotencyKey = `${operation}-${sha256(JSON.stringify(data))}`;

// ❌ BAD: Constant
const idempotencyKey = "my-key";

// ❌ BAD: Only timestamp (collisions possible)
const idempotencyKey = `${Date.now()}`;
```

#### 2. Retry Logic

```javascript
async function createTaskWithRetry(taskName, maxRetries = 3) {
    const idempotencyKey = `task-create-${uuidv4()}`;
    
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
            const response = await fetch('/tasks', {
                method: 'POST',
                headers: {
                    'Authorization': `Bearer ${API_KEY}`,
                    'Idempotency-Key': idempotencyKey,  // Same key on retry
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({ task_name: taskName })
            });
            
            if (response.ok) {
                return await response.json();
            }
            
            // Don't retry on 4xx errors (except 429)
            if (response.status >= 400 && response.status < 500 && response.status !== 429) {
                throw new Error(`Client error: ${response.status}`);
            }
            
        } catch (error) {
            if (attempt === maxRetries) {
                throw error;
            }
            // Exponential backoff
            await sleep(Math.pow(2, attempt) * 1000);
        }
    }
}
```

#### 3. Store Keys for Debugging

```javascript
// Keep track of idempotency keys for debugging
const operationLog = {
    idempotencyKey: idempotencyKey,
    operation: 'create_task',
    timestamp: Date.now(),
    data: { task_name: "buy milk" }
};

localStorage.setItem(`operation-${idempotencyKey}`, JSON.stringify(operationLog));

// If issues arise, you can trace what happened
```

---

## Comparison to Alternatives

### Why Not Just Use PUT for Idempotency?

**PUT is naturally idempotent:**
```http
PUT /tasks/123
{
  "task_name": "buy milk",
  "is_done": false
}
```

Sending this multiple times always results in the same state.

**But PUT has limitations:**

| Issue | Why It's a Problem |
|-------|-------------------|
| **Requires client-generated IDs** | Client must create `task_123` before sending |
| **No creation semantics** | PUT is "upsert" (update or create), not pure "create" |
| **Conflicts with REST** | In REST, POST creates, PUT replaces |
| **Can't detect retries** | Server can't tell first request from retry |

**POST + Idempotency Keys solves this:**

- Server generates IDs (better control)
- Clear "create" semantics
- Server knows when requests are retries
- Follows REST conventions

---

### Why Not Just Deduplicate by Content?

**Alternative approach:**
"If I receive two identical requests within a time window, ignore the second."

```python
def deduplicate_by_content(request_body):
    hash = sha256(request_body)
    if recent_hashes.contains(hash):
        return cached_response
    # else process...
```

**Problems:**

| Issue | Why It Fails |
|-------|-------------|
| **False positives** | Two users creating "buy milk" → only one succeeds |
| **No client control** | Client can't force a retry if genuinely needed |
| **Hash collisions** | Rare but possible |
| **Short time window** | Either too short (misses retries) or too long (blocks legitimate requests) |

**Idempotency keys avoid all of these** by giving clients explicit control.

---

### Why Not Just Make Everything Idempotent Naturally?

**Some operations are naturally idempotent:**

```
SET user.name = "Alice"    (idempotent)
SET user.name = "Alice"    (same result)
```

**But many operations are not:**

```
CREATE task              (not idempotent)
CREATE task              (creates duplicate!)

INCREMENT counter        (not idempotent)
INCREMENT counter        (different result each time)
```

**Idempotency keys make non-idempotent operations behave idempotently.**

---

## Quick Reference

### Key Concepts

| Concept | Explanation |
|---------|-------------|
| **Idempotency** | Same request → same result (no duplicates) |
| **Idempotency Key** | Client-generated unique identifier |
| **Cache Window** | 24 hours |
| **Composite Key** | (Idempotency-Key + Method + Path + API Key) |

### When to Use

| Operation | Use Idempotency Key? |
|-----------|---------------------|
| POST (create) | ✅ Yes |
| PATCH (update) | ✅ Yes |
| DELETE | ✅ Yes |
| GET (read) | ❌ No (naturally idempotent) |

### HTTP Status Codes

| Status | Meaning |
|--------|---------|
| **201** | Created (first request) |
| **200** | OK (cached response from retry) |
| **409** | Conflict (same key, different parameters) |

### Error Codes

| Code | Meaning |
|------|---------|
| `idempotency_conflict` | Same key used with different parameters |
| `idempotency_key_too_long` | Key exceeds maximum length |

### Client Checklist

- [ ] Generate unique key per operation
- [ ] Use same key on retry
- [ ] Don't reuse keys across different operations
- [ ] Store keys for debugging
- [ ] Implement exponential backoff on retries

### Server Checklist

- [ ] Store idempotency records in database
- [ ] Use composite primary key
- [ ] Implement 24-hour expiration
- [ ] Use transactions for atomicity
- [ ] Clean up expired records
- [ ] Monitor hit rate
- [ ] Return 409 on parameter mismatch

---

## Summary: The Mental Model

Think of idempotency as **short-term memory** for your API:

```
Client: "Hey, can you create a task called 'buy milk'? Here's my reference: abc-123"
Server: "Sure!" *creates task_123* "Done! I'll remember this as 'abc-123' for 24 hours."

Client: "Hey, can you create a task called 'buy milk'? Here's my reference: abc-123"
Server: "I remember! You already asked me this. Here's task_123 again."

Client: "Hey, can you create a task called 'buy CHEESE'? Here's my reference: abc-123"
Server: "Wait, you told me 'milk' for abc-123 before. This is different! Error!"
```

**The power:** Clients can safely retry without fear of duplicates.

**The cost:** Server maintains a cache table for 24 hours.

**The trade-off:** Worth it for production reliability.
