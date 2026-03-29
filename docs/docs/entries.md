# Entries App Documentation

## Overview

The `entries` app manages posts (entries), comments, and likes in the Social Distribution project. Entries support multiple content types and visibility levels. GitHub activity can be automatically imported as public entries.

---

## Models

**File:** `models.py`

### `Entry`

Represents a post created by an author.

| Field | Type | Description |
|---|---|---|
| `uuid` | UUIDField (PK) | Auto-generated, never changes. Used in all URLs. |
| `fqid` | URLField (unique, nullable) | Fully-qualified API URL (e.g. `https://host/api/authors/<aid>/entries/<uuid>`). Lazily assigned on first access. |
| `author` | ForeignKey → `Author` | The author who created the entry. |
| `title` | CharField (max 200) | Entry title. |
| `description` | CharField (max 400) | Short summary. |
| `content_type` | CharField (choices) | See `EntryContentType` below. |
| `content` | TextField | Entry body. Image entries store base64-encoded data here. |
| `visibility` | CharField (choices) | See `EntryVisibility` below. |
| `published` | DateTimeField | Creation timestamp (defaults to now). |
| `updated_at` | DateTimeField | Auto-updated on save. |
| `source_id` | CharField (indexed) | External deduplication key (e.g. `github:<event_id>`). |

**Computed methods/properties:**

- `is_deleted()` — returns `True` if `visibility == DELETED`.
- `is_image()` — returns `True` if `content_type` starts with `image/`.
- `image_mime_type` — returns the MIME type without the `;base64` suffix.
- `is_markdown` — returns `True` if `content_type == text/markdown`.

---

### `EntryVisibility`

| Value | Description |
|---|---|
| `PUBLIC` | Visible to everyone. |
| `FRIENDS` | Visible only to the author and mutual friends. |
| `UNLISTED` | Visible to anyone with a direct link; hidden from listings. |
| `DELETED` | Soft-deleted; hidden from all normal views and listings. |

---

### `EntryContentType`

| Value | Description |
|---|---|
| `text/plain` | Plain text. |
| `text/markdown` | Markdown (rendered in templates). |
| `application/base64` | Generic base64-encoded binary. |
| `image/png;base64` | PNG image stored as base64. |
| `image/jpeg;base64` | JPEG image stored as base64. |

---

### `Comment`

Represents a comment on an entry.

| Field | Type | Description |
|---|---|---|
| `uuid` | UUIDField (PK) | Auto-generated. |
| `fqid` | URLField (unique, nullable) | Fully-qualified API URL. |
| `author` | ForeignKey → `Author` | The commenter. |
| `entry` | ForeignKey → `Entry` | The entry being commented on. |
| `comment` | TextField | Comment text. |
| `content_type` | CharField | `text/plain` or `text/markdown` only. |
| `published` | DateTimeField | Creation timestamp. |
| `updated_at` | DateTimeField | Auto-updated on save. |

Default ordering: newest first (`-published`).

---

### `Like`

Represents a like on an entry.

| Field | Type | Description |
|---|---|---|
| `uuid` | UUIDField (PK) | Auto-generated. |
| `fqid` | URLField (unique, nullable) | Fully-qualified API URL. |
| `author` | ForeignKey → `Author` | The author who liked the entry. |
| `entry` | ForeignKey → `Entry` (nullable) | The liked entry. |
| `object_fqid` | URLField | URL of the liked object (for API payloads). |
| `summary` | CharField | Human-readable summary (e.g. "Alice likes your entry"). |
| `published` | DateTimeField | Timestamp. |

Constraint: one like per `(author, entry)` pair — duplicate likes are rejected at the database level.

Default ordering: newest first (`-published`).

---

## Access Control

**File:** `access.py`

`can_access_entry(entry, viewer_author) -> bool`

Centralized read-access gate used by both web and API layers.

| Condition | Result |
|---|---|
| Entry is `DELETED` | `False` (always denied) |
| Entry is `PUBLIC` or `UNLISTED` | `True` (no auth required) |
| No authenticated author | `False` |
| Viewer is the entry's author | `True` |
| Entry is `FRIENDS` and viewer is a mutual friend | `True` |
| Otherwise | `False` |

---

## Services

**File:** `services.py`

| Function | Description |
|---|---|
| `create_comment(*, author, entry, text, content_type)` | Creates and persists a new `Comment`. Validation is the caller's responsibility. |
| `like_entry(*, author, entry)` | Creates a `Like` or returns the existing one (idempotent). Returns `(like, created)`. Backfills `object_fqid` and `summary` on older rows if missing. |
| `like_comment(*, author, comment)` | Creates a `Like` targeting a comment (`object_fqid = comment.fqid`) with idempotent behavior for repeated likes by the same author. |

---

## Views (Browser)

**File:** `views.py`

All views are nested under `/authors/<author_id>/entries/`.

