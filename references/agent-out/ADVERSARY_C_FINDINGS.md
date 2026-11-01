# ADVERSARY C — The Intelligence Architect: findings on the pk33 intelligence layer

**Seat:** C (recreation case) · **Host:** pk33 · **Date:** 2026-11-02
**Scope of work:** read-only enumeration of the code-knowledge-graph / context-pack
layer, the case for rebuilding rather than restoring each component, a target
architecture, a build order with a verification gate, and a ruthless list of what
to throw away.

**Standing constraint honoured:** the intelligence layer is PAUSED. All six `zk`
crons are `DISABLED` in `efus3/_logs/agent-crons/registry.tsv` (lines 135, 138, 144,
145, 152, 153) and `crontab -l | grep -c "pcl-selfcron:zk"` returns **0**. Nothing was
re-enabled, restarted, deleted, moved, edited or committed. Every claim below comes
from a file I opened or a command I ran; commands are quoted inline.

**One correction to the brief up front.** The brief says the KG index holds
"35,919 docs / 737,128 chunks". Those are two different stores conflated:
`index.sqlite` holds **134,812 docs**; 35,919 is the *embedding population* (md
docs), and 737,128 is the chunk count in `vec.sqlite`. Also, `fd5fa4a83` is
**historical, not present**: I measured **0** graphify-derived paths in the live
index. The *class* of defect it represents is still live, through `_logs/`. Details
in INVENTORY §1.4 and §1.5.

---

## INVENTORY

### 0. The estate of the intelligence layer, in one table

| # | Component | Path | Bytes | Input | Output | Who reads it |
|---|---|---|---|---|---|---|
| 1 | FTS5 index | `zdash222/_logs/zk/index.sqlite` | 2.2 G | 134,063 files across 46 scopes | `docs` FTS5 table + `meta` | `bin/zk search`, `zk_embed.py`, `zk_check_paths.sh` |
| 2 | Vector store | `zdash222/_logs/zk/vec.sqlite` | 3.9 G | `index.sqlite` md docs | 737,128 chunks × 1024-dim | `bin/zk vsearch` (query embedding) |
| 3 | Stale index | `zdash222/_logs/zk/index.sqlite.prev` | 858 M | Sept-15 15-scope generation | 52,719 docs | **nobody** |
| 4 | Code graphs | `<repo>/graphify-out/` × 53 dirs | 7.7 G | repo source tree | `graph.json` + `GRAPH_REPORT.md` + `cache/` + 99 dated snapshot dirs | `zk explain` (graph.json only); `zk_graph.sh --check` (mtime only) |
| 5 | Scope list | `zdash222/_logs/zk/scopes.tsv` (+ `.indexed`) | 2.6 K | `docs/registers/clones.tsv` + `~/.claude/projects/*/memory` + `docs/knowledge/zk-scopes-extra.tsv` | 53 rows / 47 scope names | `zfts.py` (walk), `zk_graph.sh`, `zk_embed.py` |
| 6 | Context packs | `zexp1/zprotocols/ctx/` | 220 K | `efus3/scripts/LANES.tsv`, `skills/_meta/registry.tsv`, `zversionkeeper.sh`, `zclaudemanager.sh`, `factory6000/docs/assessment` | 25 `CONTEXT-v1.0.N.md` + `LATEST.md` + 25 `.sha256` | `skills/zctx/SKILL.md` (agents read LATEST), ZP-008, `zk_report.sh` |
| 7 | CLI surface | `zdash222/bin/zk` (12,902 B) + `efus3/scripts/zfts.py` (20,929 B) + `bin/zk_embed.py` (23,034 B) + `bin/zk_graph.sh` (2,806 B) + `bin/zk_report.sh` (1,778 B) + `bin/zk_tunnel.sh` (1,255 B) + `bin/zk_check_paths.sh` (2,122 B) | 65 K | scope list + repo trees | the seven surfaces above | agents via `efus3/skills/zk/SKILL.md`; DoD rows in `docs/dod/WORK.tsv` |
| 8 | Second (separate) FTS index | `efus3/_logs/zfts/index.sqlite` | 12.3 M | efus3 builtin scopes (skills/registers/protocols/receipts) | 1,281 docs | `zfts.sh q`; still live via crontab `11 4 * * *` |
| 9 | Parallel vector layer | `zexp1/zvec1/` (`data/zvec.sqlite`, `data/models/`) | 514 M + 65 M | efus3 only | 140,808 chunks × 384-dim bge-small-en-v1.5, model cached locally | **nobody** (register A-14 verdict `NEVER-WIRED`) |

Total on-disk footprint of the derived layer: **≈ 15.5 GB** (6.9 GB in `_logs/zk/`
+ 7.7 GB of `graphify-out` + 0.6 GB elsewhere).

### 1. The FTS index (`index.sqlite`) — measured, not inherited

Schema, from `efus3/scripts/zfts.py:208-211`:

```sql
CREATE VIRTUAL TABLE docs USING fts5(path UNINDEXED, scope UNINDEXED,
    kind UNINDEXED, title, body, tokenize='porter unicode61');
CREATE TABLE meta (key TEXT PRIMARY KEY, value TEXT);
```

Live state (read with `sqlite3` in `mode=ro`):

```
built_at = 2026-10-31T21:00:48Z
n_docs   = 134812            (code 89,980 · md 40,959 · tsv 3,873)
49 scope rows in meta
```

The index is **three weeks stale** (built 21:00Z on Oct 31; the pause landed 21:57Z
the same night, per `efus3/_logs/rework-ledger/2026-10-31.md` row 1.3).

**1.1 The rot is still in the index, not just in the scope list.** The scope list was
pruned at 21:57Z (commit `c3fa3e7`) but the index was not rebuilt. Comparing
`scopes.tsv` against `meta`:

```
scopes in INDEX but NOT in current scopes.tsv: ['3.24.5', 'zt331-not-worthy']
   3.24.5           -> /home/pk33/fvm/versions/3.24.5   (Flutter SDK)  825 docs
   zt331-not-worthy -> /home/pk33/trash/zt331-not-worthy (discarded)       1 doc
```

**1.2 One scope can never be satisfied — the perpetual-RED engine.** `bin/zk scopes`
emits a row for every `host==pk33` clone in `docs/registers/clones.tsv` **without
checking that the root exists** (`bin/zk:63-66`); only the *extra* rows get an
existence check (`bin/zk:135-138`). One pk33 clone root no longer exists:

```
/home/pk33/awgbackups/pk33-cleanup      <- MISSING (the dir is now /home/pk33/pk33backupsv1/pk33-cleanup)
```

So the `pk33-cleanup` scope currently contributes **6 stale docs** and, on the next
rebuild, **0**. `zfts.py cmd_check_coverage` counts any scope with `meta[scope:X] < 1`
as empty and returns 1 (`zfts.py:296-314`) → `bin/zk status` prints `STALE` forever.
This is not an accident of history; it is the design: **the walker admits a scope it
cannot satisfy.**

**1.3 `fleet-mirror-pk` is a scope for a disk that is gone.** `docs/knowledge/zk-scopes-extra.tsv`
still lists `fleet-mirror-pk → /mnt/wdhdd/fleet-mirror/pk`. `/mnt/wdhdd` exists and
is **empty**. The scope is therefore skipped at generation time (`SKIP extra (root
missing)`), so it never appears in `scopes.tsv` — but its DoD row
`ZDOD-M-20260915T235544Z-2098922-22664` in `docs/dod/WORK.tsv` requires
`grep -qP '^fleet-mirror-pk\t' _logs/zk/scopes.tsv` and can therefore **never pass**.
Register row `D-09` in `docs/reinit/REGISTER.tsv` already recorded the mirror as
"26 dirs under fleet-mirror/pk/zexp1, EMPTY=26 (100%) … a shell that reads as a backup".

**1.4 The derived-output leak is fixed, but the *class* of leak is live.**
`efus3` commit `fd5fa4a83` ("zfts: prune graphify-out/ from the walk (17701 derived
graph files had entered the zk index)") added `graphify-out` to `PRUNE_DIRS` at
`zfts.py:43`. I measured the live index:

