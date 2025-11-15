Here is the full API specification formatted in Markdown for your `README.md` file.

You can copy and paste the entire block below directly into your file.

````markdown
# Task API — Full Specification (Stripe-Style)

## Overview

A predictable, versioned, resource-oriented API for creating, listing, updating, completing, and deleting tasks.

Follows Stripe-like conventions:
* Opaque prefixed IDs
* Object envelopes (`object: "task"` or `"list"`)
* Cursor-based pagination
* UTF-8 support
* Unix timestamps
* Structured error objects
* API versioning
* Strong idempotency guarantees
* Deterministic sorting
* Clear concurrency behavior

---

## 1. Authentication

All requests must authenticate with an API key:

```http
Authorization: Bearer sk_test_XXXXXXXXXXXX
````

### API Key Rules

  * **Prefixes:**
      * `sk_test_` (test mode)
      * `sk_live_` (live mode)
  * Remaining characters are opaque and must not be parsed.
  * Keys are ASCII and URL-safe.

### API Key Security

  * API keys are secrets and must never appear in client-side code or public repositories.
  * Exposed keys should be revoked and rotated immediately.
  * Missing or invalid API keys return `401 Unauthorized`.

-----

## 2\. API Versioning

Clients may specify a version via the `Task-Version` header:

```http
Task-Version: 2025-01-01
```

### Versioning Rules

  * Breaking changes only occur across versions.
  * If no version is supplied, the server applies the account’s default version.
  * Responses include the applied version in the `Task-Version` header:
    ```http
    Task-Version: <applied-version>
    ```

-----

## 3\. Idempotency

All non-GET endpoints accept an `Idempotency-Key` header:

```http
Idempotency-Key: <key>
```

### Idempotency Rules

  * Keys are stored for 24 hours per (`key` + `method` + `path` + `API key`).
  * Same key + same parameters → identical response.
  * Same key + different parameters → `409 Conflict`.
  * Prevents accidental duplicates on retries.

-----

## 4\. Resources & Data Model

### 4.1 Task Object

```json
{
  "id": "task_123",
  "object": "task",
  "task_name": "buy milk",
  "is_done": false,
  "completed_at": null,
  "created_at": 1731345000,
  "updated_at": 1731345000
}
```

### Field Definitions

| Field | Type | Description |
| :--- | :--- | :--- |
| `id` | string | Unique opaque ID, prefixed with `task_`. |
| `object` | string | Always `"task"`. |
| `task_name` | string | UTF-8 string; description of the task. |
| `is_done` | boolean | Completion flag. |
| `completed_at` | integer or null | When task was marked complete. |
| `created_at` | integer | Unix timestamp of creation. |
| `updated_at` | integer | Unix timestamp of last change. |

-----

## 5\. Timestamp Semantics

  * All timestamps are Unix timestamps (seconds).
  * No ISO timestamps.
  * `updated_at` changes only when a field value actually changes.
  * No-op updates do not change `updated_at`.

### `completed_at` Behavior

  * `false` → `true` → set to current timestamp
  * `true` → `false` → set to `null`
  * `true` → `true` → unchanged
  * `false` → `false` → unchanged

-----

## 6\. ID Generation

  * IDs are opaque, prefixed with `task_`.
  * Not time-sortable.
  * Cursor-based pagination requires:
    1.  Lookup of the cursor task
    2.  Extracting its `created_at` and `id`
    3.  Using those for pagination boundaries

-----

## 7\. List Behavior

### 7.1 Sorting

All task lists are sorted by:

1.  `created_at` **descending**
2.  `id` **descending**

This defines the canonical order for cursor pagination.

### 7.2 Pagination Parameters

| Param | Type | Description |
| :--- | :--- | :--- |
| `limit` | int | 1–100, default 10. |
| `starting_after`| string | Return tasks after this ID. |
| `ending_before` | string | Return tasks before this ID. |

### Exclusivity

  * `starting_after` and `ending_before` cannot be used together.
  * Returns `400 invalid_request_error`, `invalid_pagination`.

### 7.3 Filtering

| Filter | Type | Description |
| :--- | :--- | :--- |
| `is_done` | boolean | Filter completed/uncompleted tasks. |
| `created_at[gte]` | integer | Minimum creation time. |
| `created_at[lte]` | integer | Maximum creation time. |
| `query` | string | Case-insensitive substring match. Max 512 characters. |

**Date Range Rule:**

  * If `created_at[gte]` \> `created_at[lte]` → `400 invalid_request_error`.

### 7.4 Cursor Validation

A cursor ID must:

  * Start with `task_`.
  * Refer to an existing task.
  * Be part of the result set defined by filters.

**Cursor Error Conditions:**

  * Malformed ID → `400 invalid_cursor`
  * Well-formed but missing → `400 resource_missing`
  * Exists but excluded by filters → `400 invalid_cursor`

**Cursor Stability:**

  * If a task used as a cursor is deleted between paginated requests, the cursor becomes invalid and the API returns `400 invalid_request_error`.
  * Clients encountering cursor errors should restart pagination from the beginning or from the last successfully retrieved page.

### 7.5 Search Behavior

  * `query` performs a case-insensitive substring match.
  * Matches anywhere inside `task_name`.

### 7.6 List Response Format

```json
{
  "object": "list",
  "data": [ ... ],
  "has_more": false
}
```

**Empty Results:**
Filtering that matches no tasks returns an empty list, never a `404`.

```json
{
  "object": "list",
  "data": [],
  "has_more": false
}
```

-----

## 8\. Validation Rules

### `task_name`

  * Required for `POST /tasks`.
  * Optional for `PATCH`.
  * UTF-8 supported; emoji allowed.
  * No control characters (newlines, tabs, ASCII control codes).
  * Trimmed value must be non-empty.
  * Maximum 255 characters.

### `query`

  * UTF-8 string.
  * Maximum 512 characters.

### `limit`

  * Integer between 1 and 100.
  * Non-integer or out-of-range → `400 invalid_request_error`.

### Unknown Parameters

  * Unrecognized fields → `400 invalid_request_error`.

-----

## 9\. Concurrency Model

  * **Last-write-wins (Stripe-style)**
  * Concurrent `PATCH` requests are processed independently.
  * The request processed last determines final state.
  * No optimistic locking or version checks.
  * May be expanded in future API versions.

-----

## 10\. Rate Limiting

  * Default: 100 requests per minute per API key.
  * Test (`sk_test_`) and live (`sk_live_`) keys have separate rate limits.
  * Exceeding limit → `429 rate_limit_error`.
  * Optional headers:
      * `X-RateLimit-Limit`
      * `X-RateLimit-Remaining`
      * `X-RateLimit-Reset`

-----

## 11\. Error Object

All error responses have a structured body:

```json
{
  "error": {
    "type": "invalid_request_error",
    "message": "Missing required param: task_name.",
    "code": "missing_param",
    "param": "task_name",
    "doc_url": "[https://docs.example.com/errors#missing_param](https://docs.example.com/errors#missing_param)"
  }
}
```

### Error Types

  * `invalid_request_error`
  * `authentication_error`
  * `permission_error`
  * `rate_limit_error`
  * `api_error`
  * `external_dependency_error`

### Key Error Codes

  * `missing_param`
  * `invalid_param`
  * `resource_missing`
  * `invalid_cursor`
  * `invalid_pagination`

-----

## 12\. HTTP Status Codes

| Code | Meaning |
| :--- | :--- |
| 200 | OK |
| 201 | Created |
| 400 | Invalid parameters |
| 401 | Authentication failed |
| 402 | Logical failure |
| 403 | Permission error |
| 404 | Resource missing |
| 409 | Idempotency conflict |
| 424 | External dependency failure |
| 429 | Rate limit exceeded |
| 500/502/503/504 | Server errors |

-----

## 13\. Endpoints

### 13.1 Create a Task

**`POST /tasks`**

**Request:**

```http
POST /tasks
Authorization: Bearer sk_test_xxx
Task-Version: 2025-01-01
Idempotency-Key: abc-123
Content-Type: application/json
```

**Payload:**

```json
{
  "task_name": "buy milk",
  "is_done": false
}
```

**Response: (`201 Created`)**

```json
{
  "id": "task_123",
  "object": "task",
  "task_name": "buy milk",
  "is_done": false,
  "completed_at": null,
  "created_at": 1731345000,
  "updated_at": 1731345000
}
```

### 13.2 List Tasks

**`GET /tasks`**

**Example request:**

```http
GET /tasks?limit=2&is_done=false&query=milk&starting_after=task_120
Authorization: Bearer sk_test_xxx
```

**Example response: (`200 OK`)**

```json
{
  "object": "list",
  "data": [
    {
      "id": "task_125",
      "object": "task",
      "task_name": "buy milk 🥛",
      "is_done": false,
      "completed_at": null,
      "created_at": 1731347000,
      "updated_at": 1731347000
    },
    {
      "id": "task_124",
      "object": "task",
      "task_name": "buy oat milk",
      "is_done": false,
      "completed_at": null,
      "created_at": 1731346500,
      "updated_at": 1731346500
    }
  ],
  "has_more": true
}
```

### 13.3 Retrieve a Task

**`GET /tasks/{id}`**

**Response: (`200 OK`)**

```json
{
  "id": "task_123",
  "object": "task",
  "task_name": "buy milk",
  "is_done": false,
  "completed_at": null,
  "created_at": 1731345000,
  "updated_at": 1731345000
}
```

### 13.4 Update a Task

**`PATCH /tasks/{id}`**

**Example payload:**

```json
{
  "task_name": "buy almond milk",
  "is_done": true
}
```

**Response: (`200 OK`)**

```json
{
  "id": "task_123",
  "object": "task",
  "task_name": "buy almond milk",
  "is_done": true,
  "completed_at": 1731348000,
  "created_at": 1731345000,
  "updated_at": 1731348000
}
```

### 13.5 Delete a Task

**`DELETE /tasks/{id}`**

**Response: (`200 OK`)**

```json
{
  "id": "task_123",
  "object": "task",
  "deleted": true
}
```

After deletion, `GET /tasks/{id}` → `404 resource_missing`.

```
```