| View | Method(s) | URL | Description |
|---|---|---|---|
| `entries_page` | GET, POST | `/` | Lists an author's non-deleted entries. POST creates a new entry (owner only). Image entries accept a file upload. |
| `edit_entry` | GET, POST | `/<entry_id>/edit/` | Edit form for an existing entry. Owner only. Returns 403 otherwise. |
| `entry_detail_view` | GET, POST | `/<entry_id>/` | View an entry. POST with `_method=DELETE` soft-deletes it. POST with content fields updates it. Enforces visibility rules on GET. |
| `add_comment` | POST | `/<entry_id>/comments/` | Adds a comment. Requires login and access to the entry. Blank comments are silently ignored. |
| `like_comment_action` | POST | `/<entry_id>/comments/<comment_id>/like/` | Likes a comment from the entry detail page. Requires login and comment visibility access. Idempotent on repeat. |
| `like_entry_action` | POST | `/<entry_id>/like/` | Likes an entry. Idempotent. Requires login and access. |
| `github_sync_view` | POST | `/github-sync/` | Triggers GitHub activity import for the author. Owner only. Redirects with `?synced=<count>` on success. |
| `entry_image_view` | GET | `/<entry_id>/image` | Returns raw binary image data for an image entry. Returns 404 for non-image entries. |

**Helper functions:**

- `_ensure_entry_fqid(request, entry)` — lazily backfills `fqid` if not yet set. Called before creating comments or likes.
- `_require_me(request)` — returns the logged-in author or raises `PermissionDenied`.

---

## URLs

**File:** `urls.py`

Mounted at `/authors/<author_id>/entries/` from the `authors` app.

| Pattern | Name | Notes |
|---|---|---|
| `/` | `entries:page` | Entry list + create |
| `/<uuid>/` | `entries:detail` | Entry detail, edit, soft-delete |
| `/<uuid>/edit/` | `entries:edit` | Edit form |
| `/<uuid>/comments/` | `entries:add_comment` | Add comment |
| `/<uuid>/comments/<uuid>/like/` | `entries:like_comment` | Like comment |
| `/<uuid>/like/` | `entries:like` | Like entry |
| `/<uuid>/image` | `entries:image` | Raw image data |
| `/github-sync/` | `entries:github_sync` | GitHub activity sync |

---

## GitHub Sync

**File:** `github_sync.py`

`sync_github_activity(author, request=None) -> int`

Fetches the author's public GitHub events and creates `PUBLIC` entries for any not yet imported.

- Extracts the GitHub username from `author.github` URL.
- Fetches up to 30 events from the GitHub API.
- Skips events already imported (matched via `source_id = "github:<event_id>"`).
- Builds a Markdown title and content for each event using `_EVENT_TITLES` templates.
- Returns the count of newly created entries.
- Network or JSON errors are caught silently; the function returns `0`.

**Supported GitHub event types:** `PushEvent`, `CreateEvent`, `DeleteEvent`, `ForkEvent`, `WatchEvent`, `IssuesEvent`, `IssueCommentEvent`, `PullRequestEvent`, `PullRequestReviewCommentEvent`, `ReleaseEvent`, `PublicEvent`, `MemberEvent`, `CommitCommentEvent`, `GollumEvent`.

---

## Admin

**File:** `admin.py`

| Model | Key features |
|---|---|
| `Entry` | List by title, author, visibility, content type, published date. Auto-fills `fqid` on save if missing. `uuid` and `updated_at` are read-only. |
| `Comment` | List by entry, author, content type, published date. `uuid`, `published`, and `updated_at` are read-only. |
| `Like` | List by entry, author, published date. `uuid` and `published` are read-only. |

All three models use `autocomplete_fields` for author/entry lookups.

---

## Tests

**File:** `tests.py`

| Test Class | What it covers |
|---|---|
| `EntryDetailShareableLinkTests` | Visibility rules: public/unlisted accessible to anyone; deleted → 404; friends-only → 403 for anonymous. UUID stable after edit. API `fqid` is a full URL. |
| `EntryEditDeleteTests` | Owner can edit and soft-delete entries via browser. Non-owners receive 403. Deleted entries are excluded from listings. |
| `GitHubSyncTests` | Sync creates public entries from GitHub events. Duplicate events are not re-imported (deduplication). |
| `EntryCommentLikeWebTests` *(truncated)* | Web form comment/like flows. Includes comment-like action, friendship checks on friends-only entries, idempotent likes, and anonymous redirect behavior. |
| `EntryCommentLikeApiTests` | API comment and like endpoints. Includes comment-like list/create endpoints, response shape validation (`type`, `src`, `count`), friends-only visibility, and idempotent creates (`201` then `200`). |

---

## Key Behaviours

- **Soft delete:** Setting `visibility = DELETED` hides an entry everywhere. The database row is preserved.
- **Stable UUID:** The `uuid` primary key never changes, keeping all URLs valid after edits.
- **Lazy `fqid`:** The fully-qualified API URL is assigned on first use (entry creation, comment/like creation, or GitHub sync). Older rows are backfilled automatically.
- **Image storage:** Images are stored as base64-encoded strings in the `content` field. The raw binary can be retrieved via the `/image` endpoint.
- **Idempotent likes:** The `unique_entry_like_per_author` database constraint and `get_or_create` logic prevent duplicate likes regardless of how many times the action is triggered.
- **Friends-only comment visibility:** On friends-only entries, comments are visible to friends and to the comment author for their own comment resource.
