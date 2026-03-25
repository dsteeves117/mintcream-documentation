# SocialDistribution API Documentation

All endpoints are prefixed with `/api/`. Authentication uses Django session cookies (log in via the browser first) or HTTP Basic Auth for remote nodes and alternate clients. Responses are JSON.

## Node-Admin Story Notes

These two node-admin requirements are implemented explicitly:

- RESTful interface for most operations: the API exposes predictable resource URLs under `/api/` and uses standard HTTP methods such as `GET`, `POST`, `PUT`, `PATCH`, and `DELETE` across authors, entries, followers, comments, and likes.
- UI only communicates with this node's web server: HTML responses send a Content Security Policy with `connect-src 'self'` and `form-action 'self'`, so browser-side form submissions and client-side requests stay on this node.

---

## Table of Contents

1. [Authors Collection — `GET /api/authors/` · `POST /api/authors/`](#1-authors-collection)
2. [Author Detail — `GET/PUT/PATCH/DELETE /api/authors/{author_id}/`](#2-author-detail)
3. [Entries List — `GET /api/authors/{author_id}/entries/` · `POST`](#3-entries-list)
4. [Entry Detail — `GET/PUT/PATCH/DELETE /api/authors/{author_id}/entries/{entry_id}/`](#4-entry-detail)
5. [Followers List — `GET /api/authors/{author_id}/followers/`](#5-followers-list)
6. [Follower Relation — `GET/PUT/DELETE /api/authors/{author_id}/followers/{follower_fqid}/`](#6-follower-relation)
7. [Follow Requests — `GET /api/authors/{author_id}/follow_requests/`](#7-follow-requests)
8. [Following API — `GET /api/authors/{author_id}/following/` · `GET/PUT/DELETE /api/authors/{author_id}/following/{followee_fqid}/`](#8-following-api)
9. [Comments API](#9-comments-api)
10. [Commented API](#10-commented-api)
11. [Likes API](#11-likes-api)
12. [Liked API](#12-liked-api)
13. [Field Reference](#13-field-reference)
14. [Pagination](#14-pagination)
15. [Error Responses](#15-error-responses)

---

## 1. Authors Collection

### `GET /api/authors/`

**When to use:** Use this endpoint to discover all active authors registered on this node. An Android client or remote node uses this to build a directory of people to follow.

**Why not:** If you already know the author's UUID, use [Author Detail](#2-author-detail) directly — it is faster and does not require paging through results.

**Authentication:** None required (public endpoint).

**Pagination:** Yes — uses `page` and `size` query parameters. See [Pagination](#14-pagination).

#### Query Parameters

| Parameter | Type    | Default | Description                             |
|-----------|---------|---------|-----------------------------------------|
| `page`    | integer | `1`     | 1-based page number.                    |
| `size`    | integer | `10`    | Number of results per page. Max `100`.  |

#### Response `200 OK`

```json
{
  "type":        "authors",
  "page_number": 1,
  "size":        10,
  "count":       42,
  "src": [
    {
      "type":         "author",
      "id":           "http://localhost:8000/api/authors/5a4e7f22-3b1a-4c9d-8e2f-1d0a3c7b6e5f",
      "host":         "http://localhost:8000/api/",
      "displayName":  "Alice",
      "description":  "Software developer",
      "github":       "https://github.com/alice",
      "profileImage": "https://example.com/alice.jpg",
      "web":          "http://localhost:8000/authors/5a4e7f22-3b1a-4c9d-8e2f-1d0a3c7b6e5f"
    }
  ]
}
```

| Field         | Type    | Example                                            | Purpose                                                    |
|---------------|---------|----------------------------------------------------|------------------------------------------------------------|
| `type`        | string  | `"authors"`                                        | Always `"authors"`. Identifies the object type.            |
| `page_number` | integer | `1`                                                | The current page number returned.                          |
| `size`        | integer | `10`                                               | Number of items per page for this response.                |
| `count`       | integer | `42`                                               | Total number of active authors on this node.               |
| `src`         | array   | `[{...}]`                                          | The author objects for this page. See Author fields below. |

**Author object fields** (same for all author responses):

| Field          | Type   | Example value                                                                      | Purpose                                                      |
|----------------|--------|------------------------------------------------------------------------------------|--------------------------------------------------------------|
| `type`         | string | `"author"`                                                                         | Always `"author"`.                                           |
| `id`           | string | `"http://localhost:8000/api/authors/5a4e7f22..."` | Fully-qualified URL identifier. Stable and globally unique.  |
| `host`         | string | `"http://localhost:8000/api/"`                                                     | Base URL of the node's API. Used by remote nodes to route.   |
| `displayName`  | string | `"Alice"`                                                                          | Human-readable name shown in the UI.                         |
| `description`  | string | `"Software developer"`                                                             | Bio text. May be empty string.                               |
| `github`       | string | `"https://github.com/alice"`                                                       | Author's GitHub profile URL. May be empty string.            |
| `profileImage` | string | `"https://example.com/alice.jpg"`                                                  | URL to avatar image. May be empty string.                    |
| `web`          | string | `"http://localhost:8000/authors/5a4e7f22..."`                                      | Browser-facing profile page URL (not the API URL).           |

#### Examples

**List first page, default size:**
```
GET /api/authors/
```

**List page 3, 5 items per page:**
```
GET /api/authors/?page=3&size=5
```

**Response when no active authors exist:**
```json
{"type":"authors","page_number":1,"size":10,"count":0,"src":[]}
```

---

### `POST /api/authors/`

**When to use:** Create a new author profile. Typically used by admin tools or during the initial provisioning of a remote node account. Does **not** create a Django login user — it creates the Author profile object only.

**Why not:** Normal user signup is done via the browser at `/accounts/signup/`. Only use this API endpoint if you need to create an author programmatically (e.g., bootstrapping test data or federating a remote node's author).

**Authentication:** None required (open). Apply server-side access controls if needed.

**Request body** (`Content-Type: application/json`):

| Field          | Type   | Required | Example                          | Purpose                                          |
|----------------|--------|----------|----------------------------------|--------------------------------------------------|
| `displayName`  | string | ✅ Yes   | `"Bob"`                          | Display name. 1–32 characters, must not be blank.|
| `description`  | string | No       | `"Blogger"`                      | Bio text. Defaults to `""`.                      |
| `github`       | URL    | No       | `"https://github.com/bob"`       | GitHub profile URL. Must be a valid URL.         |
| `profileImage` | URL    | No       | `"https://example.com/bob.jpg"`  | Avatar URL. Must be a valid URL.                 |
| `host`         | URL    | No       | `"http://remotenode.example/"` | Host override for federated authors.             |
| `isActive`     | bool   | No       | `true`                           | Defaults to `true`.                              |

#### Response `201 Created`

Returns the newly created author object (same shape as GET detail).

```json
{
  "type":         "author",
  "id":           "http://localhost:8000/api/authors/9f3c1a2b-...",
  "host":         "http://localhost:8000/api/",
  "displayName":  "Bob",
  "description":  "",
  "github":       "https://github.com/bob",
  "profileImage": "",
  "web":          "http://localhost:8000/authors/9f3c1a2b-..."
}
```

#### Response `400 Bad Request`

```json
{"displayName": ["This field is required."]}
```

#### Examples

**Minimal create:**
```json
POST /api/authors/
{"displayName": "Charlie"}
```

**Full create:**
```json
POST /api/authors/
{
  "displayName":  "Diana",
  "description":  "Artist and coder",
  "github":       "https://github.com/diana",
  "profileImage": "https://example.com/diana.png"
}
```

---

## 2. Author Detail

### `GET /api/authors/{author_id}/`

**When to use:** Fetch a single author's full profile by their UUID. Use this when you already know the author's ID (e.g., from a post's `author.id` field) and need their latest profile data.

**Why not:** If you need multiple authors at once, use the collection endpoint with pagination.

**Authentication:** None required (public).

#### Path Parameters

| Parameter   | Type | Example                                | Purpose               |
|-------------|------|----------------------------------------|-----------------------|
| `author_id` | UUID | `5a4e7f22-3b1a-4c9d-8e2f-1d0a3c7b6e5f` | The author's UUID PK. |

#### Response `200 OK` — see Author object fields in section 1.

#### Response `404 Not Found`

Returned if the UUID does not exist **or** the author has been deactivated (`isActive=False`).

```json
{"detail": "Not found."}
```

#### Examples

```
GET /api/authors/5a4e7f22-3b1a-4c9d-8e2f-1d0a3c7b6e5f/
```

---

### `PUT /api/authors/{author_id}/`

**When to use:** Replace an author's profile fields. All writable fields must be supplied. Intended for admin tooling or profile management UIs.

**Why:** Full replacement ensures the client is always in sync with the server state without guessing which fields are optional.

**Authentication:** Required (session or Basic Auth). The request is accepted from any authenticated user — access control should be added server-side for production.

**Request body** — same fields as POST, `displayName` required:

```json
{
  "displayName": "Alice Updated",
  "description": "New bio",
  "github":      "https://github.com/alice-new",
  "profileImage": ""
}
```

#### Response `200 OK` — returns the updated author object.

#### Response `400 Bad Request` — if `displayName` is missing or blank.

---

### `PATCH /api/authors/{author_id}/`

**When to use:** Update one or more fields without supplying the full object. Perfect for the "edit profile" UI where only the changed fields are sent.

**Why not:** If the client always has the full profile loaded, `PUT` is simpler and avoids partial-update edge cases.

**Request body** — any subset of writeable fields:

```json
{"description": "Just updating my bio"}
```

#### Response `200 OK` — returns the full updated author object.

#### Examples

**Update only GitHub:**
```json
PATCH /api/authors/5a4e7f22-.../
{"github": "https://github.com/alice-v2"}
```

**Update display name and bio:**
```json
PATCH /api/authors/5a4e7f22-.../
{"displayName": "Alice V2", "description": "New bio"}
```

---

### `DELETE /api/authors/{author_id}/`

**When to use:** Soft-delete an author. Sets `isActive=False` — does **not** remove the database row or any of their posts.

**Why soft delete:** Preserves referential integrity (entries still reference the author). The author's content remains accessible internally but is hidden from public discovery.

**Authentication:** Required.

**Response `204 No Content`** — no body.

**After deletion:** `GET /api/authors/{id}/` returns `404`.

---

## 3. Entries List

### `GET /api/authors/{author_id}/entries/`

**When to use:** Fetch all entries belonging to an author. Used by Android clients, remote nodes, or the stream aggregator to pull an author's posts. Results include all non-deleted visibility levels — the client is responsible for filtering by `visibility` as appropriate for its context.

**Why not filter by visibility server-side here:** The spec requires the full list to be available to authenticated nodes so they can apply their own friend-of-friend logic. Filtering PUBLIC-only can be done client-side.

**Authentication:** Not required. Unauthenticated requests receive the full list (excluding DELETED entries). Future revisions may restrict FRIENDS entries.

**Pagination:** Not yet paginated — returns all entries for the author in a single list, newest first.

#### Path Parameters

| Parameter   | Type | Purpose               |
|-------------|------|-----------------------|
| `author_id` | UUID | The author's UUID PK. |

#### Response `200 OK`

```json
{
  "type": "entries",
  "items": [
    {
      "type":        "entry",
      "id":          "http://localhost:8000/api/authors/5a4e.../entries/3c1f2a...",
      "web":         "http://localhost:8000/authors/5a4e.../entries/3c1f2a...",
      "uuid":        "3c1f2a7b-...",
      "author":      { "type": "author", "id": "...", "displayName": "Alice", ... },
      "title":       "Hello World",
      "description": "My first post",
      "contentType": "text/plain",
      "content":     "This is the body of my post.",
      "visibility":  "PUBLIC",
      "published":   "2026-02-20T12:00:00.000000Z",
      "updated_at":  "2026-02-20T12:05:00.000000Z"
    }
  ]
}
```

| Field        | Type    | Example                                                  | Purpose                                                                          |
|--------------|---------|----------------------------------------------------------|----------------------------------------------------------------------------------|
| `type`       | string  | `"entries"`                                              | Always `"entries"`. Identifies the collection type.                              |
| `items`      | array   | `[{...}]`                                                | Array of entry objects. Empty array if no entries exist.                         |

**Entry object fields:**

| Field         | Type     | Example                                              | Purpose                                                                       |
|---------------|----------|------------------------------------------------------|-------------------------------------------------------------------------------|
| `type`        | string   | `"entry"`                                            | Always `"entry"`.                                                             |
| `id`          | string   | `"http://.../api/authors/.../entries/..."`           | Fully-qualified URL identifier. Stable. Use this as the canonical reference.  |
| `web`         | string   | `"http://.../authors/.../entries/..."`               | Browser-facing URL for the entry's detail page.                               |
| `uuid`        | string   | `"3c1f2a7b-..."`                                     | The raw UUID of the entry (the last segment of `id`).                         |
| `author`      | object   | see Author fields                                    | Embedded full author object for this entry.                                   |
| `title`       | string   | `"Hello World"`                                      | Title of the post. Required.                                                  |
| `description` | string   | `"My first post"`                                    | Short summary shown in lists. May be empty.                                   |
| `contentType` | string   | `"text/plain"`                                       | MIME type of `content`. See allowed values below.                             |
| `content`     | string   | `"This is the body..."`                              | The post body. For image types, this is a base64-encoded string.              |
| `visibility`  | string   | `"PUBLIC"`                                           | One of: `PUBLIC`, `FRIENDS`, `UNLISTED`. Never `DELETED` in responses.       |
| `published`   | datetime | `"2026-02-20T12:00:00.000000Z"`                      | ISO 8601 UTC timestamp. Set on creation, not modified.                        |
| `updated_at`  | datetime | `"2026-02-20T12:05:00.000000Z"`                      | ISO 8601 UTC timestamp. Updated on each PUT/PATCH.                            |

**Allowed `contentType` values:**

| Value                  | Meaning                                      |
|------------------------|----------------------------------------------|
| `text/plain`           | Plain text.                                  |
| `text/markdown`        | Markdown — rendered by the UI.               |
| `image/png;base64`     | PNG image; `content` is base64-encoded data. |
| `image/jpeg;base64`    | JPEG image; `content` is base64-encoded data.|
| `application/base64`   | Generic binary; `content` is base64-encoded. |

**Visibility values:**

| Value      | Who can see it                                          |
|------------|---------------------------------------------------------|
| `PUBLIC`   | Anyone, including unauthenticated visitors.             |
| `FRIENDS`  | Mutual followers only (both must follow each other).    |
| `UNLISTED` | Anyone with the direct link; not listed in directories. |

#### Examples

```
GET /api/authors/5a4e7f22-.../entries/
```

---

### `POST /api/authors/{author_id}/entries/`

**When to use:** Create a new entry for the specified author. The authenticated user must be the author. Used by mobile apps, bot scripts, or the web UI to publish posts.

**Why not:** Use the browser form at `/authors/{id}/entries/` for standard web posting.

**Authentication:** Required. Must be the author (403 otherwise).

**Request body** (`Content-Type: application/json`):

| Field         | Type   | Required | Example          | Purpose                                               |
|---------------|--------|----------|------------------|-------------------------------------------------------|
| `title`       | string | ✅ Yes   | `"My Post"`      | Post title. Must not be blank.                        |
| `contentType` | string | ✅ Yes   | `"text/plain"`   | MIME type. Must be one of the allowed values.         |
| `content`     | string | No       | `"Hello!"`       | Post body. For base64 types, must be valid base64.    |
| `description` | string | No       | `"A short post"` | Short summary. Defaults to `""`.                      |
| `visibility`  | string | No       | `"PUBLIC"`       | Defaults to `"PUBLIC"`. Cannot be `"DELETED"`.        |

#### Response `201 Created` — returns the created entry object.

#### Response `400 Bad Request`

```json
{"title": ["This field may not be blank."]}
```
```json
{"contentType": ["Unsupported contentType 'text/html'. Allowed: ['application/base64', ...]"]}
```
```json
{"content": ["Content must be valid base64 for this contentType."]}
```

#### Response `403 Forbidden` — if not authenticated or not the author.

#### Examples

**Plain text post:**
```json
POST /api/authors/5a4e.../entries/
{
  "title":       "My First Post",
  "contentType": "text/plain",
  "content":     "Hello, world!",
  "visibility":  "PUBLIC"
}
```

**Markdown post (friends only):**
```json
POST /api/authors/5a4e.../entries/
{
  "title":       "Secret markdown",
  "contentType": "text/markdown",
  "content":     "# Header\n\nThis is **friends only**.",
  "visibility":  "FRIENDS"
}
```

**Image post (PNG base64):**
```json
POST /api/authors/5a4e.../entries/
{
  "title":       "My Photo",
  "contentType": "image/png;base64",
  "content":     "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJ...",
  "visibility":  "PUBLIC"
}
```

---

## 4. Entry Detail

### `GET /api/authors/{author_id}/entries/{entry_id}/`

**When to use:** Fetch a single entry by its UUID. Use when you have a shareable link (the `id` field from any entry object) and need the full entry data.

**Authentication:** None required for PUBLIC and UNLISTED entries. FRIENDS entries require authentication (future work).

#### Path Parameters

| Parameter   | Type | Purpose                                  |
|-------------|------|------------------------------------------|
| `author_id` | UUID | Author who owns the entry.               |
| `entry_id`  | UUID | The entry's UUID (from the `uuid` field). |

#### Response `200 OK` — entry object (same shape as in list).

#### Response `404 Not Found` — if the entry is deleted or does not exist.

#### Examples

```
GET /api/authors/5a4e.../entries/3c1f2a7b-.../
```

---

### `PUT /api/authors/{author_id}/entries/{entry_id}/`

**When to use:** Replace all fields of an existing entry. Used when an edit form submits the complete revised post.

**Authentication:** Required. Must be the author (403 otherwise).

**Request body:** Same as POST but all fields supplied.

#### Response `200 OK` — returns the updated entry object.

#### Examples

```json
PUT /api/authors/5a4e.../entries/3c1f.../
{
  "title":       "Revised Title",
  "contentType": "text/markdown",
  "content":     "Updated **content**.",
  "visibility":  "PUBLIC"
}
```

---

### `PATCH /api/authors/{author_id}/entries/{entry_id}/`

**When to use:** Partially update an entry. Only the fields sent are changed — others are left as-is. Useful for changing just the visibility or just the title.

**Authentication:** Required. Must be the author.

#### Examples

**Make entry friends-only:**
```json
PATCH /api/authors/5a4e.../entries/3c1f.../
{"visibility": "FRIENDS"}
```

**Update title only:**
```json
PATCH /api/authors/5a4e.../entries/3c1f.../
{"title": "Better Title"}
```

---

### `DELETE /api/authors/{author_id}/entries/{entry_id}/`

**When to use:** Soft-delete an entry. Sets `visibility` to `DELETED` — the row remains in the database but is hidden from API responses and the UI.

**Why soft delete:** Preserves data integrity across nodes that may have cached the entry. The entry simply becomes invisible rather than causing broken references.

**Authentication:** Required. Must be the author.

**Response `204 No Content`** — no body.

**After deletion:** `GET` on the same URL returns `404`.

---

## 5. Followers List

### `GET /api/authors/{author_id}/followers/`

**When to use:** Retrieve the list of authors who have been **accepted** as followers of `author_id`. Used by Android clients and remote nodes to check who is following someone, for friend-of-friend visibility calculations.

**Why not:** This endpoint only returns *accepted* follows. To see pending requests, use [Follow Requests](#7-follow-requests).

**Authentication:** Required.

#### Response `200 OK`

```json
{
  "type": "followers",
  "followers": [
    {
      "type":         "author",
      "id":           "http://localhost:8000/api/authors/9f3c...",
      "displayName":  "Bob",
      "host":         "http://localhost:8000/api/",
      "description":  "",
      "github":       "",
      "profileImage": "",
      "web":          "http://localhost:8000/authors/9f3c..."
    }
  ]
}
```

| Field       | Type   | Example       | Purpose                                                          |
|-------------|--------|---------------|------------------------------------------------------------------|
| `type`      | string | `"followers"` | Always `"followers"`. Identifies the response type.             |
| `followers` | array  | `[{...}]`     | Array of author objects for each accepted follower. May be empty.|

#### Examples

```
GET /api/authors/5a4e.../followers/
```

**Empty followers list:**
```json
{"type": "followers", "followers": []}
```

---

## 6. Follower Relation

Path parameter for this section:

| Parameter       | Type   | Example                                                                                                 | Notes |
|----------------|--------|---------------------------------------------------------------------------------------------------------|-------|
| `follower_fqid` | string | `https%3A%2F%2Fnode.example.com%2Fapi%2Fauthors%2F11111111-2222-3333-4444-555555555555` | URL-encoded full author FQID used as a path segment. |

### `GET /api/authors/{author_id}/followers/{follower_fqid}/`

**When to use:** Check whether a specific author (identified by FQID) is an accepted follower of `author_id`.

**Authentication:** Required.

#### Response `200 OK`

Returns the follower author object when the relation exists.

#### Response `404 Not Found`

Returned when the follower relation does not exist or the provided FQID cannot be resolved locally.

#### Example

```
GET /api/authors/5a4e.../followers/https%3A%2F%2Fnode.example.com%2Fapi%2Fauthors%2F11111111-2222-3333-4444-555555555555/
```

---

### `PUT /api/authors/{author_id}/followers/{follower_fqid}/`

**When to use:** Accept a pending follow request from `follower_fqid`. Sets `accepted=True` on the `Follow` relation. This is the moderation endpoint — the owner of `author_id` decides whether to approve an incoming request.

**Why:** Keeps follow-request approval server-authoritative. The requester cannot approve their own request.

**Authentication:** Required. Must be the owner of `author_id` (403 otherwise).

#### Response `200 OK` — the accepted follow relation:

```json
{
  "type":      "follow_request",
  "follower":  { "type": "author", "id": "...", "displayName": "Bob", ... },
  "following": { "type": "author", "id": "...", "displayName": "Alice", ... },
  "summary":   "Bob wants to follow Alice",
  "accepted":  true,
  "created_at": "2026-02-20T10:00:00.000000Z"
}
```

#### Response `200 OK` when no pending request exists:

```json
{
  "type":     "follow_request",
  "detail":   "No pending follow request found.",
  "accepted": true
}
```
(Idempotent — calling PUT on an already-accepted relation is safe.)

#### Response `400 Bad Request` — if `author_id` and `follower_fqid` identify the same author (self-follow).

#### Examples

```
PUT /api/authors/5a4e.../followers/https%3A%2F%2Fnode.example.com%2Fapi%2Fauthors%2F11111111-2222-3333-4444-555555555555/
```

---

### `DELETE /api/authors/{author_id}/followers/{follower_fqid}/`

**When to use:** Remove a follower relation — either denying a pending request or revoking an already-accepted follower. The `Follow` row is deleted from the database.

**Authentication:** Required. Must be the owner of `author_id`.

**Response `204 No Content`** — no body. Idempotent: deleting a non-existent relation also returns 204.

#### Examples

**Deny a pending request:**
```
DELETE /api/authors/5a4e.../followers/https%3A%2F%2Fnode.example.com%2Fapi%2Fauthors%2F11111111-2222-3333-4444-555555555555/
```

**Remove an existing follower:**
```
DELETE /api/authors/5a4e.../followers/https%3A%2F%2Fnode.example.com%2Fapi%2Fauthors%2F11111111-2222-3333-4444-555555555555/
```

---

## 7. Follow Requests

### `GET /api/authors/{author_id}/follow_requests/`

**When to use:** List all pending (unaccepted) follow requests directed at `author_id`. The owner uses this to decide who to approve or deny. Typically polled periodically by an Android client's notification screen.

**Why not:** This endpoint is owner-only. If you need to check whether a specific person follows someone, use `GET /api/authors/{author_id}/followers/{follower_fqid}/` instead.

**Authentication:** Required. Must be the owner of `author_id` (403 if you request another author's queue).

**Pagination:** Not paginated — returns all pending requests.

#### Response `200 OK`

```json
{
  "type": "follow_requests",
  "items": [
    {
      "type":      "follow_request",
      "follower":  { "type": "author", "id": "...", "displayName": "Charlie", ... },
      "following": { "type": "author", "id": "...", "displayName": "Alice", ... },
      "summary":   "Charlie wants to follow Alice",
      "accepted":  false,
      "created_at": "2026-02-20T09:00:00.000000Z"
    }
  ]
}
```

| Field  | Type   | Example             | Purpose                                                      |
|--------|--------|---------------------|--------------------------------------------------------------|
| `type` | string | `"follow_requests"` | Always `"follow_requests"`. Identifies the response type.    |
| `items`| array  | `[{...}]`           | Array of follow-request objects. Empty if no pending requests.|

**Follow-request item fields:**

| Field        | Type     | Example                         | Purpose                                                     |
|--------------|----------|---------------------------------|-------------------------------------------------------------|
| `type`       | string   | `"follow_request"`              | Always `"follow_request"`.                                  |
| `follower`   | object   | Author object                   | The author who wants to follow.                             |
| `following`  | object   | Author object                   | The author being requested to be followed (always `author_id`).|
| `summary`    | string   | `"Bob wants to follow Alice"`   | Human-readable label for the request. May be empty.         |
| `accepted`   | boolean  | `false`                         | Always `false` in this list (these are pending only).       |
| `created_at` | datetime | `"2026-02-20T09:00:00.000000Z"` | When the follow request was created.                        |

#### Examples

**Logged in as Alice, check Alice's queue:**
```
GET /api/authors/5a4e.../follow_requests/
```

**Approve the first request:**
```
PUT /api/authors/5a4e.../followers/https%3A%2F%2Fnode.example.com%2Fapi%2Fauthors%2F11111111-2222-3333-4444-555555555555/
```

**Deny the first request:**
```
DELETE /api/authors/5a4e.../followers/https%3A%2F%2Fnode.example.com%2Fapi%2Fauthors%2F11111111-2222-3333-4444-555555555555/
```

---

## 8. Following API

Path parameter for relation endpoints in this section:

| Parameter        | Type   | Example                                                                                                 | Notes |
|-----------------|--------|---------------------------------------------------------------------------------------------------------|-------|
| `followee_fqid` | string | `https%3A%2F%2Fnode.example.com%2Fapi%2Fauthors%2F11111111-2222-3333-4444-555555555555` | URL-encoded full author FQID used as a path segment. |

### `GET /api/authors/{author_id}/following/`

**When to use:** Retrieve accepted outgoing follows for `author_id`.

**Authentication:** Required.

#### Response `200 OK`

```json
{
  "type": "following",
  "items": [
    {
      "type": "author",
      "id": "http://localhost:8000/api/authors/9f3c...",
      "displayName": "Bob",
      "host": "http://localhost:8000/api/",
      "description": "",
      "github": "",
      "profileImage": "",
      "web": "http://localhost:8000/authors/9f3c..."
    }
  ]
}
```

#### Example

```
GET /api/authors/5a4e.../following/
```

### `GET /api/authors/{author_id}/following/{followee_fqid}/`

**When to use:** Check whether `author_id` is currently following a specific author.

**Authentication:** Required.

#### Response `200 OK`

Returns the followee author object when the accepted outgoing relation exists.

#### Response `404 Not Found`

Returned when the relation does not exist or the followee FQID cannot be resolved locally.

### `PUT /api/authors/{author_id}/following/{followee_fqid}/`

**When to use:** Create or reuse an outgoing follow request from `author_id` to `followee_fqid`.

**Authentication:** Required. Must be the owner of `author_id` (403 otherwise).

#### Response `201 Created`

Returned when a new follow row is created.

#### Response `200 OK`

Returned when the relation already exists (idempotent repeat call).

#### Response `400 Bad Request`

Returned when attempting to follow self.

### `DELETE /api/authors/{author_id}/following/{followee_fqid}/`

**When to use:** Unfollow the target author by deleting the outgoing relation (pending or accepted).

**Authentication:** Required. Must be the owner of `author_id`.

#### Response `204 No Content`

Always idempotent. Missing relation also returns `204`.

#### Examples

```
PUT /api/authors/5a4e.../following/https%3A%2F%2Fnode.example.com%2Fapi%2Fauthors%2F11111111-2222-3333-4444-555555555555/
```

```
DELETE /api/authors/5a4e.../following/https%3A%2F%2Fnode.example.com%2Fapi%2Fauthors%2F11111111-2222-3333-4444-555555555555/
```

**Friendship note:** there is no separate `/friends` endpoint. Friendship is derived from two accepted follows in opposite directions.

---

## 9. Comments API

### `GET /api/authors/{author_id}/entries/{entry_id}/comments/`

**When to use:** Retrieve comments for an entry the requester can access.

**Authentication:** Optional for read, but visibility rules still apply.

#### Response `200 OK`

```json
{
  "type": "comments",
  "id": "http://localhost:8000/api/authors/{author_id}/entries/{entry_id}/comments/",
  "web": "http://localhost:8000/authors/{author_id}/entries/{entry_id}/",
  "page_number": 1,
  "size": 1,
  "count": 1,
  "src": [
    {
      "type": "comment",
      "id": "http://localhost:8000/api/authors/{author_id}/entries/{entry_id}/comments/{comment_id}",
      "author": { "type": "author", "id": "...", "displayName": "Bob", "...": "..." },
      "comment": "Nice post!",
      "contentType": "text/plain",
      "published": "2026-03-01T10:00:00.000000Z",
      "entry": "http://localhost:8000/api/authors/{author_id}/entries/{entry_id}",
      "likes": {
        "type": "likes",
        "id": "http://localhost:8000/api/authors/{author_id}/entries/{entry_id}/comments/{comment_id}/likes/",
        "web": "http://localhost:8000/authors/{author_id}/entries/{entry_id}/",
        "page_number": 1,
        "size": 0,
        "count": 0,
        "src": []
      }
    }
  ]
}
```

### `GET /api/entries/{entry_fqid}/comments/`

**When to use:** Retrieve comments by looking up the entry using its FQID (URL-encoded in the path).

### `GET /api/authors/{author_id}/entries/{entry_id}/comments/{comment_fqid}/`

**When to use:** Retrieve one comment object by comment FQID.

Privacy note for friends-only entries:
- comments are visible to friends who can access the entry,
- and to the comment author for their own comment resource.

### `GET /api/authors/{author_id}/entries/{entry_id}/comments/{comment_fqid}/likes/`

**When to use:** Retrieve likes for a single comment object.

**Authentication:** Optional for read, but friends-only visibility rules still apply.

#### Response `200 OK`

Returns a `likes` collection object with `type/id/web/page_number/size/count/src`.

#### Errors
- `403` when requester cannot view the comment.
- `404` when comment does not exist or does not belong to the entry.

### `POST /api/authors/{author_id}/entries/{entry_id}/comments/{comment_fqid}/like/`

**When to use:** Local convenience endpoint to like a comment (UI-friendly create path).

**Authentication:** Required.

#### Response
- `201 Created` on first like.
- `200 OK` if the same author already liked the same comment (idempotent).

#### Errors
- `403` when requester cannot like/view the target comment.
- `404` when comment does not exist or does not belong to the entry.

---

## 10. Commented API

### `POST /api/authors/{author_id}/commented/`

**When to use:** Create a comment on an entry as the authenticated local author.

**Authentication:** Required.

**Request body**:

```json
{
  "entry": "http://localhost:8000/api/authors/{author_id}/entries/{entry_id}",
  "comment": "Great write-up.",
  "contentType": "text/plain"
}
```

#### Response `201 Created`

Returns the created `comment` object (same shape as the `src` item above).

#### Errors
- `400` when request body is invalid (for example blank comment).
- `403` when requester cannot access the target entry.
- `404` when target entry does not exist.

---

## 11. Likes API

### `GET /api/authors/{author_id}/entries/{entry_id}/likes/`

**When to use:** List likes for an entry the requester can access.

#### Response `200 OK`

```json
{
  "type": "likes",
  "id": "http://localhost:8000/api/authors/{author_id}/entries/{entry_id}/likes/",
  "web": "http://localhost:8000/authors/{author_id}/entries/{entry_id}/",
  "page_number": 1,
  "size": 1,
  "count": 1,
  "src": [
    {
      "type": "like",
      "id": "http://localhost:8000/api/authors/{liker_author_id}/liked/{like_id}",
      "author": { "type": "author", "id": "...", "displayName": "Alice", "...": "..." },
      "object": "http://localhost:8000/api/authors/{author_id}/entries/{entry_id}",
      "summary": "Alice likes your entry",
      "published": "2026-03-01T11:00:00.000000Z"
    }
  ]
}
```

### `GET /api/entries/{entry_fqid}/likes/`

**When to use:** List likes for an entry by entry FQID (URL-encoded in the path).

---

## 12. Liked API

### `GET /api/authors/{author_id}/liked/`

**When to use:** List all likes made by an author.

### `GET /api/authors/{author_id}/liked/{like_id}/`

**When to use:** Retrieve a single like object by local like UUID.

### `GET /api/liked/{like_fqid}/`

**When to use:** Retrieve a single like object by FQID (URL-encoded in the path).

---

## 13. Field Reference

### Author fields (re-used across all endpoints)

| Field          | Type   | Nullable | Notes                                                    |
|----------------|--------|----------|----------------------------------------------------------|
| `type`         | string | No       | Always `"author"`.                                        |
| `id`           | string | No       | Full URL: `{host}authors/{uuid}`. Always stable.          |
| `host`         | string | No       | Base API URL of the node: `{scheme}://{domain}/api/`.     |
| `displayName`  | string | No       | 1–32 characters.                                          |
| `description`  | string | No       | May be empty string `""`.                                 |
| `github`       | string | No       | Valid URL or `""`.                                        |
| `profileImage` | string | No       | Valid URL or `""`.                                        |
| `web`          | string | No       | Browser profile URL. Not the API URL.                     |

### Entry content types

| `contentType`          | `content` format      |
|------------------------|-----------------------|
| `text/plain`           | UTF-8 plain text.     |
| `text/markdown`        | Markdown source text. |
| `image/png;base64`     | Base64-encoded PNG.   |
| `image/jpeg;base64`    | Base64-encoded JPEG.  |
| `application/base64`   | Base64-encoded bytes. |

---

## 14. Pagination

Only `GET /api/authors/` is currently paginated. All other list endpoints return full results.

### Query parameters

| Parameter | Type    | Default | Valid range | Description                    |
|-----------|---------|---------|-------------|--------------------------------|
| `page`    | integer | `1`     | ≥ 1         | 1-based page number.           |
| `size`    | integer | `10`    | 1 – 100     | How many items per page.       |

### Response fields

| Field         | Type    | Description                                          |
|---------------|---------|------------------------------------------------------|
| `page_number` | integer | Current page number (may differ from request if OOB).|
| `size`        | integer | Items per page used for this response.               |
| `count`       | integer | Total matching items across all pages.               |
| `src`         | array   | Items on this page.                                  |

### How to iterate all authors

```
page = 1
loop:
  GET /api/authors/?page={page}&size=50
  if len(src) < size: break
  page += 1
```

### Edge cases

- Requesting a page beyond the last page returns the **last** available page (not 404).
- `size` is clamped to max `100`. Larger values are silently reduced.
- `size=0` is treated as `size=1`.

---

## 15. Error Responses

All errors follow DRF's standard envelope:

```json
{"detail": "Human-readable error message."}
```

Or for field-level validation failures:

```json
{
  "fieldName": ["Error message for this field."],
  "otherField": ["Another error."]
}
```

### Common HTTP status codes

| Code | Meaning                                                              |
|------|----------------------------------------------------------------------|
| 200  | OK — GET/PUT/PATCH succeeded.                                        |
| 201  | Created — POST succeeded.                                            |
| 204  | No Content — DELETE succeeded.                                       |
| 400  | Bad Request — validation failed; see response body for details.      |
| 401  | Unauthorized — authentication credentials were not provided.         |
| 403  | Forbidden — authenticated but not permitted (wrong user, etc.).      |
| 404  | Not Found — UUID does not exist, or resource is soft-deleted.        |
| 405  | Method Not Allowed — the HTTP verb is not supported for this route.  |