```
docs with path LIKE '%/graphify-out/%'      -> 0
docs with path LIKE '%/node_modules/%'       -> 0
docs with path LIKE '%/dist/%'               -> 0
docs with path LIKE '%/ssot_engine_%/dc/%'   -> 0
docs with path LIKE '%/tool-results/%'       -> 0
docs with path LIKE '%fvm/versions/%'        -> 825   <- the Flutter SDK, still there
docs with path LIKE '%/_logs/%'              -> 7357  <- 5.46% of the index
```

The `fd5fa4a83` leak is genuinely closed. But `_logs/` — **cron output** — is 5.46%
of the index and is not pruned by `zfts.py` (only by `zk_embed.py`'s `POP_FILTER`,
which applies to the *embedding* population, not the FTS index). The index therefore
still eats its own machine's logs, which grow every cron tick, which is what makes
the index change, which is what triggers rebuilds.

**1.5 One fifth of the index is the same files twice.**

```
total docs 134,812 · distinct repo-relative paths 108,187 · duplicate copies 26,625 (19.7%)
top duplicated: tsconfig.json ×4, package.json ×4, tsconfig.base.json ×4, README.md ×4
```

The bulk of the duplication is one scope: `efus3-demoguest-merge` contributes 24,619
docs of which **24,610 (99.96%) have an identical repo-relative path in the `efus3`
scope** — 9 unique documents. It is a work branch of the same repo
(`feat/demoguest-merge`, 11 GB), not a separate estate. It is 18% of the index and,
per `graphify.log`, up to **1,626 s** of CPU on every graphify pass.

### 2. The vector store (`vec.sqlite`) — a dead dependency wearing 3.9 GB

```
chunks 737,128 · distinct paths 36,145 · model bge-m3 · dims 1024
vmeta: built_at 1791980706 (2026-10-14) n_chunks 737128 n_docs 36145
```

The store is keyed off `index.sqlite` and embedded via HTTP at
`ZK_EMBED_URL=http://127.0.0.1:18095` (`zk_embed.py:54`), which is an SSH tunnel to
**pk31:8095** (`bin/zk_tunnel.sh:11`). The tunnel is dead and the target host is gone:

```
$ getent hosts pk31.local            -> (nothing), rc=2
$ ss -ltn | grep 18095               -> (nothing)
$ curl http://127.0.0.1:18095/v1/models -> curl: (7) Failed to connect
$ tailscale status | grep pk31
  100.114.44.10  pk31  ... active; relay "blr"; offline, last seen 4d ago
$ tail _logs/zk/tunnel.log
  ssh: Could not resolve hostname pk31.local: Name or service not known
  ZK TUNNEL: RED (pk31 unreachable)
```

Consequences, both fatal:

* **The layer cannot be rebuilt.** `zk embed` needs the server.
* **The layer cannot be used.** `cmd_vsearch` embeds the *query* through the same
  server (`zk_embed.py:488`), so `zk vsearch` is dead even though 737,128 vectors
  sit on disk. Restoring `vec.sqlite` from any backup would restore 3.9 GB that
  cannot answer a single query.

The DoD row for this unit is literally unmeetable on pk33 today
(`docs/dod/WORK.tsv`, `ZDOD-M-20260916T000019Z-2173664-20048`):
`curl -sf http://127.0.0.1:18095/v1/models` fails as the *first* conjunct.

Cost history for the record (`_logs/zk/embed-cron.log`, `embed-full.log`):

```
incremental: docs=35919 chunks=737128 new=34 changed=14 removed=0 secs=125.1
full:        docs=15340 chunks=389206 new=6161 changed=8 removed=1 secs=4326.1
```

The "125.1 s rebuild" quoted in audit #3 is the **hash-only** path: only 48 documents
actually changed. The honest cold cost is **4,326 s (72 min)** for 389,206 chunks
over the pk31 tunnel. And `embed-full.log` also contains a real crash —
`NameError: name 'os' is not defined` at `zk_embed.py:244` — a mid-run failure
that destroyed a 72-minute pass (the missing `import os` was added later, in
`fc0fc7a`).

**There is a second, offline embedding path on this box, unused.** `zexp1/zvec1` runs
`fastembed` / `BAAI/bge-small-en-v1.5` (384-dim ONNX) **on CPU, with the model
cached locally** (`zvec1/data/models/models--qdrant--bge-small-en-v1.5-onnx-q/snapshots/`
contains `model_optimized.onnx`, `tokenizer.json`, `config.json` — 65 MB, complete).
Its full build cost is logged:

```
ZVEC-INDEX: files=18330 chunks=140808 embedded=126468 deleted=0 in 24225.2s   (= 5.8 chunks/s)
```

So: remote bge-m3 ≈ 90 chunks/s (but the host is gone); local bge-small ≈ 5.8 chunks/s
(and always available). **15× slower, but the dependency risk is zero.**

### 3. The code graphs (`graphify-out/`)

```
53 graphify-out dirs under /home/pk33        (not 43 — the audit's count is stale)
7,705 MB total
  6,189 MB in 99 dated snapshot dirs (graphify-out/2026-09-16 … 2026-10-30)  <- 80%
    657 MB in graphify-out/cache/  (AST cache: efus3 145 M, fvm 134 M, demoguest 97 M …)
    ~859 MB live graph.json + GRAPH_REPORT.md + .graphify_labels.json
48 of the 53 sit under a current scope root; 5 are orphans:
  /home/pk33/fvm/versions/3.24.5            (Flutter SDK, 230 MB)
  /home/pk33/trash/{zt331-not-worthy, x001-trash-copy/x0-pk-only, .trash-20261009-072454/x0-pk-only}
  /home/pk33/pk33backupsv1/pk33-cleanup
```

`graphify` is v0.9.27 (`/usr/local/bin/graphify` → `/opt/pipx/bin/graphify`, installed
2026-07-26). Each `graphify update` writes a dated snapshot directory *as well as*
the live `graph.json` — 99 snapshots for 53 roots, none of which anything reads.

**3.1 The RED is structural, not incidental.** `bin/zk_graph.sh --check` requires a
`graph.json` younger than 48 h for **every repo scope**. Measured against the
current (pruned) scope list:

```
scope rows 53 · repo scopes (have .git) 45 · non-repo scopes 8 (pk33-cleanup + 7× claude-memory)
repo scopes with graph.json <48h = 0      -> the check gate reads ok=0 red=45
```

The last log lines (`_logs/zk/graph-cron.log`) read `ok=4 red=45` — that was the
pre-prune 49-repo population on 2026-10-31T21:03Z. `graphify.log` contains
**30,672 RED lines** against 31,165 per-scope records.

**3.2 The runtime claim, measured precisely.** 1,730 `ZK GRAPH:` summary lines in
`graph-cron.log`:

```
all-time mean 348.7 s   (older eras were green: 189 runs at ok=15 red=0)
last 50 runs  693.6 s
last 20 runs  748.8 s     <- the "738 s" figure in audit #3 is the RECENT mean, and is right
last 60 per-scope records sum to 774 s, 60/60 RED
```

**3.3 Why 738 s cannot fit a 10-minute cron, measured.** Slowest scopes ever logged
(`graphify.log`):

```
efus3-demoguest-merge  1626 s      efus3  1616 s      3.24.5 (Flutter SDK)  311 s
zcontent1               253 s      zsm1-tzerp  152 s   rest of the 45       < 80 s each
```

Two scopes alone are **54 minutes** of CPU. A 10-minute cron cannot contain that, so
it overlapped itself perpetually. That is the whole of the "738 s" story: not a slow
walk, a **scheduling impossibility** produced by two oversized scopes.

**3.4 Who reads a code graph?** `bin/zk explain` reads `$root/graphify-out/graph.json`
(`bin/zk:256-263`). Nothing else in the estate reads `graph.json`, `GRAPH_REPORT.md`
or `cache/`. `zk_graph.sh --check` reads only the file's mtime.

### 4. The scope list (`scopes.tsv`)

