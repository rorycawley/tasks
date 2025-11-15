# Real-World API Scenario

**How a Professional API Spec Handles Production Challenges**

---

## Overview

This document demonstrates how the three core goals of a professional API specification—**Predictability**, **Professionalism**, and **Developer Experience**—work together to handle real-world challenges that occur in production systems.

Through a single scenario involving a user with a flaky network connection, concurrent edits, and a security incident, we'll see how multiple API features coordinate to prevent data loss, crashes, and security breaches.

---

## Scenario: The Subway Task

### Setup

Sarah is riding the subway and using a task management app. She has a task "buy milk" that she wants to mark as complete. However, her phone has a terrible, flaky internet connection. Let's trace what happens as various challenges emerge.

---

## The Timeline

### T=0 seconds: First Tap

Sarah taps "Complete Task" on her phone.

**Her app sends:**
```
PATCH /tasks/task_123
Authorization: Bearer sk_live_xyz789
Idempotency-Key: sarah-complete-milk-2024
{
  "is_done": true
}
```

**What happens:**
- ✓ **API Key** authenticates her (she's legit)
- ✓ **Idempotency Key** is stored: "sarah-complete-milk-2024"
- ✓ Task updated: `is_done: true`, `completed_at: 1731348000`
- ✓ **Timestamp Rule**: `updated_at` changes because value changed
- ✓ Response: **200 OK** with updated task

Her phone receives it... barely. Screen shows ✓ complete.

#### 🎯 GOALS ACHIEVED:

| Goal | How Achieved |
|------|--------------|
| **Professionalism** 🔵 | API Key authentication prevents unauthorized access |
| **Predictability** 🟠 | Idempotency Key stored for future retry detection; Timestamp Rule ensures consistent behavior |

---

### T=2 seconds: Network Hiccup

The connection drops mid-response. Sarah's app didn't receive confirmation. From her perspective: "Did it work??"

App logic decides: "Not sure, better retry..."

**Second request sent (same Idempotency Key):**
```
PATCH /tasks/task_123
Authorization: Bearer sk_live_xyz789
Idempotency-Key: sarah-complete-milk-2024  ← SAME KEY
{
  "is_done": true
}
```

**What happens:**
- ✓ **Idempotency System** recognizes: "I've seen this exact key + endpoint + payload before"
- ✓ Returns the **cached response** from T=0
- ✓ **NO duplicate update** happens
- ✓ `updated_at` stays the same (because nothing actually changed)

**Result:** Safe retry. No duplicate. No confusion.

#### 🎯 GOALS ACHIEVED:

| Goal | How Achieved |
|------|--------------|
| **Predictability** 🟠 | Same request produces identical result; no duplicate task created; timestamps remain consistent |
| **Developer Experience** 🟢 | App developer doesn't need to write complex retry logic; API handles it automatically |

---

### T=5 seconds: Sarah Changes Her Mind

"Wait, I didn't buy milk yet!" Sarah taps "Uncomplete"

**Her app sends (note the different Idempotency Key):**
```
PATCH /tasks/task_123
Authorization: Bearer sk_live_xyz789
Idempotency-Key: sarah-uncomplete-milk-2024  ← DIFFERENT KEY
{
  "is_done": false
}
```

**What happens:**
- ✓ **New Idempotency Key** = new operation (not a retry)
- ✓ Task updated: `is_done: false`, `completed_at: null`
- ✓ **Timestamp Rule**: `completed_at` → null (spec defines this)
- ✓ `updated_at` changes (real change happened)

#### 🎯 GOALS ACHIEVED:

| Goal | How Achieved |
|------|--------------|
| **Predictability** 🟠 | Different key recognized as new operation; `completed_at` null behavior is deterministic per spec |

---

### T=7 seconds: Concurrent Edit

Sarah's friend Tom (on their shared family account) marks the same task complete from his phone at the exact same moment Sarah uncompletes it.

**What happens:**
- ⚠️ **Last-Write-Wins** concurrency model kicks in
- ✓ Server processes both requests
- ✓ Whichever arrives at the database last wins
- ✓ Let's say Tom's wins: task ends up `is_done: true`

**No crash. No error. No data corruption.** Just one wins.

(In a future API version, they might add optimistic locking, but for now: simple, predictable behavior)

#### 🎯 GOALS ACHIEVED:

| Goal | How Achieved |
|------|--------------|
| **Predictability** 🟠 | Last-Write-Wins provides simple, deterministic concurrency behavior; no complex conflicts to handle |
| **Developer Experience** 🟢 | Developers don't need to implement complex merge logic; straightforward model |

---

### T=10 seconds: Security Attack

Meanwhile, a bot found an old `sk_test_` key that was accidentally leaked on GitHub. It tries to spam the API:

**Attack:** 100 requests per second to delete all tasks

**What happens:**
- ✓ **API Key Prefix** check: "This is a TEST key"
- ✓ Only touches test data, not Sarah's live tasks
- ✓ **Rate Limiting**: After 100 requests/minute, returns:

```json
{
  "error": {
    "type": "rate_limit_error",
    "message": "Too many requests. Try again in 60 seconds.",
    "code": "rate_limit_exceeded",
    "doc_url": "https://docs.example.com/errors#rate_limit"
  }
}
```

**HTTP 429 status code**

**Result:** Attack stopped. Sarah never notices. Your server is fine.

#### 🎯 GOALS ACHIEVED:

| Goal | How Achieved |
|------|--------------|
| **Professionalism** 🔵 | API Key Prefixes (`sk_test_` vs `sk_live_`) isolate environments; Rate Limiting prevents abuse |
| **Developer Experience** 🟢 | Structured Error Response with clear message and error code; machine-readable format |

---

## Summary: What Just Happened

In just 10 seconds, the API specification automatically handled:

1. ✓ Flaky network (via idempotency)
2. ✓ User uncertainty (via safe retries)
3. ✓ State changes (via timestamp rules)
4. ✓ Concurrent edits (via last-write-wins)
5. ✓ Security leak (via key prefixes)
6. ✓ Attack (via rate limiting)
7. ✓ Clear errors (via structured responses)

**All automatically. No special code in your app needed.**

---

## The Key Insight

Sarah just wanted to complete a simple task, but the real world threw multiple challenges at her:

- Bad networks
- Retries
- Concurrent users
- Security issues

**The specification anticipated ALL of this in advance.**

This is what "professional-grade" means: handling the messy reality of production systems before problems occur, not after.

---

## How The Three Goals Were Achieved

### 🟠 Predictability

**Definition:** Same request always gets same result; no surprises or weird edge cases.

**Achieved through:**
- Same request always gets same result (idempotency)
- Timestamps change only when data actually changes
- `completed_at` behavior is deterministic (null when uncompleted)
- Last-Write-Wins provides simple concurrency model

**Impact:** Developers can sleep at night knowing the API won't randomly break. No weeks spent debugging race conditions.

---

### 🔵 Professionalism

**Definition:** Handles all the "boring but important" stuff (security, errors, rate limits).

**Achieved through:**
- API Key authentication prevents unauthorized access
- Key prefixes (`sk_test_` vs `sk_live_`) isolate environments
- Rate limiting stops spam and attacks

**Impact:** One security incident or outage can kill a startup. Professional APIs prevent this automatically.

---

### 🟢 Developer Experience

**Definition:** Clear, consistent patterns; good error messages that actually help.

**Achieved through:**
- Developers don't need complex retry logic
- No complex concurrency/merge code needed
- Structured errors with clear messages and codes

**Impact:** Bad DX means developers avoid your API. Good DX means they recommend it to others. It's literally a competitive advantage.

---

## Complete Goal Achievement Matrix

| Timestamp | Event | Predictability 🟠 | Professionalism 🔵 | Developer Experience 🟢 |
|-----------|-------|-------------------|---------------------|-------------------------|
| T=0 | First tap | Idempotency key stored; timestamp rules applied | API key authentication | — |
| T=2 | Network hiccup retry | Same request = same result; no duplicates | — | No complex retry logic needed |
| T=5 | Mind change | Different key = new operation; deterministic null behavior | — | — |
| T=7 | Concurrent edit | Last-write-wins model | — | No merge logic needed |
| T=10 | Security attack | — | Key prefixes isolate environments; rate limiting | Structured error response |

---

## Conclusion

This scenario demonstrates that professional API design isn't about adding features—it's about **anticipating reality**. 

Every feature in the spec exists because without it, something breaks in the real world:
- Without idempotency → duplicate data
- Without rate limiting → server crashes
- Without key prefixes → security breaches
- Without structured errors → developers waste hours debugging

The three goals work together to create an API that **just works** in production, under stress, with real users and real problems.
