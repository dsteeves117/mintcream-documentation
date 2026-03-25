# Nodes App Documentation

## Overview

The `nodes` app manages remote nodes and handles the inbox functionality for receiving objects from remote authors.

---

## Inbox

### `POST {remote node base url}/api/authors/{author_id}/inbox`

**When to use:** Send an object (entry, like, comment, or follow) to authors on remote nodes. 

#### Response `200 OK`
- `created` when a new object was created in the remote node's database
- `updated` when an existing entry was edited in the remote node's database
- `processed` when the remote node received the request but made no changes to its databse

#### Errors
- `400 Bad Request`
  - `Unknown Type` when object is not an entry, like, comment, or follow
  - `Invalid entry payload.` when there is missing or invalid id in request body of an entry request
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

## Requests

**Show list of known nodes** `GET /nodes/`

**Response:**
```
200 OK
Content-Type: text/html
```

Renders an HTML page displaying a list of the names and urls of known RemoteNode objects


**Create new RemoteNode object** `POST /nodes/add/`

**Response:**
```
200 OK
Content-Type: text/html
```


**Edit an existing RemoteNode object** `<int:node_id>/edit/`

**Response:**
```
200 OK
Content-Type: text/html
```


**Delete a RemoteNode object** `<int:node_id>/delete/`

**Response:**
```
200 OK
Content-Type: text/html
```