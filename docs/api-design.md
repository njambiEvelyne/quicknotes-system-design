# QuickNotes REST API

This document describes the production QuickNotes API, not the JSONPlaceholder practice service. All endpoints are versioned under `/v1`, use JSON, and require an `Authorization: Bearer <access-token>` header except registration and login. IDs are opaque strings in the public API.

## Endpoints

| Method | Path | Description | Success status |
| --- | --- | --- | --- |
| POST | `/v1/users` | Register an account | `201 Created` |
| POST | `/v1/sessions` | Authenticate and create a session | `201 Created` |
| GET | `/v1/notes` | List the authenticated user's notes; accepts `limit` and `cursor` | `200 OK` |
| POST | `/v1/notes` | Create a note owned by the authenticated user | `201 Created` |
| GET | `/v1/notes/{noteId}` | Retrieve one of the authenticated user's notes | `200 OK` |
| PATCH | `/v1/notes/{noteId}` | Update a note's title, body, or tags | `200 OK` |
| DELETE | `/v1/notes/{noteId}` | Delete a note and its tag links | `204 No Content` |
| GET | `/v1/tags` | List the authenticated user's tags | `200 OK` |

List results use a stable, opaque `nextCursor`; omit it when there is no next page. `limit` defaults to 20 and is capped at 100. The service derives `userId` from the access token; clients cannot assign notes to another user. A note title is required and limited to 100 characters. Bodies may be empty.

## Create a note

Request:

```http
POST /v1/notes HTTP/1.1
Authorization: Bearer <access-token>
Content-Type: application/json
```

```json
{
  "title": "Plan the launch",
  "body": "Review the checklist before Friday.",
  "tagIds": ["tag_8f3a"]
}
```

Response (`201 Created`):

```json
{
  "id": "note_01J8Q6M2",
  "userId": "user_01J8Q5A1",
  "title": "Plan the launch",
  "body": "Review the checklist before Friday.",
  "tags": [
    { "id": "tag_8f3a", "name": "work" }
  ],
  "createdAt": "2026-10-08T16:45:00Z",
  "updatedAt": "2026-10-08T16:45:00Z"
}
```

## List notes

Request:

```http
GET /v1/notes?limit=2 HTTP/1.1
Authorization: Bearer <access-token>
```

Response (`200 OK`):

```json
{
  "items": [
    {
      "id": "note_01J8Q6M2",
      "userId": "user_01J8Q5A1",
      "title": "Plan the launch",
      "body": "Review the checklist before Friday.",
      "tags": [{ "id": "tag_8f3a", "name": "work" }],
      "createdAt": "2026-10-08T16:45:00Z",
      "updatedAt": "2026-10-08T16:45:00Z"
    },
    {
      "id": "note_01J8Q60P",
      "userId": "user_01J8Q5A1",
      "title": "Reading list",
      "body": "",
      "tags": [],
      "createdAt": "2026-10-07T12:00:00Z",
      "updatedAt": "2026-10-07T12:00:00Z"
    }
  ],
  "nextCursor": "eyJ1cGRhdGVkQXQiOiIyMDI2LTEwLTA3VDEyOjAwOjAwWiJ9"
}
```

## Errors

Errors use `application/json` and a consistent envelope. `requestId` lets support correlate a response with server logs.

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "title is required and must be at most 100 characters.",
    "requestId": "req_01J8Q7"
  }
}
```

| Status | Meaning | Example |
| --- | --- | --- |
| `400 Bad Request` | Malformed JSON, invalid fields, or invalid pagination | Missing or overlong title |
| `401 Unauthorized` | Missing, invalid, or expired credentials | Expired bearer token |
| `403 Forbidden` | Authenticated user is not allowed to perform the operation | Attempt to update a note owned by another user |
| `404 Not Found` | Requested resource does not exist (or is not visible to this user) | Unknown note ID |
| `500 Internal Server Error` | Unexpected server-side failure | Database operation failed unexpectedly |

Clients should not receive internal exception details in a `500` response. All errors include a stable machine-readable `code` and a safe, human-readable `message`.
