# Persistent Priority Queue

A priority queue that supports both `extract_min` and `extract_max`, with
state persisted to disk (JSON file) so it survives process restarts. An
optional PostgreSQL-backed variant with the same interface is included as
a bonus (`postgres_backend.py`).

## Project structure

```
persistent-priority-queue/
├── README.md              # this file
├── DOCUMENTATION.md        # step-by-step walkthrough of the design/build process
├── module.py               # main implementation (required file)
├── postgres_backend.py     # optional: equivalent PostgreSQL-backed implementation
├── demo.py                 # small runnable usage walkthrough
├── test_module.py          # unit tests (17 tests, all passing)
└── requirements.txt        # dependencies (stdlib only for module.py)
```

## Setup / running

No external dependencies are needed for the required (file-based) implementation
— it only uses the Python standard library.

```bash
cd persistent-priority-queue
python3 demo.py              # runs a guided walkthrough
python3 -m unittest test_module.py -v   # runs the test suite
```

If you want to try the optional PostgreSQL backend:

```bash
pip install -r requirements.txt   # installs psycopg2-binary
```

```python
from postgres_backend import PostgresPriorityQueue
pq = PostgresPriorityQueue(dsn="dbname=pq_db user=pq_user password=secret host=localhost")
```

The table and index are created automatically on first connection if they
don't already exist — no separate init script is required.

## API

```python
from module import PriorityQueue

pq = PriorityQueue(storage_path="pq_data.json")

id = pq.insert(priority, value=None, id=None)   # -> id (string)
pq.extract_min()                                # -> {"id", "priority", "value"}
pq.extract_max()                                # -> {"id", "priority", "value"}
pq.peek(mode="min")                             # or mode="max"
pq.update(id, priority=None, value=None, update_value=False)
pq.delete(id)
pq.is_empty()                                   # -> bool
len(pq)                                         # number of items currently queued
```

Lower `priority` values are treated as "more urgent" for `extract_min`
(this is the usual convention — e.g. a task-scheduler priority queue) but
priorities can be any comparable number and `extract_max` is available
whenever you want the other end of the ordering.

## Implementation notes

A single binary heap is only efficient in one direction (min *or* max), so
supporting both `extract_min` and `extract_max` efficiently rules out "just
use `heapq`" directly. Two well-known approaches solve this:

1. A **min-max heap** (one array, alternating min/max levels) — elegant,
   but the trickle-down logic (comparing against both children and
   grandchildren) is fiddly to get right and easy to get subtly wrong.
2. **Two ordinary heaps + lazy deletion** — a min-heap and a max-heap that
   both reference the same canonical item store. This is the approach used
   here, because it reuses Python's well-tested `heapq` for both
   directions and is much easier to reason about and verify.

**How lazy deletion works.** `_entries` (a dict keyed by item id) is the
single source of truth for "what's actually in the queue right now." Every
entry in either heap carries `[priority, seq, id]` where `seq` is a
monotonically increasing counter value captured at push time. When
`update()` is called, we don't try to relocate the item inside the heap
arrays (that's an O(n) operation on a binary heap) — instead we bump the
seq counter, overwrite `_entries[id]`, and push a *new* `[priority, seq,
id]` triple into both heaps. The old heap entries are now "stale". When we
later pop from a heap, we check the popped entry's `(seq, priority)`
against the current `_entries[id]`; if they don't match (or the id was
deleted entirely), we discard it and keep popping. Because every stale
entry is discarded at most once over the life of the queue, this keeps all
operations at **O(log n) amortized**.

`delete(id)` is even simpler: just remove `id` from `_entries` (O(1) plus a
persist). Any heap entries for that id become stale and vanish lazily the
next time they'd otherwise surface at the top.

**Persistence.** Only `_entries` (plus a couple of small counters) is
written to disk — the heap arrays are not persisted, since they're cheap to
rebuild with `heapq.heapify()` from `_entries` on load and persisting them
would just duplicate information already implied by `_entries`. Every
mutating call (`insert`, `update`, `delete`, `extract_min`, `extract_max`)
writes the new state to a temp file and then atomically `os.replace()`s it
over the real file, so a crash mid-write can't leave a half-written,
corrupted JSON file behind.

**Complexity summary**

| Operation      | Time (amortized) |
|----------------|-------------------|
| insert         | O(log n)          |
| extract_min    | O(log n)          |
| extract_max    | O(log n)          |
| peek           | O(log n) worst case (cleans stale heap tops), O(1) typical |
| update         | O(log n)          |
| delete         | O(1) + O(log n) amortized cleanup deferred to later pops |
| is_empty       | O(1)              |

## Real-world use cases for priority queues

- **OS process/task scheduling** — the OS scheduler picks the next process
  to run based on priority (and can dynamically reprioritize with
  `update()`, e.g. anti-starvation aging).
- **Support/incident ticketing systems** — route the most urgent ticket to
  an agent first (`extract_min`), while still being able to see the
  lowest-priority backlog item (`extract_max`) for capacity planning. This
  is exactly what `demo.py` simulates.
- **Dijkstra's / A\* shortest-path algorithms** — repeatedly extract the
  unvisited node with the smallest tentative distance, and `update()` a
  node's distance when a shorter path to it is discovered ("decrease-key").
- **Event simulation / timers** — a discrete-event simulator or a timer
  wheel extracts the next event whose scheduled time is soonest.
- **Bandwidth/QoS packet scheduling** — network routers dequeue
  higher-priority packets (e.g. VoIP) ahead of lower-priority bulk traffic.
- **Load shedding / cache eviction** — evict the least valuable ("max"
  priority-as-cost) item first when a cache or queue is full.
- **Job queues with retries** — a background worker pool pulls the
  earliest-due job (`extract_min` on next-run-time), and failed jobs get
  `update()`d with a new backoff time and pushed back into the running
  order rather than requeued from scratch.

## Notes on design choices

- Priorities can be duplicated; ties are broken FIFO using the same `seq`
  counter used for staleness detection, so insertion order is preserved
  for equal priorities.
- IDs are auto-generated (`"1"`, `"2"`, ...) if not supplied, or you can
  pass your own `id=` to `insert()` and use it later for `update`/`delete`.
- `update()` can change priority, value, or both in a single call.
- The file-based version needs no external services and is what the test
  suite exercises. `postgres_backend.py` is provided as a drop-in
  alternative with an identical method surface for anyone who wants
  PostgreSQL-backed durability instead (uses a `(priority, seq)` B-Tree
  index so the RDBMS itself does the min/max lookups).
