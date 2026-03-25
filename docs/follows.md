# Follows App Documentation

## Overview

The `follows` app manages follow relationships between authors. Every follow starts as a pending request and must be explicitly approved by the target author. Mutual accepted follows make two authors "friends," which unlocks access to friends-only entries.

---

## Model: `Follow`

**File:** `models.py`

Represents a directional follow relationship from one author to another.

| Field | Type | Description |
|---|---|---|
| `follower` | ForeignKey → `Author` | The author initiating the follow. |
| `following` | ForeignKey → `Author` | The author being followed. |
| `fqid` | URLField (unique, nullable) | Fully-qualified API URL for this relationship. Auto-assigned on save if `BASE_URL` is configured. |
| `summary` | CharField (max 255) | Human-readable description (e.g. "Alice wants to follow Bob"). |
| `accepted` | BooleanField | `False` = pending request; `True` = approved follow. |
| `created_at` | DateTimeField | Auto-set on creation. |

**Constraints:**

- `unique_follow` — only one `Follow` row per `(follower, following)` pair.
- `no_self_follow` — a database-level check prevents an author from following themselves.

**`fqid` auto-assignment:** On `save()`, if `fqid` is empty and `settings.BASE_URL` is set, the field is populated as `<BASE_URL>/api/authors/<following_id>/followers/<url-encoded follower fqid>`.

---

## Selectors

**File:** `selectors.py`

Read-only functions for querying follow relationships.

| Function | Returns | Description |
|---|---|---|
| `get_followers(author)` | QuerySet | Accepted followers of `author` (pending excluded), ordered newest first. |
| `get_following_public(author)` | QuerySet | Authors that `author` follows, accepted only. For non-owner viewers. |
| `get_following_for_self(author)` | QuerySet | All outgoing follow relations including pending. For the owner's own view. |
| `get_pending_requests(author)` | QuerySet | Incoming follow requests awaiting approval, ordered newest first. |
| `is_following(follower, following) -> bool` | bool | Returns `True` if an accepted follow from `follower` to `following` exists. |
| `are_friends(author_a, author_b) -> bool` | bool | Returns `True` if both authors follow each other (both accepted). Used by entry access control. |

---

## Services

**File:** `services.py`

All service functions are wrapped in `@transaction.atomic`.

| Function | Description |
|---|---|
| `request_follow(*, actor, target, base_url="")` | Creates a pending follow (`accepted=False`) from `actor` to `target`. Raises `ValidationError` on self-follow. Idempotent — returns the existing record if one already exists. |
| `accept_follow(*, target, follower)` | Sets `accepted=True` on an existing follow. Returns `None` (no-op) if no record exists, allowing idempotent accept handling. |
| `reject_follow(*, target, follower)` | Deletes the pending follow request. No-op if the record doesn't exist. Only removes pending requests (`accepted=False`). |
| `unfollow(*, actor, target)` | Deletes any follow relation from `actor` to `target`, regardless of accepted state. |

---

## Views (Browser)

**File:** `views.py`

All views are nested under `/authors/<author_id>/` from the `authors` app.

| View | Method | URL | Description |
|---|---|---|---|
| `followers_page` | GET | `followers/` | Shows accepted followers. Pending requests are excluded. |
| `following_page` | GET | `following/` | Shows who the author follows. Owner sees pending + accepted; others see accepted only. Requires login. |
| `follow_requests_page` | GET | `requests/` | Shows incoming pending requests. Owner only — returns 403 for others, redirects anonymous users. |
| `send_follow_request` | POST | `follow/` | Creates a pending follow request from the logged-in user to this author. Self-follow returns 403. Redirects to target's profile. |
| `accept_request` | POST | `requests/<follower_id>/accept/` | Approves one pending request. Owner only. Idempotent redirect on missing request. |
| `reject_request` | POST | `requests/<follower_id>/reject/` | Denies one pending request. Owner only. Idempotent no-op if request is already gone. |
| `unfollow_action` | POST | `unfollow/` | Removes the follow relation from the logged-in user to this author. Self-unfollow returns 403. |

