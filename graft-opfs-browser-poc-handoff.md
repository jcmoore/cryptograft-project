# Graft OPFS Browser PoC Handoff

## Goal

Build a **browser-compatible proof of concept** for Graft that can **participate in the same remote replication protocol and S3 object layout as native Graft**, while keeping the implementation primarily in **TypeScript** and avoiding an immediate Rust/Wasm port.

The working assumption for the first PoC is:

- keep the existing **upper sqlite/pglite layering** conceptually intact,
- replace the **native Fjall-backed local Graft store** with a **browser-native local store** built on **wa-sqlite**,
- preserve **wire/storage compatibility** with native Graft at the **remote object format and replication semantics** layer.

---

## Why this exists

The existing native prototype already establishes a useful multi-layer stack:

1. **PGlite VFS** persists PGlite data.
2. That VFS stores its backing data in another **virtual filesystem backed by SQLite**.
3. The Graft SQLite extension then uses a **Graft VFS backed by Fjall**, which persists locally and replicates remotely.

That stack gives a nice property today:

- SQLite/PGlite sees a filesystem-like substrate.
- The inner SQLite file holds encrypted PGLite file chunks that are opaque on disk and in the cloud (though file paths remain transparent).
- Graft handles lazy / partial / strongly consistent replication.

The browser problem is that **#3 currently depends on Fjall**, and the browser does not have a direct equivalent that cleanly matches Graft’s use of **immutable snapshots** and **transactional isolation**.

A direct IndexedDB KV adapter like `browser-level` is probably **not sufficient** because Graft relies on repeatable snapshot-style reads, while `browser-level` explicitly does **not** provide snapshot guarantees across repeated iterator calls.

Therefore the best seam for a browser PoC is:

