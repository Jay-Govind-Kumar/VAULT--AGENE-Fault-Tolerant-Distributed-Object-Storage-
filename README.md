# VAULT v2.4 — Fault-Tolerant Distributed Object Storage

VAULT is a hackathon-ready distributed object-storage system with a central control plane and independently running storage-node agents.

## Problem-statement coverage

| Requirement | VAULT implementation |
|---|---|
| Store objects | Multipart upload API + local node storage |
| Large volumes | Configurable chunking (default 8 MiB) |
| Replicate | Configurable replication factor (RF) |
| Durability policies | Configurable write quorum (W) and read quorum (R) |
| Concurrent reads/writes | Async FastAPI + concurrent replica writes + batch multi-file uploads + runtime telemetry |
| Node failure | Kill/revive simulation + heartbeat failure detection |
| Partial network partition | Partition/heal simulation; node remains a process but is unreachable |
| Data corruption | Byte-level corruption injection + SHA-256 of actual stored bytes |
| Replica inconsistency | Stale-version injection + version verification |
| Integrity verification | Checksums, expected-vs-actual byte hash, version checks |
| Automatic replica repair | Background repair worker |
| Metadata consistency | SQLite metadata with WAL, foreign keys, indexes and migrations |
| Background rebalancing | Automatic rebalancer + manual `POST /api/rebalance` |
| Predictable availability | Quorum-based writes and reads; reads fail only when the configured R cannot be met |
| Recovery time | Repair events record start/completion/duration |
| Storage overhead | Physical/logical usage and replication-overhead metrics |
| Live observability | WebSocket event stream + metrics dashboard |

## Local Windows run

### First setup

Install Python 3.10+ and the backend requirements:

```bat
python -m pip install -r requirements.txt
```

You can also run the same install from inside `backend` with `python -m pip install -r requirements.txt`.

For the automated test suite:

```bat
python -m pip install -r backend\requirements-dev.txt
```

### Start everything

From the project root:

```bat
start_local.bat
```

The launcher starts **one coordinator process**. The coordinator automatically starts the 8 storage-node agents.

Dashboard:

```text
http://127.0.0.1:8000/
```

API documentation:

```text
http://127.0.0.1:8000/docs
```

Health helper:

```bat
CHECK_VAULT.bat
```

Stop the local cluster by closing the coordinator process or using:

```bat
stop_local.bat
```

Do not double-click `frontend/index.html` for the functional demo.

## What happens at startup

```text
start_local.bat
    ↓
FastAPI coordinator :8000
    ↓
Node 1 ... Node 8
8101 ... 8108
    ↓
Heartbeat monitor
Repair worker
Integrity scanner
Rebalancer
Object-status reconciler
    ↓
Dashboard + WebSocket
```

## Upload policy

The dashboard supports single-file and multi-file uploads. Node cards now display precise storage utilization plus recent activity telemetry instead of rounding small values to 0%. Up to 20 files can be submitted in one batch; the coordinator processes up to 3 files concurrently so one failed file does not discard successful uploads.

Every upload can choose:

```text
RF = replication factor
W  = write quorum
R  = read quorum
```

Example:

```text
RF=3
W=2
R=2
```

The coordinator writes to healthy nodes concurrently. A write is accepted only when W successful replica writes are achieved. A read verifies replicas and requires R verified replicas before serving data.

## Failure and recovery demo

Recommended judge sequence:

1. Upload a file with `RF=3, W=2, R=2`.
2. Open the object detail and show chunk placement.
3. Kill one replica node.
4. Watch the node switch to `FAILED` and the object become temporarily under-replicated.
5. Background repair chooses a healthy source and replacement node.
6. The replacement is written and verified before the repair is marked complete.
7. Inject byte corruption into a replica.
8. The integrity scanner hashes the actual bytes and detects the mismatch.
9. Inject a stale-replica version to demonstrate replica inconsistency detection.
10. Partition and then heal a node.
11. Run `Rebalance now` to demonstrate safe replica movement between uneven nodes.
12. Use the live event feed to explain each state transition.

## Automated test suite

Run:

```bat
RUN_TESTS.bat
```

The integration suite starts a temporary 8-node cluster and checks:

```text
✓ 8-node startup and health
✓ concurrent uploads
✓ concurrent multi-file batch uploads
✓ configurable RF/W/R policy
✓ live node storage/activity telemetry
✓ large-object chunking
✓ concurrent reads
✓ read quorum behavior
✓ node failure detection
✓ automatic replica repair
✓ byte corruption detection and repair
✓ replica version inconsistency detection and repair
✓ network partition and heal
✓ rebalancing endpoint without reducing RF
✓ API/health/policy/metrics contract
✓ SHA-256 unit test
```

The v2.4 build was exercised in the build environment with the full integration suite; the backend suite completed with **2 passed tests** (unit + end-to-end integration test).

## API

```text
GET    /api/health
GET    /api/cluster
GET    /api/metrics
GET    /api/policies
GET    /api/objects
GET    /api/objects/{object_id}
POST   /api/objects
POST   /api/objects/batch
GET    /api/objects/{object_id}/download
DELETE /api/objects/{object_id}
POST   /api/nodes/{id}/kill
POST   /api/nodes/{id}/revive
POST   /api/nodes/{id}/partition
POST   /api/nodes/{id}/heal
POST   /api/nodes/{id}/corrupt/{chunk_id}
POST   /api/nodes/{id}/inconsistent/{chunk_id}
POST   /api/repair
POST   /api/rebalance
POST   /api/chaos
GET    /api/events
WS     /ws
```

## Online deployment

For Vercel/Netlify, deploy only the static `frontend` directory and set `frontend/config.js` (or an equivalent environment-specific configuration) to your public FastAPI backend URL and WebSocket URL.

The backend requires a persistent host capable of running long-lived Python processes and persistent storage. The architecture is still the same:

```text
Vercel/Netlify frontend
        ↓ HTTPS/WSS
FastAPI coordinator
        ↓
storage-node agents
        ↓
persistent node storage + SQLite metadata
```

## Data locations

```text
data\
├── vault.db
├── node-01\
├── node-02\
├── ...
├── node-08\
└── logs\
```

The metadata database is authoritative for placement/state. Physical object chunks stay on node storage directories.

### Telemetry semantics
The dashboard's **Operations** metric shows cumulative completed object operations as `readR / writeW`. The smaller detail line shows how many reads/writes completed in the last 60 seconds and how many coordinator operations are active right now. The node cards show physical node `ops/min`, total node operations, storage utilization, and the last operation time. A value such as `0 active` is normal when the cluster is idle; it is not the total number of operations.

The dashboard uses lightweight, dependency-free, spring-like Web Animations/CSS transitions for node state changes, metric updates, event entry, charts, and controls. No frontend package install is required for the local demo.
