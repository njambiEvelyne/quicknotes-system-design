# QuickNotes data model

QuickNotes uses PostgreSQL and the following four core entities. All timestamps are stored as `TIMESTAMPTZ` in UTC. `BIGINT GENERATED ALWAYS AS IDENTITY` supplies internal keys; the API can expose opaque IDs independently of these database keys.

## Entities and relationships

| Entity | Columns | Primary key | Foreign keys |
| --- | --- | --- | --- |
| `users` | `id BIGINT`, `email VARCHAR(320)`, `password_hash TEXT`, `display_name VARCHAR(120)`, `created_at TIMESTAMPTZ`, `updated_at TIMESTAMPTZ` | `id` | None |
| `notes` | `id BIGINT`, `user_id BIGINT`, `title VARCHAR(100)`, `body TEXT`, `created_at TIMESTAMPTZ`, `updated_at TIMESTAMPTZ` | `id` | `user_id` references `users.id` |
| `tags` | `id BIGINT`, `user_id BIGINT`, `name VARCHAR(50)`, `created_at TIMESTAMPTZ` | `id` | `user_id` references `users.id` |
| `note_tags` | `note_id BIGINT`, `tag_id BIGINT`, `created_at TIMESTAMPTZ` | (`note_id`, `tag_id`) | `note_id` references `notes.id`; `tag_id` references `tags.id` |

One user can own many notes and many tags (one-to-many). Notes and tags have a many-to-many relationship: a note can have multiple tags and each tag can be attached to multiple notes, represented by `note_tags`. Tags are scoped to their owner. Application logic must verify that a tag belongs to the note's owner before linking it.

## PostgreSQL schema

```sql
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email VARCHAR(320) NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    display_name VARCHAR(120) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE notes (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(100) NOT NULL CHECK (length(btrim(title)) > 0),
    body TEXT NOT NULL DEFAULT '',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tags (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(50) NOT NULL CHECK (length(btrim(name)) > 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, name)
);

CREATE TABLE note_tags (
    note_id BIGINT NOT NULL REFERENCES notes(id) ON DELETE CASCADE,
    tag_id BIGINT NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (note_id, tag_id)
);
```

## Indexes

```sql
CREATE INDEX notes_user_updated_idx ON notes (user_id, updated_at DESC, id DESC);
CREATE INDEX note_tags_tag_note_idx ON note_tags (tag_id, note_id);
```

`notes_user_updated_idx` supports a user's newest-first note list without scanning other users' notes. `note_tags_tag_note_idx` supports finding all notes linked to a tag; the composite primary key already supports lookups from a note to its tags.

## Example queries

List a user's most recently updated notes with cursor-friendly ordering:

```sql
SELECT id, title, body, created_at, updated_at
FROM notes
WHERE user_id = $1
ORDER BY updated_at DESC, id DESC
LIMIT $2;
```

Fetch notes and their tag names (JOIN across the many-to-many association):

```sql
SELECT n.id AS note_id, n.title, n.body, t.id AS tag_id, t.name AS tag_name
FROM notes AS n
LEFT JOIN note_tags AS nt ON nt.note_id = n.id
LEFT JOIN tags AS t ON t.id = nt.tag_id AND t.user_id = n.user_id
WHERE n.user_id = $1 AND n.id = $2;
```

Count notes per user:

```sql
SELECT u.id, u.email, COUNT(n.id) AS note_count
FROM users AS u
LEFT JOIN notes AS n ON n.user_id = u.id
GROUP BY u.id, u.email
ORDER BY note_count DESC;
```

Create a note and attach tags atomically:

```sql
BEGIN;
INSERT INTO notes (user_id, title, body)
VALUES ($1, $2, $3)
RETURNING id;
-- For each validated tag owned by $1:
INSERT INTO note_tags (note_id, tag_id) VALUES ($4, $5);
COMMIT;
```

## SQL vs NoSQL

PostgreSQL is the better starting point because users, notes, tags, and their ownership rules have clear relational integrity, and note/tag updates need atomic transactions. Strong constraints prevent orphaned or cross-user associations, while indexed user-scoped queries fit the access pattern. A single managed PostgreSQL primary with read replicas can scale this workload before the team considers partitioning; there is no need to accept the additional consistency and modeling complexity of a document store for the initial service.