> **Replace the local Graft/Fjall implementation (#3) with a browser-native SQLite-backed implementation that preserves Graft’s higher-level replication behavior.**

---

## Relevant existing files and references

### User prototype / current stack

- `pglite-cryptograft-afs/src/cryptograft-afs.ts`
  - https://github.com/jcmoore/pglite-cryptograft-afs/blob/cryptograft-afs/src/cryptograft-afs.ts
- `pglite-cryptograft-afs/examples/pglite-cryptograft.ts`
  - https://github.com/jcmoore/pglite-cryptograft-afs/blob/cryptograft-afs/examples/pglite-cryptograft.ts
- AgentFS VFS spec (basis for the SQLite-backed VFS schema, but there is some divergence)
  - https://github.com/tursodatabase/agentfs/blob/main/SPEC.md#virtual-filesystem

### Graft

- Repo: https://github.com/orbitinghail/graft
- Architecture docs: https://graft.rs/docs/internals/
- Future work docs: https://graft.rs/docs/internals/future/

### Browser SQLite / storage

- wa-sqlite persistence / VFS comparison:
  - https://www.powersync.com/blog/sqlite-persistence-on-the-web#vfs-summary-table
- SQLite Wasm persistence docs:
  - https://sqlite.org/wasm/doc/trunk/persistence.md
- Partytown notes on sync-feeling cross-thread calls:
  - https://partytown.qwik.dev/how-does-partytown-work/

---

## Important observed facts

### From the current prototype

The current custom VFS setup uses:

```ts
const bootstrap = new Database(":memory:");
bootstrap.loadExtension(graftExtPath);
bootstrap.close();

const db = new Database(openUri);
```

Source:
- `examples/pglite-cryptograft.ts`
- https://github.com/jcmoore/pglite-cryptograft-afs/blob/cryptograft-afs/examples/pglite-cryptograft.ts#L39-L53

The SQLite-backed VFS prototype also diverges intentionally from the base AgentFS schema by:

- giving each inode its own `chunk_size`, and
- storing file data in per-inode tables named like `fs_data_inode_<ino>`.

Representative snippets:

```ts
function chunkTableName(ino: number): string {
  return `fs_data_inode_${Math.trunc(ino)}`
}
```

```ts
SELECT ino, mode, nlink, uid, gid, size, atime, mtime, ctime, rdev, chunk_size
FROM fs_inode
WHERE ino = ?
```

Source:
- `src/cryptograft-afs.ts`
- https://github.com/jcmoore/pglite-cryptograft-afs/blob/cryptograft-afs/src/cryptograft-afs.ts

### From Graft’s architecture docs

Graft’s local/remote split is conceptually:

- local state: `tags`, `volumes`, `log`, `pages`
- remote state: `commits`, `segments`, `checkpoints`

Graft’s transaction model depends on:

- **snapshot isolation for reads**, and
- **strict commit serialization**.

Key lines from the docs:

> “All reads operate against immutable Snapshots.”

> “VolumeWriters provide transactional write isolation on top of a snapshot.”

This matters because the browser implementation must preserve the **logical contract** of snapshots and commit serialization even if the local storage engine is different.

### From Graft’s future-work docs

Graft’s docs explicitly mention browser/Wasm as a desired direction and specifically name:

- SQLite official Wasm build,
- `wa-sqlite`,
- `sql.js`.

This is a strong signal that a **SQLite-backed browser implementation** is aligned with the project’s intended future direction.

### From browser SQLite persistence resources

`wa-sqlite`’s **`OPFSCoopSyncVFS`** is a good first candidate because it is positioned as a synchronous-access-handle based VFS with good performance and recent-browser support.

Important caveat: OPFS is **worker-only** in SQLite’s official browser persistence docs.

That implies the first PoC should strongly prefer running the browser-side local store in a **dedicated worker**.

### From Partytown

Partytown is relevant for the *pattern*, not because we should literally adopt it.

Useful idea:

- a worker can expose **blocking/synchronous-feeling APIs** using `SharedArrayBuffer + Atomics.wait()` when cross-origin isolation is enabled,
- otherwise one can fall back to service-worker / synchronous-XHR style interception patterns.

This is likely relevant because Graft’s higher layers expect a **synchronous-feeling storage boundary**.

---

## Decision summary

For the first browser PoC:

1. **Do not try to port Fjall.**
2. **Do not use `browser-level` / IndexedDB KV wrappers as the primary store.**
3. Implement a **new browser-local Graft store in TypeScript**.
4. Back that store with **wa-sqlite**, using **`OPFSCoopSyncVFS`** first.
5. Preserve compatibility with native Graft by matching:
   - remote object naming/layout,
   - commit/segment/checkpoint formats,
   - lazy pull / partial replication semantics,
   - page and log semantics expected by native peers.

---

## Scope of the first PoC

### In scope

- Browser runtime implemented mainly in **TypeScript**.
- New **browser local-store adapter** for Graft semantics.
- `wa-sqlite`-backed local persistence.
- Dedicated worker runtime.
- Remote object compatibility with native Graft.
- Cross-runtime compatibility testing:
  - native writes -> browser reads/pulls,
  - browser writes -> native reads/pulls.

### Explicitly out of scope for phase 1

- Reusing existing Rust code through Wasm.
- Perfect performance parity with native Graft.
- Full browser compatibility matrix from day one.
- Production-ready background sync / UX.

### Phase-2 / later scope

- GC / compaction / checkpoint optimization.
- Broader browser support.

---

## Functional requirements

The browser implementation must:

1. **Open and maintain a local metadata/page store** in the browser.
2. Support **snapshot-style logical reads** suitable for Graft’s `VolumeReader` semantics.
3. Support **transactional writes** with:
   - immutable base snapshot,
   - staged writes,
   - read-your-write semantics,
   - validation/serialization on commit.
4. Support **lazy / partial replication**.
5. Store and fetch **remote segments / commits / checkpoints** using the same format/protocol expected by native Graft.
6. Be able to consume objects written by native Graft.
7. Be able to emit objects consumable by native Graft.
8. Provide enough sync behavior that the higher layers (#1 and #2 in the existing stack) can function without being deeply rewritten around async storage.

---

## Non-functional requirements

1. **Primary implementation language**: TypeScript.
2. **Storage backend**: `wa-sqlite` first.
3. **Initial VFS**: `OPFSCoopSyncVFS`.
4. **Thread model**: dedicated worker.
5. **API seam**: storage adapter abstraction so the VFS choice is swappable later.
6. **Repository strategy**: implement in a **fork of the Graft repo**.
7. **Testing strategy**: compatibility against native Graft, not just unit tests in isolation.

---

## Why not browser-level / LevelDB-family adapters?

Because this PoC is not merely asking for:

- ordered KV,
- range scans,
- browser persistence.

It specifically needs something closer to:

- immutable snapshot reads,
- transactional staging over a base snapshot,
- commit-time validation/serialization,
- long-lived logical consistency while scanning state relevant to replication.

A plain IndexedDB-backed KV adapter is too low-level and too weak semantically for this without rebuilding a substantial amount of machinery above it.

SQLite-in-browser is a better foundation because:

- the logical transactional machinery is stronger,
- storage can still live in browser persistence,
- it is closer to how the current stack already thinks about data,
- Graft itself already points toward SQLite/Wasm/wa-sqlite as a future direction.

---

## Proposed repository layout in the Graft fork

Suggested additions inside a fork of `orbitinghail/graft`:

```text
/graft
  /graft-opfs
    /src
      /bridge
        main-thread-bridge.ts
        worker-bridge.ts
        sab-rpc.ts
      /sqlite
        sqlite-factory.ts
        opfs-coop-vfs.ts
        schema.ts
        migrations.ts
      /store
        graft-opfs-store.ts
        snapshot.ts
        tx.ts
        volume-reader.ts
        volume-writer.ts
        log-store.ts
        page-store.ts
        tag-store.ts
        checkpoint-store.ts
      /remote
        remote-client.ts
        segment-codec.ts
        commit-codec.ts
        checkpoint-codec.ts
        sync.ts
      /compat
        native-fixtures.ts
        format-tests.ts
      /tests
        browser-native-roundtrip.test.ts
        browser-push-native-pull.test.ts
        native-push-browser-pull.test.ts
        interrupted-upload.test.ts
        lazy-page-fetch.test.ts
      index.ts
    package.json
    tsconfig.json
```

This does **not** need to mirror the Rust file layout exactly. The goal is to create a browser-focused compatibility layer that can evolve independently while still living in the same fork.

---

## Architectural plan

### 1. Create a storage adapter boundary immediately

Define a small interface so the rest of the browser Graft logic is not coupled directly to `wa-sqlite` or OPFS.

Example:

```ts
export interface GraftOPFSLocalStore {
  init(): Promise<void>

  beginReadSnapshot(volumeId: string, at?: bigint): Promise<OPFSSnapshot>
  beginWrite(volumeId: string): Promise<OPFSWriteTxn>

  readPage(volumeId: string, pageNo: number, snapshot: OPFSSnapshot): Promise<Uint8Array | null>
  getHeadLSN(volumeId: string): Promise<bigint | null>

  applyRemoteCommit(input: ApplyRemoteCommitInput): Promise<void>
  materializeSegment(input: MaterializeSegmentInput): Promise<void>
}

export interface OPFSSnapshot {
  readonly volumeId: string
  readonly snapshotId: string
  readonly visibleLogs: readonly string[]
  readonly lsnUpperBound: bigint
}

export interface OPFSWriteTxn {
  readPage(pageNo: number): Promise<Uint8Array | null>
  writePage(pageNo: number, data: Uint8Array): Promise<void>
  commit(): Promise<{ commitLsn: bigint }>
  rollback(): Promise<void>
}
```

The internal implementation can be SQLite-heavy even if this public seam remains logically KV/page oriented.

---

### 2. Use SQLite tables to model Graft’s local state

Do **not** try to emulate Fjall at the key/value API level. Instead, directly model the logical pieces Graft needs.

Initial schema direction:

```sql
CREATE TABLE gopfs_volume (
  volume_id TEXT PRIMARY KEY,
  created_at_ms INTEGER NOT NULL,
  head_lsn TEXT,
  meta_json BLOB NOT NULL
);

CREATE TABLE gopfs_log (
  volume_id TEXT NOT NULL,
  log_id TEXT NOT NULL,
  start_lsn TEXT NOT NULL,
  end_lsn TEXT,
  parent_log_id TEXT,
  state TEXT NOT NULL,
  PRIMARY KEY (volume_id, log_id)
);

CREATE TABLE gopfs_page_version (
  volume_id TEXT NOT NULL,
  page_no INTEGER NOT NULL,
  lsn TEXT NOT NULL,
  log_id TEXT NOT NULL,
  segment_id TEXT,
  inline_bytes BLOB,
  content_hash BLOB,
  PRIMARY KEY (volume_id, page_no, lsn)
);

CREATE INDEX gopfs_page_visible_idx
  ON gopfs_page_version(volume_id, page_no, lsn DESC);

CREATE TABLE gopfs_commit (
  volume_id TEXT NOT NULL,
  lsn TEXT NOT NULL,
  commit_id TEXT NOT NULL,
  log_id TEXT NOT NULL,
  segment_id TEXT,
  commit_json BLOB NOT NULL,
  PRIMARY KEY (volume_id, lsn)
);

CREATE TABLE gopfs_checkpoint (
  volume_id TEXT NOT NULL,
  replica_id TEXT NOT NULL,
  checkpoint_json BLOB NOT NULL,
  PRIMARY KEY (volume_id, replica_id)
);

CREATE TABLE gopfs_tag (
  tag TEXT PRIMARY KEY,
  value_json BLOB NOT NULL
);
```

Notes:

- `TEXT` for LSNs is acceptable initially if encoded canonically (or use fixed-width blobs / integers if JS bigint transport is settled cleanly).
- This schema is intentionally logical, not a direct copy of Fjall internals.
- Staged transaction state can either live in temp tables or in normal tables keyed by a txn id.

---

### 3. Represent snapshots logically, not as long-lived storage-engine cursors

The browser implementation should **not** depend on long-lived cursor/iterator snapshot semantics from the underlying browser persistence API.

Instead:

- a snapshot should capture a **logical visibility boundary**,
- e.g. `{volumeId, visible log ranges, lsnUpperBound}`,
- reads resolve the latest visible page version `<= lsnUpperBound` consistent with visible logs.

This is much closer to Graft’s architecture docs and avoids building the whole design around fragile iterator behavior.

Example query shape:

```sql
SELECT pv.inline_bytes, pv.segment_id, pv.content_hash, pv.lsn, pv.log_id
FROM gopfs_page_version pv
WHERE pv.volume_id = ?
  AND pv.page_no = ?
  AND pv.lsn <= ?
ORDER BY pv.lsn DESC
LIMIT 1;
```

If a page version is not materialized locally, it can trigger lazy fetch of the referenced segment/page bytes.

---

### 4. Stage writes separately from committed state

A browser-side write transaction should maintain:

- base snapshot metadata,
- staged page writes,
- read-your-write behavior,
- commit-time validation.

Candidate approach:

```sql
CREATE TABLE gopfs_txn_page_stage (
  txn_id TEXT NOT NULL,
  volume_id TEXT NOT NULL,
  page_no INTEGER NOT NULL,
  page_bytes BLOB NOT NULL,
  PRIMARY KEY (txn_id, page_no)
);
```

Read path during a write txn:

1. check staged table first,
2. otherwise read through base snapshot visibility rules.

Commit path:

1. acquire commit lock / serialize commit attempt,
2. validate head/base snapshot assumptions,
3. assign next LSN(s),
4. encode segment / commit metadata,
5. persist committed versions locally,
6. publish remote objects,
7. update head/checkpoint state.

The exact transaction orchestration can start conservative and simple. Correctness matters more than elegance in the first pass.

---

### 5. Run the local store in a dedicated worker

Because OPFS is worker-only, assume this topology:

```text
main thread
  -> worker bridge
    -> wa-sqlite + OPFSCoopSyncVFS
      -> graft-opfs local store
        -> remote replication client
```

Two possible bridge modes:

#### Preferred mode

- `SharedArrayBuffer + Atomics.wait()` RPC bridge.
- Requires cross-origin isolation headers.
- Gives the higher layers a more synchronous-feeling interface.

#### Fallback mode

- a service-worker / sync-XHR style bridge only if absolutely necessary.

---

### 6. Preserve remote compatibility as the primary contract

The most important compatibility target is **not** “matching Fjall internally.”

It is:

- native Graft can consume browser-written remote objects,
- browser Graft can consume native-written remote objects.

That means the browser implementation must match:

- segment encoding,
- commit metadata format,
- checkpoint format,
- ordering / publish semantics,
- lazy page materialization semantics,
- whatever atomicity assumptions native Graft expects around remote visibility.

If the remote format/codec code can be copied conceptually from the Rust implementation into TS, that is preferred over inventing a parallel format.

---

## Concrete milestones

### Milestone 0 — repo setup and analysis

- Use submodule fork located under  `modules/github.com/orbitinghail/graft/v/0`.
- Add `graft-opfs/` package.
- Identify native code paths that define:
  - segment encoding,
  - commit encoding,
  - checkpoint handling,
  - remote object naming,
  - sync-point semantics.
- Document those paths in comments / README for the browser package.

Deliverable:

- checked-in scaffold,
- list of native source files that define compatibility requirements.

### Milestone 1 — local SQLite store only

- Implement `sqlite-factory.ts`.
- Initialize `wa-sqlite` + `OPFSCoopSyncVFS` in a worker.
- Create migrations and schema.
- Implement local snapshot reads and local staged write transactions.
- No remote sync yet.

Deliverable:

- tests proving snapshot-style logical reads and commit serialization locally.

### Milestone 2 — remote read compatibility

- Implement remote client for reading native-written commits / segments / checkpoints.
- Browser can pull native data and materialize readable pages.
- Lazy page fetch works.
- Use SeaweedFS to pride the required S3-compatible endpoint on localhost (refer to the submodule under `modules/github.com/seaweedfs/seaweedfs/v/4`).

Deliverable:

- `native -> browser` compatibility test passes.

### Milestone 3 — remote write compatibility

- Implement browser-side segment/commit generation.
- Publish browser-generated objects to the same remote format as native Graft.
- Native Graft can pull and consume browser-written updates.

Deliverable:

- `browser -> native` compatibility test passes.

### Milestone 4 — integrate with the higher stack

- Replace the current native-only Graft layer (#3) with the browser `graft-opfs` equivalent.
- Validate that the existing PGlite + inner SQLite-backed VFS concept still works through the browser local store.
- Measure whether a SAB/Atomics bridge is required for acceptable behavior.

Deliverable:

- end-to-end demo with browser participant in replication.

### Milestone 5 — fallback / robustness

- Add restart/recovery tests.
- Add interrupted upload / partial download recovery tests.
- Add cross-browser smoke tests.

---

## Testing matrix

The compatibility matrix matters more than microbenchmarks.

### Core interop cases

1. **Native writes, browser pulls, browser reads.**
2. **Browser writes, native pulls, native reads.**
3. **Browser writes, browser reloads, browser continues.**
4. **Native and browser interleave commits on same volume, sequentially.**

### Failure / recovery cases

5. Interrupted segment upload.
6. Commit uploaded after segment delay.
7. Lazy page fetch after cache eviction.
8. Browser crash/reload during local staged txn.
9. Recovery from stale checkpoint / partial local materialization.

### Storage backend cases

10. `OPFSCoopSyncVFS`.

---

## Suggested initial implementation notes for the agent

### Start simple on commit serialization

Do not overcomplicate the first commit path.

A simple single-row lock table is acceptable for the PoC:

```sql
CREATE TABLE gopfs_commit_lock (
  lock_name TEXT PRIMARY KEY,
  owner_id TEXT NOT NULL,
  acquired_at_ms INTEGER NOT NULL
);
```

or even just rely on a single writer worker in the first pass.

The point is to preserve **correct serialized commits**, not to perfectly mirror native internals immediately.

### Inline bytes first, segment indirection second

For the first local-store-only milestone, it is acceptable to keep page bytes inline in SQLite.
Later, if necessary, segment indirection and lazy fetch can be layered in more aggressively.

### Prefer compatibility fixtures over reverse engineering from prose

If native Graft can generate small deterministic fixtures, use them.
Examples:

- single-volume single-page fixture,
- multi-commit fixture,
- branch/merge-like log topology fixture,
- partially replicated fixture.

Then write TS tests that consume those fixtures byte-for-byte.

---

## Risks / open questions

1. **How much of Graft’s remote format logic is easy to port into TS?**
   - This determines whether compatibility is straightforward or whether some shared spec/fixtures need to be authored first.

2. **Do higher layers truly require blocking sync calls, or is worker-async good enough initially?**
   - This affects the necessity of OPFSCoopSyncVFS, SAB/Atomics, and/or synchronous-XHR in the first pass.

3. **What exact invariants does native Graft rely on from Fjall snapshots?**
   - The docs suggest the important contract is logical snapshot isolation, but verify this in code.

4. **What is the minimum viable browser set?**
   - Recent Chromium + Firefox is likely easiest first.
   - Safari may require more care depending on OPFS path and worker model.

---

## Recommended first coding order

1. Set up `graft-opfs/` package in a Graft fork.
2. Build worker + `wa-sqlite` + `OPFSCoopSyncVFS` bootstrap.
3. Implement schema + migrations.
4. Implement local logical snapshots.
5. Implement local write txn staging + serialized commit.
6. Add small deterministic native fixtures.
7. Implement remote read compatibility.
8. Implement remote write compatibility.

---

## Short brief for the agentic coder

Implement a **TypeScript browser-side Graft runtime** inside a fork of the Graft repo.

The runtime should replace the **Fjall-backed local store** with a **wa-sqlite-backed local store** using **`OPFSCoopSyncVFS`** in a dedicated worker. The browser implementation must preserve **Graft’s logical snapshot + commit semantics** and, most importantly, must remain **compatible with native Graft’s remote replication artifacts** so browser and native peers can exchange commits/segments/checkpoints through the same remote backend.

Do not try to port Fjall. Do not start with Rust/Wasm reuse. Do not start with `browser-level`. Start with a clean TS implementation of the **logical storage semantics** Graft needs.

---

## Appendix: selected references

- Current prototype: `cryptograft-afs.ts`
  - modules/github.com/jcmoore/pglite-cryptograft-afs/v/0/src/cryptograft-afs.ts
- Current prototype example: `pglite-cryptograft.ts`
  - modules/github.com/jcmoore/pglite-cryptograft-afs/v/0/examples/pglite-cryptograft.ts
- AgentFS VFS spec
  - modules/github.com/tursodatabase/agentfs/v/0/SPEC.md#virtual-filesystem
- Graft repo
  - modules/github.com/orbitinghail/graft/v/0
- Graft architecture
  - https://graft.rs/docs/internals/
- Graft future work
  - https://graft.rs/docs/internals/future/
- wa-sqlite / persistence comparison
  - https://www.powersync.com/blog/sqlite-persistence-on-the-web#vfs-summary-table
- SQLite Wasm persistence
  - https://sqlite.org/wasm/doc/trunk/persistence.md
- Partytown communication model
  - https://partytown.qwik.dev/how-does-partytown-work/