Generated by `bin/zk scopes` from three sources:

| source | rows | note |
|---|---|---|
| `docs/registers/clones.tsv` (`host==pk33`) | 55 | regenerated by `bin/recon_fleet.sh` every beat; 1 root missing; **includes the Flutter SDK and 2 trash dirs** |
| `~/.claude/projects/*/memory` glob, hardcoded in `bin/zk:113` | 7 | 5 empty dirs were skipped by the Oct-31 prune; 7 dirs → 7 rows under **one** scope name |
| `docs/knowledge/zk-scopes-extra.tsv` | 0 (1 dead) | `fleet-mirror-pk` → `/mnt/wdhdd/fleet-mirror/pk`, root missing |

```
scopes.tsv          : 53 rows / 47 unique names  (claude-memory ×7)
scopes.tsv.indexed  : 61 rows / 50 unique names  (claude-memory ×12; + fvm; + 2 trash)
```

The `.indexed` file is the population the live index was built from. Its diff against
the current list is the entire rot story in five lines: one Flutter SDK, two trash
directories, and four extra `claude-memory` rows (scratchpads, `/tmp`, bare
`/home/pk33`). The Oct-31 fix (`c3fa3e7`) applied the prune **at the scope layer**,
which is correct — `clones.tsv` is machine-regenerated and a hand-edit there is undone
within the hour (the commit message says exactly this).

The 12-vs-7 `claude-memory` count is *cosmetic, not destructive* — `zfts.py:78`
does `scopes.setdefault(scope, []).append(root)`, so same-named rows merge by
appending roots. The ledger entry at `efus3/_logs/rework-ledger/2026-10-31.md`
("Correction to audit #4's open question 2") reached the same conclusion.

**But there is no scope *policy*.** There is a scope *generator*. Admission is
implicit: "every pk33 clone row, plus whatever `bin/zk` hardcodes, plus whatever a
hand-maintained TSV lists." Nothing states what *earns* a scope, and nothing rejects
a scope that cannot be satisfied (§1.2), cannot be fetched (`fleet-mirror-pk`), or is
a duplicate of another scope (`efus3-demoguest-merge`, §1.5).

### 5. The context packs (`zprotocols/ctx/`) — the one healthy component

25 packs (`CONTEXT-v1.0.0.md` … `v1.0.24.md`), `LATEST.md`, and 25 `.sha256`
sidecars. Compiled by `efus3/scripts/zctx.sh compile`; verified by `zctx.sh verify`
with five checks: sha integrity, LATEST resolution, **content** freshness (sha256 of
every compile source — "mtime is never consulted"), a spine digest, a 3,000-token
budget, and a secret-shape scan.

This is the best-engineered artefact in the layer:

* every fact is computed at compile time from a named source; nothing is hand-typed;
* every pack carries its own seal;
* staleness is a *content* question, answerable in milliseconds;
* the refusal rule is explicit — "Nothing else writes into `ctx/`", because a stray
  `CONTEXT-v*.md` would be mis-parsed as a pack version (`protocols/ZP-008-context-pack.md:22-24`);
* it is **git-tracked**, and the protocol `ZP-008` is `LIVE` in
  `zprotocols/registry/PROTOCOLS.tsv`.

Two real defects, both in the *pack history*, not the design:

**5.1 Nine packs exist only as staged git index entries.**

```
$ git -C zexp1/zprotocols status --porcelain ctx/
A  ctx/CONTEXT-v1.0.16.md   ... through ...
A  ctx/CONTEXT-v1.0.24.md
A  ctx/CONTEXT-v1.0.24.md.sha256
M  ctx/LATEST.md
$ git -C zexp1/zprotocols ls-tree --name-only HEAD ctx/ | grep -oE 'v1\.0\.[0-9]+' | sort -V | tail -1
v1.0.15
```

`HEAD == github/main == 78d7850`, so **v1.0.16 → v1.0.24 (9 packs + 9 seals + LATEST.md)
are in no commit and on no remote.** One `git reset` and they are gone. The
`zctx-refresh` cron stages them and never commits them (audit #3 §4.2, verified:
`git log` shows the last real commit is `7fdf32a "zctx: packs v1.0.14/v1.0.15"`).

