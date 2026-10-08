# QuickNotes service architecture

## Requirements

### Functional

- Users can register, authenticate, and manage sessions.
- An authenticated user can create, list, retrieve, update, and delete only their own notes.
- Users can create tags and associate multiple tags with notes.
- Note lists support pagination, and updates are reflected consistently on subsequent reads.

### Non-functional

- Protect credentials and note data in transit and at rest; enforce authorization on every resource operation.
- Keep the service available across an application instance or availability-zone failure.
- Scale stateless API capacity horizontally and keep normal note-list latency low.
- Back up persistent data and monitor latency, errors, saturation, queue lag, and replica lag.
- Meet a first-release target of 99.9% monthly API availability and p95 read latency below 300 ms, excluding client network time.

## Capacity estimate for 1 million registered users

This is a planning estimate, not a forecast. Assume 10% of registered users are active daily (100,000 DAU), with 20 note reads and 2 note writes per active user per day. Spread average traffic across 86,400 seconds; provision for a 10x peak-hour/burst factor.

| Measure | Average | 10x planning peak |
| --- | ---: | ---: |
| Reads | 2,000,000/day = about 23 requests/second | about 230 requests/second |
| Writes | 200,000/day = about 2.3 requests/second | about 23 requests/second |

For storage growth, assume each daily active user creates two notes each day, and each note plus row/index overhead averages 1.5 KB. That is 73 million new notes per year and approximately 110 GB/year of primary database storage (73M × 1.5 KB), excluding replication and backups. A read replica roughly doubles live database storage; retaining backups and operational headroom makes a practical initial budget around 330 GB/year. Revisit the estimate with measured note sizes, activity, retention, and peak-to-average ratios.

## High-level architecture

```text
                         +------------------+
                         |   DNS (health     |
                         |   checked)        |
                         +---------+--------+
                                   |
Browser / mobile client --> CDN --> redundant managed load balancer
                                          |
                              +-----------+-----------+
                              |                       |
                       +------v------+         +------v------+
                       | API server 1|         | API server 2|   (autoscaled,
                       +------+------+\        +------+------+\    multi-AZ)
                              |       \               |       \
                              |        +------+-------+        |
                              |               |                |
                       +------v---------------v----------------v------+
                       |          replicated cache cluster             |
                       +----------------------+------------------------+
                                              |
                                      +-------v--------+
                                      | Primary DB     |
                                      | (multi-AZ)     |
                                      +---+---------+--+
                                          | async replication
                                 +--------v-------+
                                 | Read replica   |
                                 +----------------+

                       API servers --> durable replicated queue
                                               |
                                      +--------v--------+
                                      | Worker pool     |
                                      +-----------------+
```

## Component responsibilities

- **Client:** Presents the note interface and sends authenticated API requests over HTTPS.
- **DNS:** Resolves the service name and directs clients to healthy regional endpoints.
- **CDN:** Caches static assets near users, reducing page-load latency and origin traffic.
- **Load balancer:** Health-checks API instances and spreads requests across healthy servers.
- **API servers (at least two):** Run stateless validation, authentication, authorization, and note business logic so capacity can scale horizontally.
- **Cache:** Holds short-lived, frequently read data (for example, session or note-list results) to reduce database read load; writes invalidate or refresh affected entries.
- **Primary database:** Durably commits authoritative user, note, tag, and association data.
- **Read replica:** Serves eligible read traffic and reporting queries, separating read scale from primary writes.
- **Queue:** Durably buffers non-interactive work such as notifications or exports so API requests do not wait for it.
- **Worker:** Processes queued jobs with retries and idempotency, independently scalable from request-serving API servers.

## Request flows

### `GET /v1/notes`

1. The client resolves the API hostname through DNS, fetches static app assets from the CDN, and sends an HTTPS request with its bearer token to the load balancer.
2. The load balancer routes the request to a healthy API server; the server validates the token and derives the user ID from it.
3. The API checks a user-scoped cache entry. On a miss, it queries the read replica (or the primary when read-after-write consistency is required) using the indexed `user_id` and sort order.
4. The API serializes the page and cursor, returns `200 OK`, and caches the result briefly. Cache keys include the user ID so users cannot see one another's data.

### `POST /v1/notes`

1. The client sends an HTTPS JSON request with its bearer token through DNS, CDN routing for the API hostname, and the load balancer to a healthy API server.
2. The API authenticates the user, validates the title/body and any tag ownership, then inserts the note and tag links in one database transaction on the primary.
3. After commit, the API invalidates affected user-scoped note-list cache entries and may publish independent follow-up work to the durable queue.
4. The API returns `201 Created` with the persisted note and timestamps. A worker handles queued notifications/exports asynchronously; it is not required for the create response.

## Trade-offs and failure handling

- **Relational consistency vs. independent scaling:** PostgreSQL provides transactions and foreign-key enforcement for ownership and tag links. The trade-off is that scaling writes is more involved than adding stateless API servers; start with indexed queries and a read replica, then partition only after measurement.
- **Cache speed vs. freshness:** Caching reduces read load and latency but can briefly serve stale lists. Use short TTLs and invalidate user-scoped note lists after successful writes; route a read immediately after a write to the primary where read-your-writes is required.
- **Async work vs. operational complexity:** A durable queue keeps slow side effects off the user-facing path, but introduces retries, duplicate delivery, and lag. Make workers idempotent, monitor queue age, and define dead-letter handling.
- **Single points of failure:** Run at least two API instances across availability zones behind a health-checking redundant managed load balancer; use multi-AZ database failover plus tested backups and point-in-time recovery; deploy replicated cache and queue services; run multiple workers; and use health-checked DNS/CDN edge redundancy. The read replica is not the sole copy of data and can be rebuilt from the primary. Regular failover and restore exercises verify these protections.
