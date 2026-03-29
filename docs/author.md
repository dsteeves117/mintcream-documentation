# Authors App Documentation

## Overview

The `authors` app manages author profiles in the Social Distribution project. Each author represents a user on the network, identified by a stable UUID. Authors can be local (linked to a Django user account) or remote (from a federated node).

---

## Model: `Author`

**File:** `models.py`

Represents a user/author profile on the network.

| Field | Type | Description |
|---|---|---|
| `id` | UUIDField (PK) | Auto-generated UUID. Never changes. |
| `displayName` | CharField (max 32) | Public display name. |
| `isActive` | BooleanField | Whether the author is active. Inactive authors are treated as deleted. |
| `isPrivate` | BooleanField | Whether the author's profile is private. |
| `github` | URLField | Optional GitHub profile URL. |
| `profileImage` | URLField | Optional profile image URL. |
| `host` | URLField | The host node's base URL (e.g. `https://example.com/api/`). |
| `description` | TextField | Optional bio/description. |
| `user` | OneToOneField → `AUTH_USER_MODEL` | Linked Django user account. Nullable for remote authors. |

**Computed properties:**

- `name` — alias for `displayName`.
- `webUrl` — browser-facing profile URL (e.g. `https://example.com/authors/<uuid>`). Falls back to a relative path if `host` is not set.
- `fqid` — fully-qualified ID used in the API (e.g. `https://example.com/api/authors/<uuid>`).

---

## Selectors

**File:** `selectors.py`

Read-only functions for querying authors from the database.

| Function | Description |
|---|---|
| `get_active_authors()` | Returns all active authors ordered by `id`. |
| `get_all_authors()` | Returns all authors (active and inactive) ordered by `id`. |
| `get_author_or_404(author_id)` | Returns an active author by UUID, or raises HTTP 404. |

---

## Services

**File:** `services.py`

Business logic functions for authors.

| Function | Description |
|---|---|
| `get_me(request)` | Returns the `Author` profile linked to the currently authenticated user, or `None` if unauthenticated or no profile exists. |

---

## Views (Browser)

**File:** `views.py`

| View | Method | URL | Description |
|---|---|---|---|
| `index` | GET | `/authors/` | Lists all active authors. |
| `detail` | GET | `/authors/<uuid>/` | Author profile page. Shows public entries (newest first) and follow status. Profile owner sees their pending follow-request count. |
| `edit_profile` | POST | `/authors/<uuid>/edit/` | Updates the author's own `displayName`, `description`, `github`, and `profileImage`. Raises 403 if the user is not the profile owner. |

---

## URLs

**File:** `urls.py`

| Pattern | Name | Notes |
|---|---|---|
| `/authors/` | `authors:index` | Author list |
| `/authors/<uuid>/` | `authors:detail` | Author profile |
| `/authors/<uuid>/edit/` | `authors:edit_profile` | Edit profile (POST only) |
| `/authors/<uuid>/` | `follows` namespace | Includes `follows.urls` |
| `/authors/<uuid>/entries/` | `entries` namespace | Includes `entries.urls` |

---

## Admin

**File:** `admin.py`

The `AuthorAdmin` class registers `Author` in the Django admin with:

- **List display:** display name, short UUID, linked user, active status, GitHub, host, web URL link.
- **Filters:** `isActive`, `host`.
- **Search:** by `displayName`, `id`, `github`, `host`. Supports pasting a full UUID or a URL like `authors/<uuid>` to find an author by ID.
- **Actions:**
  - `approve_authors` — sets `isActive = True` on the author and its linked user.
  - `reject_authors` — sets `isActive = False` on the author and its linked user.
- **Inlines:** `FollowersInline`, `FollowingInline` from the `follows` app.

---

## Tests

**File:** `tests.py`

| Test Class | What it covers |
|---|---|
| `AuthorConsistentIdentityTests` | UUID-based URLs remain stable; inactive authors return 404; API `id` field is always a full URL. |
| `AuthorProfilePublicEntriesTests` | Profile page shows only `PUBLIC` entries, ordered newest first. Non-public entries are hidden. |
| `AuthorEditProfileTests` | Edit profile via browser form and via API PUT. Enforces ownership (403 for wrong user). |
| `AuthorsAPICollectionTests` | `GET /api/authors/` returns only active authors with full UUIDs. `POST /api/authors/` creates an author; validates required fields. |
| `AuthorAPIDetailTests` | `GET/PUT/PATCH/DELETE /api/authors/<uuid>/`. Covers field updates, soft-delete behaviour, 404 for inactive/missing authors, and public readability. |

---

## Key Behaviours

- **Soft delete:** Setting `isActive = False` effectively hides the author from all API responses and profile pages. The record is not removed from the database.
- **Stable identity:** The UUID primary key never changes, even when the display name is updated. All URLs remain valid.
- **Remote authors:** An `Author` without a linked `user` represents a profile from a federated remote node. The `host` and `fqid` fields identify which node they belong to.