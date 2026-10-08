# QuickNotes System Design

QuickNotes is a browser API-client exercise and a system-design package for a notes service intended to scale to one million registered users. The front end uses JSONPlaceholder as a practice API; the design documents describe a separate production QuickNotes API and backend.

## Run the API client

The static client has no build step or package dependencies. From the repository root, start a local static web server:

```sh
python -m http.server 8000
```

Then open <http://localhost:8000>. Select **Load notes** to fetch ten posts, or use the form to create a note and its **Delete** button to remove it from the displayed list. The client requires an internet connection to `jsonplaceholder.typicode.com`.

JSONPlaceholder is a practice service: POST responses look like created records, and DELETE responses report success, but neither operation persists server-side changes. The client displays successful operations locally; reloading notes restores the server's original sample data.

## Design documents

- [API design](docs/api-design.md) — production REST endpoints, examples, and error contract.
- [Data model](docs/data-model.md) — PostgreSQL schema, relationships, indexes, and queries.
- [Architecture](docs/architecture.md) — requirements, capacity estimates, architecture, flows, and trade-offs.

## What I learned

- Designing REST endpoints means specifying ownership, validation, pagination, and error behavior—not only choosing paths and verbs.
- A relational schema and transaction boundaries make user ownership and many-to-many note/tag relationships enforceable.
- Capacity estimates are only useful when their workload assumptions are explicit and revisited against measured traffic.
- Caches and replicas improve read scale but introduce freshness and failover trade-offs that need deliberate handling.
