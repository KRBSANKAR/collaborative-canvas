# ARCHITECTURE

## Data Flow Diagram (High-Level)

```
User Input (mouse/touch) 
   -> local prediction draw 
   -> emit 'stroke:begin' + 'stroke:points' (batched) + 'stroke:end' over WebSocket
   -> server validates/assigns opId, appends to room history, updates version
   -> server broadcasts deltas ('stroke:begin/points/end') to room
   -> all clients apply deltas incrementally
   -> on global undo/redo: server updates history/tombstones, bumps version, broadcasts 'history:patch' (undo/redo)
   -> clients re-render deterministically from history (fast path uses cached bitmap)
```

## WebSocket Protocol (Room-scoped)

**Events (client → server):**
- `join` `{ room: string }`
- `cursor:update` `{ x, y }`
- `stroke:begin` `{ opId, color, size, mode: 'draw'|'erase', start: {x,y}, ts }`
- `stroke:points` `{ opId, points: [{x,y,ts}, ...] }` (batched; can repeat many times)
- `stroke:end` `{ opId }`
- `undo` `{}` (global – undo latest non-tombstoned op)
- `redo` `{}` (global – redo latest tombstoned op)

**Events (server → client):**
- `init` `{ self: {id,color}, users: [{id,color}], history: [ops], tombstones: [opId], version }`
- `user:join` `{ id, color }`
- `user:leave` `{ id }`
- `cursor:update` `{ id, x, y }`
- `stroke:begin|points|end` (same payloads as client, authoritative broadcast)
- `history:patch` `{ action: 'undo'|'redo', opId, version }`

**Operation Shape (stored in history):**
```
{
  opId: string,
  author: string,
  color: string,
  size: number,
  mode: 'draw'|'erase',
  points: [{x:number,y:number,ts:number}] // includes the starting point
}
```

## Undo/Redo Strategy (Global)

- The **server is authoritative**. Each room has:
  - `history[]`: list of committed ops (strokes).
  - `tombstones Set<opId>`: ops currently undone.
  - `version`: monotonic integer to detect stale clients.
- `undo`: remove the latest non-tombstoned op (irrespective of author) by adding it to `tombstones` and broadcasting `history:patch`.
- `redo`: remove the latest entry from tombstones by restoring its order (tracked by `undoStack[]`) and broadcast it.
- Clients re-render by replaying `history` excluding tombstones. For performance, they keep an **offscreen cache** and apply only the delta when possible. On version mismatch or after several undos, they do a full re-render.

## Conflict Resolution

- Overlapping drawings are **order-dependent** (latest op on top) which matches natural canvas semantics.
- Global undo/redo is **intentional** (last action wins globally). If needed, a future enhancement could be **per-user undo** by filtering on `author`.
- For in-flight strokes from multiple users, we stream points; each client draws their own prediction immediately, then corrects if server rebroadcast differs (rare).

## Performance Decisions

- **Batching:** `stroke:points` batches high-frequency mouse/touch points (e.g., every 8–12ms or N points).
- **Smoothing:** Client does **quadratic curve smoothing** between points to reduce jagged lines.
- **Redraw:** 
  - Fast path: incremental draw during streams.
  - Slow path: full replay from history on version jumps or resync.
- **Canvas Layers:** Single visible canvas + offscreen canvas for full replay; this avoids flicker.
- **Dirty Rects:** (Optional) Could track min/max bounds per op to optimize regional replays under high history counts.

## Scaling Discussion

- **1K users**: split across rooms; use **Redis adapter** for Socket.IO to scale horizontally. Persist history and tombstones in Redis/DB. Use **message compression** and **binary encoding** for points.
- **CDN static**: serve client via CDN; terminate WebSockets behind a load balancer supporting sticky sessions or leverage Socket.IO with Redis adapter.
- **Backpressure**: throttle `stroke:points` per-socket and drop redundant cursor updates.

## Persistence (Optional)

- Store `history` per room in a DB (e.g., Postgres JSONB) or object storage (NDJSON). On `init`, load the latest snapshot + deltas.
- Periodic **snapshots**: store rasterized PNG every N ops to bound replay time.

## Security Notes

- No auth by scope; server assigns ephemeral IDs. For multi-tenant demos, restrict room names and rate-limit point streams.
