# Accounts App Documentation

## Overview

The `accounts` app handles user registration and the admin-approval workflow. New users cannot log in until an administrator activates their account. The app has no custom database models — it works entirely through Django's built-in `User` model and the linked `Author` profile from the `authors` app.

---

## Models

**File:** `models.py`

No custom models. User data is stored in Django's built-in `auth.User` model. Each registered user gets a linked `Author` profile created at signup.

---

## Forms

**File:** `forms.py`

### `SignUpForm`

Extends Django's `UserCreationForm` with one additional field.

| Field | Type | Description |
|---|---|---|
| `username` | CharField | From `UserCreationForm`. |
| `displayName` | CharField (max 200) | Author display name, collected at signup. |
| `password1` / `password2` | PasswordField | From `UserCreationForm`. |

---

## Views

**File:** `views.py`

| View | Method | URL | Description |
|---|---|---|---|
| `signup` | GET, POST | `/accounts/signup/` | Renders the signup form (GET) or processes it (POST). On valid submission, creates a `User` with `is_active=False` and a linked `Author` with `isActive=False`, then redirects to the pending page. |
| `pending` | GET | `/accounts/pending/` | Displays a static "awaiting approval" page after signup. No logic — just a holding screen. |

**Signup flow:**

1. User submits the form with username, display name, and password.
2. A `User` is created with `is_active=False` (cannot log in yet).
3. An `Author` profile is created with `isActive=False` and `host` set to the server's base URL.
4. User is redirected to `/accounts/pending/`.
5. An administrator approves the user via the Django admin, setting both `is_active` and `isActive` to `True`.

---

## URLs

**File:** `urls.py`

| Pattern | Name | Notes |
|---|---|---|
| `signup/` | `accounts:signup` | Registration form |
| `pending/` | `accounts:pending` | Post-signup holding page |

---

## Admin

**File:** `admin.py`

Replaces Django's default `UserAdmin` with `CustomUserAdmin`, which adds two bulk actions:

| Action | Description |
|---|---|
| `approve_users` | Sets `user.is_active = True` and, if a linked `Author` exists, sets `author.isActive = True`. |
| `reject_users` | Sets `user.is_active = False` and, if a linked `Author` exists, sets `author.isActive = False`. |

If no linked `Author` profile exists for a user, the action silently continues without error.

---

## Tests

**File:** `tests.py`

No tests are currently implemented for this app.

---

## Key Behaviours

- **Approval-gated registration:** All new accounts start inactive. Users cannot log in or access any functionality until an admin explicitly approves them.
- **Paired activation:** Approving or rejecting a user always syncs the state to their linked `Author` profile, keeping both models consistent.
- **No custom model:** The app relies entirely on `auth.User` and `authors.Author`. There is no accounts-specific database table.
- **Host assignment:** The `Author` created at signup is given a hardcoded `host` value (`http://127.0.0.1:8000`), which should be updated to the production host URL before deployment.