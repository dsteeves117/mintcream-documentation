# Nodes App Documentation

## Overview

The `nodes` app manages remote nodes and handles the inbox functionality for receiving objects from remote authors.

---

## Inbox

### `POST {remote node base url}/api/authors/{author_id}/inbox`

**When to use:** Send an object (entry, like, comment, or follow) to authors on remote nodes. 

**Why not:** Do not use if you are creating objects locally.

#### Request fields
- `type`          (choose one of `follow`, `entry`, `like`, `comment`. Required.)
- `type: follow`
  - `actor` or `follower` (Required, interchangeable)
- `type: entry`
  - `id`          (Required)
  - `author`      (default = source of request)
  - `title`       (default = "")
  - `description` (default = "")
  - `contentType` (default = text/plain)
  - `visibility`  (default = public)
  - `content`     (default = "")
  - `published`   (default = timezone.now())
- `type: like`
  - `object` or `object_fqid` (Required, interchangeable)
  - `id`
  - `author`      (default = source of request)
  - `published`   (default = timezone.now())
  - `summary`     (default = "")
- `type: comment`
  - `entry`       (Required)
  - `author`      (default = source of request)
  - `published`   (default = timezone.now())
  - `comment`     (default = "")
  - `contentType` (default = text/plain)

#### Response `200 OK`
- `created` when a new object was created in the remote node's database
- `updated` when an existing entry was edited in the remote node's database
- `processed` when the remote node received the request but made no changes to its databse

#### Errors
- `400 Bad Request`
  - `Unknown Type` when object is not an entry, like, comment, or follow
  - `Invalid entry payload.` when there is missing or invalid id in request body of an entry request
  - `Invalid visibility` when the visibility of an entry is not one of "PUBLIC", "UNLISTED", "FRIENDS", "DELETED"
  - `Invalid like payload.` when there is missing or invalid id in request body of a like request
  - `Invalid comment payload.` when there is missing or invalid entry id in request body of a comment request
  - `Comment id is required.` when there is missing or invalid id in request body of a comment request
  - `Invalid follow payload.` when we cannot identify the following user in a follow request
  - `Could not resolve follow actor.` when the followed user in a follow request does not exist
  - `Cannot follow self.` when the remote user id matches the following user's id in a follow request

- `401 Unauthorized`
  - `Unauthorized` when the remote node in a request cannot be verified

- `403 Forbidden`
  - `Node is not active or not recognized` when the node's is_active field is set to False or the node does not exist in our database

- `404 Not Found`
  - `Referenced entry not found.` when the entry being commented on in a comment request does not exist

---

## Models

**File:** `models.py`

### RemoteNode

Represents a node other than our own that we are connected to.

| Field | Type | Description |
|---|---|---|
| `url` | URLField (unique) | Base URL of node we wish to access. (e.g. `https://otherteam.herokuapp.com`) |
| `username` | CharField (max 100) | Username to access remote node |
| `password` | CharField (max 100) | Password to access remote node |
| `is_active` | BooleanField | Default set to True. Can be changed to False to disable nodes. |
| `display_name` | CharField (max 100) | Label for the node. |
| `created_at` | DateTimeField | Autofilled on creation |

---

## Forms

**File:** `forms.py`

### RemoteNodeForm

A form for creating or editing nodes.

model: RemoteNode

| Field | Type | Description |
|---|---|---|
| `url` | URLField (unique) | Base URL of node we wish to access. (e.g. `https://otherteam.herokuapp.com`) |
| `username` | CharField (max 100) | Username to access remote node |
| `password` | CharField (max 100) | Password to access remote node |
| `is_active` | BooleanField | Default set to True. Can be changed to False to disable nodes. |
| `display_name` | CharField (max 100) | Label for the node. |

---

## Requests

**Show list of known nodes** `GET /nodes/`

**Response:**
```
200 OK
Content-Type: text/html
```

**When to use:** Used to render an HTML page displaying a list of the names and urls of known RemoteNode objects.



**Create new RemoteNode object** `POST /nodes/add/`

**Response:**
```
200 OK
Content-Type: text/html
```
Renders a RemoteNodeForm with action='Add'.

**When to use:** Used to create a new node and add it to the database.


**Edit an existing RemoteNode object** `POST <int:node_id>/edit/`

**Response:**
```
200 OK
Content-Type: text/html
```
Renders a RemoteNodeForm with action='Edit'.

**When to use:** Used to modify an existing node in the database.


**Delete a RemoteNode object** `POST <int:node_id>/delete/`

**Response:**
```
200 OK
Content-Type: text/html
```
Renders a 'confirm delete' page.

**When to use:** Used to delete an existing node from the database.