---

## URLs

**File:** `urls.py`

Mounted under `/authors/<author_id>/` via the `authors` app.

| Pattern | Name | Notes |
|---|---|---|
| `followers/` | `follows:followers` | Followers page |
| `following/` | `follows:following` | Following page |
| `requests/` | `follows:follow-requests` | Pending requests page |
| `follow/` | `follows:follow` | Send follow request (POST) |
| `unfollow/` | `follows:unfollow` | Unfollow (POST) |
| `requests/<uuid>/accept/` | `follows:accept` | Accept request (POST) |
| `requests/<uuid>/reject/` | `follows:reject` | Reject request (POST) |

---

## API Endpoints

**Files:** `follows/api/urls.py`, `follows/api/views.py`

Mounted under `/api/authors/<author_id>/`.

| Method | Path | Description |
|---|---|---|
| `GET` | `followers/` | List accepted followers for the author. |
| `GET` | `followers/<follower_fqid>/` | Check whether this author is an accepted follower. |
| `PUT` | `followers/<follower_fqid>/` | Approve an incoming follow request (owner-only). |
| `DELETE` | `followers/<follower_fqid>/` | Deny/revoke follower relation (owner-only, idempotent). |
| `GET` | `follow_requests/` | List pending incoming requests (owner-only). |
| `GET` | `following/` | List accepted outgoing follows for the author. |
| `GET` | `following/<followee_fqid>/` | Check whether author follows the target. |
| `PUT` | `following/<followee_fqid>/` | Create/reuse outgoing follow request (owner-only). |
| `DELETE` | `following/<followee_fqid>/` | Unfollow target (owner-only, idempotent). |

**FQID note:** relation endpoints use URL-encoded author FQIDs in the path.

---

## Admin

**File:** `admin.py`

`FollowAdmin` registers the `Follow` model with list display of follower, following, accepted status, and creation date. Supports filtering by `accepted` and searching by display name.

Two inline classes are exported for use in `AuthorAdmin` (in the `authors` app):

- `FollowersInline` — shows who follows the author (`Follow.following = parent`). Read-only with clickable links to follower profiles.
- `FollowingInline` — shows who the author follows (`Follow.follower = parent`). Read-only with clickable links to following profiles.

Both inlines allow deletion and show `accepted` status and creation date.

---

## Tests

**File:** `tests.py`

| Test Class | What it covers |
|---|---|
| `FollowRequestsWebTests` | Owner can view, accept, and reject requests via browser. Non-owners receive 403. Anonymous users are redirected. Followers page excludes pending requests. Accept/reject of non-existent requests are no-ops. |
| `FollowRequestsApiTests` | API accept (PUT → 200) and reject (DELETE → 204) on follower relation URL. Non-owner cannot list requests. Missing request accept/reject return correct status codes. |
| `FollowAPIAuthTests` | Unauthenticated access returns 401/403. Response shape validation (`type`, `followers`, `items` keys). Follower objects include `displayName` and full URL `id`. Self-follow returns 400. Pending requests excluded from followers list. Accept moves follower into list; reject removes from requests list. |

---

## Key Behaviours

- **Two-phase follow:** Every follow starts as `accepted=False`. It only becomes a real follow after the target author approves it. Pending requests are visible only to the target.
- **Friendship is derived:** Two authors are "friends" when both have accepted follow relations in each direction. This is checked by `are_friends()` and used by `entries.access.can_access_entry` to gate friends-only entries. There is no dedicated `/friends` endpoint.
- **Self-follow prevention:** Enforced at both the database level (check constraint) and service layer (`ValidationError`), and the API returns 400.
- **Idempotent operations:** Accept and reject are safe to call multiple times. `request_follow` uses `get_or_create` so repeated calls return the existing record.
- **Privacy:** The following page shows pending outgoing requests only to the owner. Public viewers see accepted follows only.