**5.2 The pack's NOTES INDEX section is empty because a compile source is missing.**
`zctx.sh _sources()` includes `$BASE/factory6000/docs/assessment`; `/home/pk33/zexp1/factory6000`
does not exist on pk33. The pack still verifies GREEN — the sources-list digest is
computed over *names*, and the missing dir contributes no names — but the section is
silently empty. (I recomputed all four live source digests by hand; they match the
pack's `## SOURCES` block exactly, so v1.0.24 is genuinely GREEN today.)

**5.3 The pack is consumed by convention, not by a machine contract.** Consumers:
`efus3/skills/zctx/SKILL.md` ("agents read LATEST at start"), `ZP-008`, and
`bin/zk_report.sh:17` (`zctx.sh verify | tail -1`). No cron compiles it (the
`zctx-refresh` cron is not in the live crontab and never was in the registry rows
I checked).

### 6. The report surface (`zk_report.sh` → `seats/zdash22v1-knowledge/REPORTS.md`)

`bin/zk_report.sh` appends an hourly block — `bin/zk status`, `zk_graph.sh --check`,
the last embed line, a process count, `pcl_selfcron.sh verify`, `zctx.sh verify`, and
the seat's own DoD row counts — to `REPORTS.md`, then:

```bash
git add "$F" docs/dod/WORK.tsv && git commit -q -m "v1-knowledge: hourly computed report (cron)" \
  && git pull -q --rebase --autostash origin main >/dev/null 2>&1; git push -q origin main
```

`REPORTS.md` is now **213,607 bytes** of hourly blocks. Every block in the last
five snapshots reads RED on every row:

```
ZK STATUS: STALE docs=134812 scopes=50 age_h=0.04
ZK VEC:   STALE chunks=737128 docs=36145 fts_docs_md=35925 coverage=36145/35925 100.6% age_h=416.6
ZK GRAPH CHECK: ok=4 red=45
ZCTX-VERIFY: GREEN (v1.0.24)
```

Two things worth naming:

* **`ZK VEC` coverage reads 100.6% — a metric that can exceed 100%.** The numerator
  counts distinct paths in `chunks` (36,145); the denominator counts md docs in the
  FTS index *after* `POP_FILTER` (35,925). Numerator and denominator have different
  populations. A coverage metric that reports 100.6% is not measuring coverage.
* **`git pull --rebase` on a repo that is 82 commits ahead of origin.** `zdash222`
  is `main...origin/main [ahead 82, behind 2]`. An hourly `pull --rebase --autostash`
  on top of that is a rewrite hazard, not a sync. It also means every hourly report
  commit is a candidate for a force-push problem — and `origin/main` still points at
  `39497bc`, the *pre-rebase* main (the ledger's row 3.1/3.2: `main` was frozen for
  24 days while 81 commits landed on a detached HEAD).

### 7. The six crons, the pause, and the second index

`efus3/_logs/agent-crons/registry.tsv` (all six `DISABLED`, and zero matching lines
in the live crontab):

| name | schedule | command | state |
|---|---|---|---|
| `zk-graph` | `41 */6 * * *` | `bin/zk_graph.sh` | DISABLED |
| `zk-report` | `3 * * * *` | `bin/zk_report.sh` | DISABLED |
| `zk-embed-tunnel` | `*/5 * * * *` | `bin/zk_tunnel.sh` | DISABLED |
| `zk-index-zdash222` | `4,14,24,34,44,54 * * * *` | `zk index --scope zdash222` | DISABLED |
| `zk-index` | `*/10 * * * *` | `zk index --if-stale` + `zk_graph.sh --missing` | DISABLED |
| `zk-embed` | `23 * * * *` | `zk embed --batch 128` | DISABLED |

The pause is durable by mechanism, not by comment: `pcl_selfcron.sh disable` flips
the registry state, and `pcl_selfcron.sh heal` skips `DISABLED` rows
(`efus3/_logs/rework-ledger/2026-10-31.md`, "Validated: the pause will hold").
Note the recorded trap: `# pcl-selfcron:zk-index` is a **substring** of
`# pcl-selfcron:zk-index-zdash222`, so the disable order matters — `zk-index-zdash222`
must go first.

**Not paused, and not part of the zk family:** the crontab still runs
`11 4 * * * zfts.sh index` — efus3's own builtin-scope FTS index
(`efus3/_logs/zfts/index.sqlite`, 1,281 docs, built 2026-10-09). It is a separate,
healthy, small surface. Any rework of the intelligence layer must decide whether it
is kept, folded in, or allowed to drift alongside.

### 8. The knowledge *sources* (not derived — do not touch)

`efus3/zfable/` is 94 MB / 2,608 files of doctrine, family registers, charters and
ADRs. It is **source material**, indexed *through* the `efus3` scope, and is the
substance of the knowledge layer. The `zfts.py` builtin scope `registers` points at
`zfable/family`. The data spine (`ssot_engine_efus3/dc`, 118 GB, `spine.sqlite`
+ `intake.ndjson`) is explicitly *excluded* from indexing by `SPINE_RE`
(`zfts.py:48`) — it is data, not knowledge. Neither belongs in the throw-away list.

---

## RECREATION CASE (per component)

**The governing fact for every row below:** there is **no backup of this layer
anywhere**. I checked `docs/protocols/BACKUP-LEGS.tsv` (no `zk`/`graph`/`index`/`vec`
rows), the only rclone remote (`b2-remote:` — 10 buckets: `consolidator-v1`,
`eduz-backups-pk1`, `zconsolidator-v1`, `zcontent-books`, `zcontent-zgategoogleonly`,
`zdev-backups1-terminal`, `zncert2`, `zseeded`, `zsm-ncert`, `zsm1work` — none contains
KG artefacts), and a `find / -xdev` for `index.sqlite*` outside the live path, which
returned **only the live files**. There is no R2 remote at all. `zsm1`'s README
mentions R2 for big derived artefacts, but no R2 configuration exists on this box.

So "restore-from-backup" is not a cheaper option for any component — for most of
them it is not *an* option. That does not weaken the case for recreation; it
strengthens it, because the alternative to recreation is not "restore", it is
"leave it exactly as it is, broken".

### 1. FTS index (`index.sqlite`, 2.2 GB, 134,812 docs)

| | |
|---|---|
| **Source of truth** | the repo working trees themselves. `docs/registers/clones.tsv` + `docs/knowledge/zk-scopes-extra.tsv` (git-tracked) describe *which* trees; the trees are the *content*. |
| **Rebuild from source?** | **Yes — and it is cheap.** Measured on this box: the walk of 134,063 files / 1,794 MB of text takes **0.6 s** (`zfts.files()` enumeration). FTS5 insert rate measured on a 4,000-doc sample: **4,959 docs/s, 35.9 MB/s** → a full rebuild is a **~30–60 s** job, not a 738 s one. Corroborated in the field: the registry's own claim that a single-scope refresh of `zdash222` (7,365 files) takes "1.4 s" matches my 0.8 s for 4,000 of those same files. |
| **Restore from backup?** | No backup exists. Even if one did, it would restore a *stale, rot-bearing* index (the live one still carries `3.24.5` and `zt331-not-worthy`), and a restored index would be a fixed snapshot of a tree that has since changed. |
| **Verdict** | **REBUILD.** Cheapest by an order of magnitude, and the only option that can actually fix the rot. |

### 2. Vector store (`vec.sqlite`, 3.9 GB, 737,128 chunks)

| | |
|---|---|
| **Source of truth** | the md documents in the trees + **an embedding model**. The model is the real dependency, and it is not on pk33. |
| **Rebuild from source?** | **Not today.** The remote bge-m3 server is on pk31 (offline 4 days). The only offline path is `zvec1`'s local bge-small-en-v1.5 (model cached, 384-dim) at **5.8 chunks/s** → 737,128 chunks ≈ **35 h**; 140,808 chunks ≈ 6.7 h. The two models have different dimensionality (1024 vs 384), so the stores are not interchangeable and a "restore" cannot be migrated into a local model either. |
| **Restore from backup?** | No backup exists; and restoring would be **useless** — `cmd_vsearch` embeds the *query* through the dead server, so the vectors could not answer a single query. |
| **Verdict** | **REBUILD — but only after the model question is answered, and do not rebuild it until then.** Three options, in order of preference: (a) **drop the embedding tier entirely** and ship FTS + ripgrep only (the `zfts.sh` charter explicitly says "zfts does not UNDERSTAND; it RANKS… stop 'LLM reading markdowns which has proved disastrous'" — the FTS tier already answers "where is X / what says Y" over 134,812 docs); (b) **local bge-small** via zvec1's cached model, as a nightly batch with a declared budget; (c) remote bge-m3, **only if pk31 returns** — and then the tunnel cron returns with it. Holding 3.9 GB of unqueryable vectors is the worst of the three. |

### 3. Code graphs (`graphify-out/`, 53 dirs, 7.7 GB)

| | |
|---|---|
| **Source of truth** | the repo source tree at a known commit. `graphify` itself is on the box (`/usr/local/bin/graphify` v0.9.27, pipx). The graphs are pure derivations — the repo is the truth. |
| **Rebuild from source?** | **Yes, but only for a scope that deserves one.** Cost is dominated by two repos (efus3 1,616 s, efus3-demoguest-merge 1,626 s). A scoped rebuild of ~6 code repos, one at a time, with a hard per-repo timeout, is ~10–30 min nightly — affordable. A rebuild of all 45 repo scopes is ~60 min of CPU and is exactly the workload that overheated the box. |
| **Restore from backup?** | No backup exists. Restoring graphs would restore graphs of *old commits* — a code graph whose HEAD does not match the repo is worse than no graph, because `zk explain` would answer questions about code that no longer exists. |
| **Verdict** | **REBUILD — but rebuild less.** The honest question is not "how do I restore the graphs" but "does anything need the graphs at all". The only reader is `zk explain`, which has no recorded consumer, and whose 45 scopes are 45/45 RED today. Either retire the A-16 row (it is already `BLOCKED` with a PARK note in `WORK.tsv`) or rebuild a *small*, explicit set of code scopes with a per-repo timeout. |

### 4. Scope list (`scopes.tsv`, `clones.tsv`, `zk-scopes-extra.tsv`)

| | |
|---|---|
| **Source of truth** | git. `docs/registers/clones.tsv` and `docs/knowledge/zk-scopes-extra.tsv` are tracked in `zdash222` (`git ls-files` confirms both). |
| **Rebuild from source?** | **Yes** — trivially, `bin/zk scopes`. But generation is the problem, not the artefact: `clones.tsv` is machine-regenerated every recon beat, so scope membership currently *drifts*; and the generator has no admission test. |
| **Restore from backup?** | Meaningless — it is 2.6 KB of generated text. |
| **Verdict** | **REBUILD, but change the *kind* of artefact.** Scope membership must become a **tracked, reviewed, hand-authored register** with admission rules and an existence check at build time — not a cron-generated view of a machine-regenerated roster. (The Oct-31 fix already moved the prune to the right layer; it did not create a policy.) |

### 5. Context packs (`zprotocols/ctx/`, 25 packs) + `zctx.sh`

| | |
|---|---|
| **Source of truth** | the compile sources named in `zctx.sh _sources()`, all of which are git-tracked in `efus3` (LANES.tsv, registry.tsv, zversionkeeper.sh, zclaudemanager.sh) plus the `dc` spine's axis rows. |
| **Rebuild from source?** | **Yes — by design.** `bash scripts/zctx.sh compile` regenerates every section from computed sources, bumps the semver, writes the sha seal. That is the whole point of the pack. A fresh pack would compile in seconds. |
| **Restore from backup?** | Only v1.0.0–v1.0.15 are actually backed up (in `HEAD`/`github/main`); v1.0.16–v1.0.24 are **staged only** and on no remote. Restoring v1.0.15 would be a *regression* — the pack's own freshness digest would go stale and its version number would go backwards. |
| **Verdict** | **REBUILD (compile forward).** But first **commit the 9 staged packs** — or, better, compile a fresh pack and commit that, so the repository history stops at v1.0.15 and the live pack is a clean, single, committed artefact. This is the one component where the correct action is nearly "do nothing but commit it". |

### 6. The report surface (`zk_report.sh` → `REPORTS.md`) and the CLI surface

| | |
|---|---|
| **Source of truth** | git. All seven `bin/zk*` scripts and `efus3/scripts/zfts.py` are tracked. |
| **Rebuild from source?** | N/A — the code is intact on disk and on `github.com/zdash22x/zdash222` (`git cat-file -e origin/main:bin/zk` → PRESENT). |
| **Restore from backup?** | N/A. |
| **Verdict** | **KEEP AND EDIT, do not rebuild.** Two edits are needed, and neither is a rebuild: (a) `zk_report.sh` must stop doing `git pull --rebase` on a repo 82 commits ahead of its origin; (b) `zk_report.sh` must stop running `zk_graph.sh --check` and `zk status` when both are structurally RED — a report that is RED every hour for 213 KB teaches nothing. |

### 7. The second index (`efus3/_logs/zfts/index.sqlite`, 1,281 docs) and zvec1

| | |
|---|---|
| **Source of truth** | efus3's own tree; the scope config is `zfts.py`'s builtin `SCOPES` dict. |
| **Rebuild from source?** | Yes — `zfts.sh index`, and its cron is still live. |
| **Verdict** | **KEEP AS IS.** It is small, healthy, and independent of the paused family. But decide explicitly: two FTS engines with two scope policies over the same estate is a design smell that will be re-litigated. The register already said it (`A-15`: *"decide whether zvec1 or zfts is THE retrieval surface - do not wire both"*). |
| **zvec1** | **HOLD as the embedding fallback.** Its model is on disk and complete; its build cost is known (24,225 s for 140,808 chunks); its 514 MB store is 3 months stale; its smoke test is 7/10. It is the *only* offline embedding path, which makes it an asset precisely because nothing is currently using it. |

### 8. The summary of the case

| Component | Restore-from-backup | Rebuild-from-source | **Call** |
|---|---|---|---|
| FTS index | no backup exists; stale & rot-bearing | 0.6 s walk + ~60 s build, measured | **REBUILD** |
| Vector store | no backup; unqueryable without the dead server | blocked on pk31; 35 h locally | **REBUILD, later — or drop the tier** |
| Code graphs | no backup; graphs of dead commits | 60 min for 45 scopes, ~10–30 min for 6 | **REBUILD less, or retire** |
| Scope list | 2.6 KB, meaningless | seconds — but it must become a *register* | **REBUILD as policy** |
| Context packs | only v1.0.15 is backed up | one `zctx.sh compile` | **COMPILE FORWARD + COMMIT** |
| CLI surface | intact in git and on disk | — | **KEEP AND EDIT** |
| Second index / zvec1 | — | — | **KEEP / HOLD as fallback** |

**The one thing that is not derivable is not a document: it is the *policy*.** Every
artefact here can be regenerated from files that exist. What cannot be regenerated
is the answer to "what belongs in the knowledge layer" — and the absence of that
answer is why the layer rotted, overheated the box, and is now paused. That is the
argument for recreation rather than restoration: restoration returns the artefacts
*and the rot that produced them*; recreation has to write the policy, and the policy
is the only thing that was actually missing.

---

## TARGET ARCHITECTURE

Name: **ZK2**. One rule underneath all of it: *every artefact is derived from a
declared, existing, tracked source; every derived artefact is denied as an input to
any other derived artefact; every build is bounded, idempotent and content-verified.*

### A. Scope policy — what earns a scope

A scope is admitted **iff all six tests pass**. The tests run at build time and a
failure is a **build error that names the row**, never a scope with zero documents.

| # | test | rationale (the failure it closes) |
|---|---|---|
| A1 | The scope is a row in a **git-tracked register** `docs/registers/scopes.tsv` (hand-authored, reviewed, PR'd) — **not** auto-derived from `clones.tsv`. | Scope membership must not drift under a cron. `clones.tsv` is rewritten every recon beat; `zk-scopes-extra.tsv` is already hand-maintained and should not be the only one. |
| A2 | The root **exists on disk at build time**; a missing root is an error naming the row. | Closes `pk33-cleanup` (root gone → permanent empty scope → permanent RED) and `fleet-mirror-pk` (root gone → a DoD that can never pass). |
| A3 | The root is a git work tree (or a declared offline mirror) — i.e. it has a source of truth. | Closes the `twdpt1` / `twdpt1-local` / `efus3` / `efus3-demoguest-merge` family of sibling copies. |
| A4 | The root is **not nested inside** another admitted scope. | Keeps the existing `bin/zk:84` check, promoted from a print to a gate. |
| A5 | The scope declares a `role` ∈ `{code, knowledge, memory}` selecting the walker policy. | Stops one walker from having to guess: `.claude/projects/*/memory` is not a clone; `zfable/family` is not a repo. |
| A6 | **Duplicate test**: no admitted scope may share more than 5% of its indexed repo-relative paths with another admitted scope. Measured at build time; violation = error. | Closes `efus3-demoguest-merge` (99.96% duplicate of `efus3`, 24,619 docs, up to 1,626 s of CPU per graph pass). |

Plus, applied *inside* the walker, a **denylist of derived outputs** rather than a
patch on one directory name (the `fd5fa4a83` lesson generalised):

```
DENY_DIRS  = graphify-out, _logs, out, dist, build, cache, node_modules, .venv,
             venv, __pycache__, tool-results, shell-snapshots, .cache
DENY_PATH  = ssot_engine_*/(dc|rc|sym)      # the data spine is data, not knowledge
DENY_FILE  = *.sqlite*, *.db, *.ndjson, *.env, *.env.*, *.pem, *.key, > 2 MB
```

**and a hard invariant check after every build** (see §D, check B): *zero* indexed
paths may match the denylist. The `fd5fa4a83` fix is one line in a list; the
invariant is a gate that fails the build. `_logs/` must be on that list — it is 5.46%
of the live index today and it is cron output, i.e. exactly the "derived output
feeding its own index" pattern in its mildest form.

### B. Schema — four named artefacts, none of them inside `_logs/`

The audit's own point stands: *"a vector store living under `_logs/` is a structural
smell — nothing distinguishes 'important derived data' from 'rotten logs' in that
path."* New home: `zdash222/kg/` (git-ignored, as `_logs/zk/*.sqlite*` is today).

```
kg/scopes.tsv            tracked register (A1) + generated admitted list, diffed in CI
kg/index.sqlite          FTS: docs(path UNINDEXED, scope UNINDEXED, kind UNINDEXED,
                                   title, body, tokenize='porter unicode61')
                                meta(key TEXT PRIMARY KEY, value TEXT)
kg/vec.sqlite            chunks(path, scope, "ord", sha, text) + vec0 vec_chunks + vmeta
kg/graphs/<scope>/graph.json   + kg/graph-manifest.tsv
```

**Meta keys — the change that makes freshness a content question:**

```
built_at                 ISO-8601 UTC
n_docs                   integer
scopes_sha256            sha256 of the EXACT kg/scopes.tsv that produced this index
source_digest            sha256 over (scope, repo HEAD sha, file count, byte count)
scope:<name>             doc count for that scope
model / dims             (vec.sqlite vmeta only) — so a model change is detectable
```

Today freshness is decided by `scopes.tsv` vs `scopes.tsv.indexed` (a *population*
compare) plus a 48-hour age heuristic. Nothing asks "has the source changed?" — so a
three-week-stale index and a fresh one look the same. `source_digest` answers it in
one comparison.

### C. Bounded runtime — the scheduling law

**No cron may exceed its slot, and a job that exceeds its budget is killed and
reported RED. It does not overlap itself.** (The current failure was exactly this:
738 s of work in a 600 s slot.)

| tier | work | measured cost | budget | schedule |
|---|---|---|---|---|
| FTS full rebuild | walk + insert 134k docs | 0.6 s walk; ~30–60 s insert (4,959 docs/s, 35.9 MB/s measured) | **≤ 5 min** | on demand / nightly |
| FTS incremental | per-scope refresh | 1.5 s for `zdash222`'s 7,365 files | **≤ 30 s/scope** | hourly, ≤ 6 scopes/tick |
| Code graph | `graphify update <root>` | 80–1,626 s per repo | **≤ 300 s/repo, hard `timeout`** | nightly, explicit code scopes only |
| Embed (local) | bge-small on CPU | 5.8 chunks/s → 140,808 chunks = 6.7 h | **≤ 8 h, one batch** | nightly, opt-in |
| Embed (remote) | bge-m3 over the pk31 tunnel | 90 chunks/s → 737,128 chunks ≈ 2.3 h | ≤ 3 h | nightly, **only if pk31 is back** |
| Report | read-only over the tiers above | < 10 s | ≤ 1 min | hourly |

Two structural consequences:

* **The 10-minute cron disappears.** It existed to serve a 738 s workload. With a
  60 s rebuild and a content-digest freshness check, an hourly schedule is ample,
  and a 10-minute schedule is simply a schedule that cannot be honoured.
* **The thermal guard becomes structural, not a policy note.** Audit #3 §2.7 asked
  for "a cron that checks `sensors` and pauses graphify above 85 °C". Better: the
  per-repo `timeout` plus a bounded scope list means the worst case is 6 × 300 s of
  one core — bounded by construction, no sensor needed. A sensor check remains a
  cheap belt-and-braces, but the *primary* control is the budget.

### D. Verification gate — `bin/zk2 verify`

One command, every check exits 0/1, every check prints its population, and the
whole thing is **GREEN only if every check passes**. No check may be waived by
excluding a scope; that is the "perpetual RED" failure mode.

| check | assertion | what it closes |
|---|---|---|
| **A — scope admission** | every register row's root exists; no nested roots; no pair above the 5% duplicate threshold; **every admitted scope has ≥ 1 doc**; **every scope in the index is in the register** (no orphans). | `pk33-cleanup` (empty scope), `fleet-mirror-pk` (dead root), `3.24.5` / `zt331-not-worthy` (rot still in the index), `efus3-demoguest-merge` (duplicate) |
| **B — no derived output** | `SELECT COUNT(*) FROM docs WHERE path LIKE '%/graphify-out/%' OR path LIKE '%/_logs/%' OR path LIKE '%/node_modules/%' OR path LIKE '%/dist/%' OR path LIKE '%/ssot_engine_%/dc/%' ...` **== 0** | the `fd5fa4a83` class, generalised; today `_logs/` alone is 7,357 docs |
| **C — freshness** | `meta.source_digest` matches the live sources, OR age < budget with the reason printed | the 3-week-stale index that still reads `GREEN` on a population compare |
| **D — coverage** | every admitted scope ≥ 1 doc; `n_docs` == `SUM(scope:*)` | silent partial builds |
| **E — graph freshness** *(optional tier)* | every code scope's `graph-manifest.tsv` sha matches the file on disk, and mtime < 48 h | the 45/45 RED |
| **F — embed consistency** *(optional tier)* | chunked docs / md docs ≥ 95%, **and** the serving model's id and dims match `vmeta` | this is the check that would have caught pk31 going dead *silently*, while the store still read "coverage 100.6%" |
| **G — the honest-RED rule** | a scope may not be admitted unless it can be satisfied; a RED must name the scope and the reason | the whole 45/48 RED forever |

The gate's output must be *readable by a human in one line* and *machine-checkable*.
The current `ZK STATUS: STALE docs=134812 scopes=50 age_h=0.04` is a RED that says
nothing about *why*; that is why it sat unread for a month.

### E. Definition of Done

> **From a clean checkout of the code repos and one command, the intelligence layer
> exists, is correct, and says so.**

```
cd /home/pk33/zdash22x/zdash222
bin/zk2 build --all      # scope admission → FTS → (graphs) → (embed) → report
bin/zk2 verify           # exit 0 iff A–G all pass
```

**Stated times (from the measurements above):**

| tier | end-to-end from a clean checkout | rebuild-on-change | notes |
|---|---|---|---|
| **FTS only** (recommended) | **≤ 10 min** (walk 0.6 s + build ~60 s + verify < 10 s) | ≤ 5 min full / ≤ 30 s per scope | the tier that actually answers "where is X / what says Y" |
| **+ code graphs** (explicit scope list) | **≤ 60 min** | ≤ 300 s per repo, nightly | only for the repos pk actually asks about |
| **+ embeddings** (local bge-small) | **≤ 8 h** | overnight batch | only if the tier is wanted at all |
| **+ embeddings** (remote bge-m3) | **≤ 3 h** | overnight batch | **only if pk31 returns** |

**The DoD is not "the index exists". It is "the index can be thrown away and rebuilt
from a clean checkout in the stated time, and the verification gate is GREEN after
the rebuild."** A layer that cannot survive that test is not an index; it is a
treasure.

---

## BUILD ORDER + VERIFICATION GATE

Sequenced so that each step is verifiable before the next begins, and so that
nothing irreversible happens before its replacement exists. **No step re-enables any
currently-disabled cron.** The re-enable is step 8, and it is one
`pcl_selfcron.sh enable` per cron, each with a named budget.

| step | action | gate before proceeding | rollback |
|---|---|---|---|
| **0** | **Commit what is one commit from being lost.** Commit the 9 staged ctx packs (or compile a fresh pack and commit that). | `git -C zexp1/zprotocols status --porcelain ctx/` is empty; the pack verifies GREEN. | `git reset` to the pre-commit index (files are on disk either way). |
| **1** | **Decide the scope policy, write it down, and make it tracked.** Author `docs/registers/scopes.tsv` with the A1–A6 tests. Retire `docs/knowledge/zk-scopes-extra.tsv` (its one row is dead) and stop deriving scopes from `clones.tsv`. | Every row has an explicit role and an existing root; the duplicate report shows `efus3-demoguest-merge` at 99.96% and it is *not* on the list. | the file is new; deleting it restores the old behaviour. |
| **2** | **Write the new index at a new path.** `kg/index.sqlite`, the denylist of derived outputs, the `meta` keys `scopes_sha256` + `source_digest`, and an atomic tmp+rename build. Do **not** touch the live `index.sqlite` yet. | `bin/zk2 build --scope-all` produces an index; **check B returns 0 derived paths** (vs 7,357 `_logs/` docs today); check A returns no orphan scopes. | `rm -rf kg/` — nothing outside it was touched. |
| **3** | **Verify, then compare.** Run `bin/zk2 verify`; then diff the new index's scope set and doc counts against the old one and *explain every difference in writing*. | The only differences are the ones the policy intends (no `3.24.5`, no `zt331-not-worthy`, no `pk33-cleanup`, no `_logs/`, no duplicate scope). | the old index is untouched at `_logs/zk/index.sqlite`. |
| **4** | **Repoint the CLI.** `bin/zk` → `kg/index.sqlite`; `zk_check_paths.sh` → the same. **Do not re-enable any cron.** | `bin/zk search "twdpt1 DPR" --json` returns hits; `bin/zk status` reads GREEN **with a reason**; `zk_check_paths.sh` returns 0 missing. | revert the two path constants. |
| **5** | **Decide the code-graph tier, explicitly.** Either (a) retire the A-16 row (`ZDOD-M-20260916T000338Z-2209392-31610`, already `BLOCKED`/PARK) and delete `zk_graph.sh` from the surface, or (b) author a short explicit code-scope list and rebuild those graphs only, each under a 300 s `timeout`. | Either the `zk explain` verb is gone and nothing references it, **or** `zk_graph.sh --check` reports `ok=N red=0` for N ≤ 8, with each pass bounded. | the graphs are pure derivations; deleting them loses nothing. |
| **6** | **Decide the embedding tier, explicitly.** Drop it, or run it locally against zvec1's cached bge-small model with a declared chunk budget, or hold it for pk31. | Either `zk vsearch` is removed from the surface and its DoD row is closed as "not wanted", **or** `bin/zk2 verify` check F is GREEN and the serving model id is recorded in `vmeta`. | `vec.sqlite` is a file; it can be deleted and rebuilt. |
| **7** | **Fix the report surface.** `zk_report.sh`: drop the `git pull --rebase` (82 commits ahead of origin); report the *reason* for each RED; stop emitting an all-RED block every hour; report only what changed. | The report contains at least one GREEN row or names the specific scope that is RED and why; `git status` is unchanged by the report's commit. | `git revert` the commit; `REPORTS.md` is append-only text. |
| **8** | **Re-enable, one cron at a time, in this order** — (a) `zk-index` at an **hourly** schedule (not 10-min) with the new bounded build; (b) `zk-embed` only if step 6 chose to keep it, nightly; (c) `zk-graph` only if step 5 chose to keep it, nightly; (d) `zk-report` hourly; (e) `zk-embed-tunnel` only if pk31 is back. **Never `zk-index` on a 10-minute schedule with a 738 s workload.** | After each `pcl_selfcron.sh enable`, exactly one new registry row is `ACTIVE` and the others stay `DISABLED`; `pcl_selfcron.sh verify` shows no drift; a thermal sample (`sensors`) stays under 80 °C. | `pcl_selfcron.sh disable <name>` — the registry row survives with its schedule and command, per `efus3/_logs/rework-ledger/2026-10-31.md`. |
| **9** | **Throw away** (next section), then re-verify. | `bin/zk2 verify` still GREEN after the deletions. | see the per-item reversibility in the next section. |

**Verification gate, restated as the single command that closes the whole sequence:**

```
bin/zk2 verify    # exit 0 iff: scope admission A · no derived output B · freshness C
                  #                · coverage D · (graphs E) · (embed F) · honest-RED G
```

**Do not mark this work DONE on "the index exists".** Mark it DONE on: the layer
was destroyed and rebuilt from a clean checkout inside the stated time, and the
gate was GREEN both before the destruction and after the rebuild.

---

## WHAT TO THROW AWAY

Every item is derived, has no reader I could find, and is regenerable or already
dead. Sizes measured today. **This is a recommendation — I have deleted nothing.**

### Tier 1 — noise with a size tag (delete, no regret)

| # | item | size | why it is noise | recoverable? |
|---|---|---|---|---|
| 1.1 | `zdash222/_logs/zk/index.sqlite.prev` | **858 M** | built 2026-09-15T23:46:19Z against a **15-scope generation that no longer exists** (it still carries `zdash222-docs`, `zdash222-seats`, `zdash222-logs`, `tzerp1`). 52,719 docs of a world that was replaced. No reader. | yes — regenerable, but nobody wants it |
| 1.2 | 99 dated `graphify-out/2026-*-*` snapshot dirs | **6,189 M** | `zk explain` reads `graphify-out/graph.json` only; `zk_graph.sh --check` reads an mtime. Not one of these 99 snapshots has a reader. 80% of all graphify bytes. | yes — `graphify update` writes a new one |
| 1.3 | `graphify-out/cache/` (AST cache) | **657 M** | an accelerator for graphs that are being thrown away. efus3 145 M, fvm 134 M, efus3-demoguest-merge 97 M. | yes — rebuilt on the next `graphify update` |
| 1.4 | `graphify-out` under `trash/` × 3 and `fvm/versions/3.24.5` | ~233 M | the Flutter SDK's graph (230 M) was built for the exact scope that pinned a core at 100% and drove the package to 89–100 °C (audit #3 §2.2). The three under `trash/` are graphs of discarded content. | yes — and one of them should never be rebuilt |
| 1.5 | `graphify-out` under `pk33backupsv1/pk33-cleanup` | 1 M | orphan: not under any admitted scope root. | yes |
| 1.6 | `docs/knowledge/zk-scopes-extra.tsv` | 158 B | its one row, `fleet-mirror-pk → /mnt/wdhdd/fleet-mirror/pk`, points at a **removed disk**. Register D-09 already recorded the mirror as "26 dirs, EMPTY=26 (100%) … a shell that reads as a backup". | yes — git-tracked; the file is in the repo |
| 1.7 | `zdash222/_logs/zk/tunnel.log` | 76 K | 100% `ZK TUNNEL: RED (pk31 unreachable)` since pk31 went away. A log of a fact that is now in `tailscale status`. | yes |
| 1.8 | the `3.24.5` and `zt331-not-worthy` **scopes** (825 + 1 docs inside `index.sqlite`) | — | rot the Oct-31 scope prune removed from the *list* but which is still inside the *index*. They vanish on the next rebuild. | yes — the next build |
| 1.9 | the `pk33-cleanup` scope (root `/home/pk33/awgbackups/pk33-cleanup`, 6 docs) | — | the root does not exist. On the next rebuild it becomes 0 docs → a permanent RED scope. | yes — the next build |

**Tier 1 total: ≈ 7.9 GB freed, zero information lost.**

### Tier 2 — stale: true, but still load-bearing (fix, don't delete)

| # | item | finding | action |
|---|---|---|---|
| 2.1 | `zdash222/seats/zdash22v1-knowledge/REPORTS.md` | **213,607 bytes**, every hourly block RED on every row, growing ~1 KB/hour for two months. | Not deleted — it is the seat's receipt. **Rewrite the producer** (step 7) so it emits reasons, not a wall of RED. Then truncate to the last GREEN block plus a pointer. |
| 2.2 | `_logs/zk/index.log`, `index-zdash222.log`, `embed-cron.log`, `graph-cron.log`, `graphify.log` | 1,730 graph summaries, 31,165 per-scope graph records, 30,672 of them RED. | These are the *evidence* for the whole argument in this document. Keep the last 30 days, archive the rest. Do not delete before the ledger entry references them. |
| 2.3 | `vec.sqlite` (3.9 G) | GREEN-looking, 737,128 chunks, and **unqueryable** — pk31 is gone and `vsearch` embeds the query through the same dead server. Its `coverage=36145/35925 100.6%` is a metric with two different populations in numerator and denominator. | **Do not restore it; do not trust it.** Delete it when the tier decision (step 6) is made. If the tier is kept, rebuild against a model that is actually present. |
| 2.4 | `efus3/_logs/zfts/index.sqlite` (12.3 M, 1,281 docs) | healthy, but its cron (`11 4 * * *`) is **still live** while the whole zk family is paused. | Decide explicitly (keep / fold into zk2 / retire). Do not leave it drifting by accident. |
| 2.5 | `zexp1/zvec1/data/zvec.sqlite` (514 M, 140,808 chunks, 3 months stale, smoke 7/10) | The only offline embedding path on the box (model cached, 65 M). | **HOLD as the fallback.** Do not delete. Rebuild it only if the embedding tier is chosen. |

### Tier 3 — lying: artefacts that assert something false

| # | item | the lie | action |
|---|---|---|---|
| 3.1 | `efus3/skills/zk/SKILL.md:12` | *"crons zk-index/zk-graph/zk-embed every 6h (pcl-selfcron)"* — the real schedules were `*/10`, `41 */6`, `23 *`, and all six are now `DISABLED`. | Rewrite the frontmatter to match reality. A skill that mis-states its own schedule teaches an agent to trust stale instructions. |
| 3.2 | `efus3/skills/zk/SKILL.md:11` | *"index derived + gitignored"* — true, and therefore the whole layer is reconstructible; but the same file lists an embedding server on pk31 as a given. | Rewrite with the model/host actually in use, or with no host at all. |
| 3.3 | `zdash222/docs/reinit/REGISTER.tsv` row **A-16** | asserts graphify *"has produced ZERO graph.json artifacts anywhere on pk33 in the 51 days since install"* and *"find / -maxdepth 6 -type d -name graphify-out returns NOTHING machine-wide"*. **False today:** 53 `graphify-out` dirs, 7.7 GB, 49 `graph.json` files. The register was written on 2026-09-15 and never refreshed. | Update the row. A register whose evidence is stale is worse than no register, because it is trusted. |
| 3.4 | `zdash222/docs/dod/WORK.tsv` row `ZDOD-M-20260915T235544Z-2098922-22664` | a DoD that requires `grep -qP '^fleet-mirror-pk\t' _logs/zk/scopes.tsv` for a mirror that **does not exist on this machine** and never will. | Close it as "not wanted — D-09 never wired". A DoD that cannot pass is a decoration. |
| 3.5 | `zdash222/docs/dod/WORK.tsv` row `ZDOD-M-20260916T000019Z-2173664-20048` | a DoD whose first conjunct is `curl -sf http://127.0.0.1:18095/v1/models` — **a URL that has had no listener since pk31 went offline**. | Same: close it, or re-point it at a model that exists. Do not leave it "DOING" — it is not in flight, it is dead. |
| 3.6 | `zdash222/docs/dod/WORK.tsv` row `ZDOD-M-20260916T000338Z-2209392-31610` | DoD = `bin/zk_graph.sh --check`, i.e. **45/45 GREEN** — a target that costs ~60 min of CPU per pass by the measured per-repo costs, against a 6-hourly cron that never once reached it. | Either retire the row or re-scope it (step 5). Leaving it `BLOCKED`/PARK is honest; leaving it "in flight" is not. |
| 3.7 | `zk_embed.py`'s `POP_FILTER` denominator | produces `coverage=36145/35925 100.6%` — a coverage figure that exceeds 100% because the numerator (chunked paths) and the denominator (filtered md docs) are different populations. | Fix the metric or stop printing it. A metric that can exceed 100% is a decoration. |
| 3.8 | `docs/protocols/BACKUP-LEGS.tsv` | carries **no row** for the intelligence layer — 6.9 GB of derived artefacts with no backup leg, and no R2 remote configured on this box at all. | Add the leg for whatever survives step 6, or record explicitly that the layer is rebuild-only. |

### What must NOT be thrown away

* `efus3/scripts/zfts.py` and `efus3/scripts/zctx.sh` — the two best-engineered
  pieces of the layer. `zfts.py` has an atomic build, a content-addressed scope
  merge, a secret-shape filter, a spine exclusion, and a coverage oracle. `zctx.sh`
  has computed facts, sha seals, content freshness and a token budget. **Both are
  the raw material of ZK2.**
* `efus3/zfable/` (94 M, 2,608 files) — source doctrine, not derived data.
* `efus3/skills/zk/SKILL.md`, `efus3/skills/zctx/SKILL.md` — the surface contract
  (rewrite the frontmatter, keep the contract).
* `zdash222/docs/knowledge/zk-v0-brief.md` and `zk-unit3-embed-brief.md` — the
  original design intent, still the best statement of what the layer was for.
* `efus3/_logs/rework-ledger/2026-10-31.md` — the human ledger of the pause, with
  the recovery commands. It is the model for the ledger ZK2 needs.
* `efus3/_logs/agent-crons/registry.tsv` — machine truth for cron state. Do not
  bypass it; `pcl_selfcron.sh` exists and works.
* The `graphify-out/graph.json` of the **current** HEAD of any repo you keep in
  the code-graph tier — but not the 99 dated snapshots, and not the caches.

---

## OPEN QUESTIONS

1. **Does the code graph earn its keep at all?** `zk explain` is the only reader, it
   has no recorded consumer, and 45/45 repo scopes are RED. The A-16 row has been
   `BLOCKED`/PARK since 2026-09-18 with the note "seat v1-knowledge is a paused Fable
   seat". **Is the honest answer to retire A-16 rather than re-create it?** This
   decides 7.7 GB and the largest chunk of the CPU budget. I lean retire — but the
   decision belongs to pk, and it is the single highest-leverage question here.

2. **Is the embedding tier wanted, and on which model?** bge-m3/1024-dim (remote,
   pk31, gone) vs bge-small/384-dim (local, cached, 15× slower). They are not
   interchangeable: the stores have different dimensionalities, so a "restore"
   cannot be migrated between them. And if the answer is "neither", the layer is
   FTS + ripgrep only — which is what the `zfts.sh` charter asked for in the first
   place ("stop 'LLM reading markdowns which has proved disastrous'").

3. **Is pk31 coming back, or is pk33 now the only seat?** `tailscale status` says pk31
   "offline, last seen 4d ago" and pk32 "offline, last seen 23d ago"; `aw` and `pk`
   are up. If pk31 is gone for good, `zk-embed-tunnel` must never return, and the
   embedding tier must be local or dropped. If pk31 returns, the tunnel path is
   15× faster than local and the tier is worth rebuilding. **Nothing in the
   intelligence layer's future can be decided before this question is answered.**

4. **Who actually reads the intelligence layer?** Measured consumption today:
   one hourly report cron, two DoD oracles (`bin/zk status`, `zk_check_paths.sh`),
   and an agent-facing CLI that the `zk` skill advertises but whose one recorded
   smoke test is `bin/zk search "twdpt1 DPR" --json` — a *smoke*, not a use. TRUNK-SCOPE
   T-16's own DoD is a smoke. **Is the estate still trying to answer "where is X
   across all repos", or has it moved on?** If nobody reads it, the correct target
   architecture is 100× smaller than the one I proposed above, and that is a
   legitimate outcome of this exercise.

5. **Who is authorised to resolve the 82-commit divergence?** `zdash222` is
   `main...origin/main [ahead 82, behind 2]`; `origin/main` still points at
   `39497bc`, the pre-rebase main. The scope-prune fix `c3fa3e7` — the one commit
   that removes the rot at the scope layer — exists **only on this disk**, on local
   `main` and two local backup branches. The `gitea` remote (`pk31.local:3000`) is
   unreachable. Until this is pushed (or deliberately not), **the source of truth
   for the intelligence layer's code is this laptop's disk.** The same question
   applies to `efus3` (`ahead 2, behind 10`).

6. **Who owns the `zk_report.sh` hourly push?** It runs `git pull --rebase
   --autostash origin main` on a repository that is 82 commits ahead of its origin.
   That is a rewrite hazard on a repo that has already spent 24 days in a stuck
   rebase. Is the hourly report still wanted at all, and should it be allowed to
   push?

7. **Should the 9 staged ctx packs be committed as-is, or recompiled?** Their
   `## SOURCES` digests still match the live sources exactly (I recomputed all
   four by hand), so v1.0.24 is genuinely GREEN. But the pack's NOTES INDEX section
   is empty because `zexp1/factory6000` does not exist on pk33 — a compile source
   that is missing rather than stale. **Is factory6000 supposed to be on pk33?** If
   yes, cloning it is a one-line fix and the pack gains a section it has always
   been missing. If no, the source list should drop it.

8. **What is the `pk33-cleanup` scope actually for?** Its register row points at
   `/home/pk33/awgbackups/pk33-cleanup`, but the content lives at
   `/home/pk33/pk33backupsv1/pk33-cleanup` and `~/awgbackups/` is now **empty**.
   This is the one scope whose root moved, and it is the scope that is silently
   costing a permanent RED. Is the cleanup repo in scope at all — or was it only
   ever a staging area?

9. **Do the `~/.claude/projects/*/memory` directories still belong?** 7 of them are
   admitted as `claude-memory` (merged under one scope name by design). Four more —
   the scratchpad, `/tmp`, and bare `/home/pk33` — were dropped by the Oct-31 prune.
   Is there an admission test for *memory* scopes analogous to A1–A6 for clones, or
   does "non-empty" suffice? As of today, "non-empty" is the whole test, and it is
   a weak one.

---

*Seat C. Read-only throughout: no deletion, move, edit, commit, restart, or
cron re-enable. Every figure in this document was measured on pk33 on 2026-11-02;
every file cited was read. Where a claim originates in the audit documents
(`pk-rework-assessment-3-30oct0134hrs.md`, `pk-observations-gotchas-audit-4-30oct0134hrs.md`)
it is re-derived here, and where the audit and the live box disagree the live box
wins — see the corrections in the preamble, §1.5, §3.1 and §3.2.*
