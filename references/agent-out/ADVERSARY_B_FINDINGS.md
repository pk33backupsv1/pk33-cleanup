# ADVERSARY B — FINDINGS: The Steward / Recovery Architect

Stance: this machine (pk33) has been the core working seat for work on pk, pk31, pk32
and aw, and it ran the self-hosted Gitea. Nothing here is lost — everything below is
**paused and needs a re-initiation plan**, not a rebuild from scratch. Every verdict is
grounded in a file I actually read or a read-only command I ran on 2026-11-02. No writes,
moves, restarts, or deletions were performed on the audited system.

---

## 1. CORE WORKINGS (live/paused verdict + dependencies)

### 1.1 efus3 — the SSOT engine seat

**What it is.** The eduZ product monorepo (`/home/pk33/zexp1/efus3`): an AI-first K-12
education platform for India, 36 apex domains plus subdomains, ~330 deployable frontends,
18 backend `*-svc` microservices, the ONE zcons kernel (kc·dc·rc), and the skills SSOT.
Its submodules are `zcons1` (consolidation plane), `zsm1` (study-material repo), and
`expz2` (pk's control room). `zfable` is the trunk/family governance seat; `zdash222` is
the production control room. Source: `efus3/README.md`.

**Verdict: codebase LIVE / automation PAUSED-by-pathology.** The tree, spines, and git
history are present and healthy; what is unhealthy is the autonomous loop. The 98-entry
crontab is 97/98 pointed at efus3, and the two CPU hot spots both trace to efus3:
`fast_pulse.sh` (every 2 min, spawns `verify efus3`) and `sau_keeper.sh` (every 10 min,
drives `zcons-dc.py rc ingest`). Both degrade monotonically because
`zcons1/ssot_engine/intake.py`'s `append()` re-scans the full 91 MB `rc/intake.ndjson` on
every event. Source: `pk33-cleanup/pk-load-audit-1-30oct0134hrs.md`,
`pk33-cleanup/pk-rework-assessment-3-30oct0134hrs.md`.

**Depends on:** the zcons1 kernel submodule; zfable family governance; zdash222 as the
control room; GitHub origin `zexp1/efus3` (reachable); Gitea origin
`http://pk31.local:3000/pk/efus3.git` (**offline** — verified via `git remote -v`).

**What breaks if it stays paused.** The dead-man switch does its job: every dashboard
banner goes STALE when the prober dies, so "nothing green can silently rot." Pausing the
crons stops the spine feeders and the 6h convo-collect loop, but it destroys nothing —
the `dc`/`rc` spine data already lives on the HDD via `/etc/fstab` bind mounts, and the
full history is on GitHub.

### 1.2 zcontent1 — the live KLO-universe content design work

**What it is.** Born 2026-09-19, the content consolidation home at
`/home/pk33/zdash22x/zcontent1`. pk's ruling orients the whole working around it: build
two parallel KLO universes on the same grade ladder gK–g12 — `zsm-KLO` (3,019 KLOs) and
`zncert-KLO` (6,556 KLOs) — cross-linked through a shared topic/KC layer, design-first.
The four-box model (`zsm` × `zncert`, `engine` × `curriculum`) lives in
`docs/FOUR-BOXES.md`. `zsm1` is a git submodule here. Source: `zcontent1/README.md`.

**Verdict: LIVE — this is the rework target.** Design is done (`docs/DESIGN-TWO-UNIVERSES.md`);
the build is not — `docs/KLO-UNIVERSE-MATRIX.tsv` still carries open `gap_reason` rows.
Branch `feat/content-consolidation`, HEAD `ae50b01`. Source:
`pk33-cleanup/pk-rework-assessment-3-30oct0134hrs.md` §3.2.

**Depends on:** the zsm1 SSOT (submodule, pinned); the restored NCERT markdown
(10,411 files restored from B2 to `/mnt/wdhdd/zsm-ncert/`); the pk31 GPU zoo for the
verifier bake-off; bge-m3 embeddings served by pk31; GitHub origin `zdash22x/zcontent1`
(reachable); Gitea origin `http://pk31.local:3000/pk/zcontent1.git` (**offline**).

**What breaks if it stays paused.** The "content per KLO" unit of account that pk
explicitly asked for stays half-built; the 5,786 AMBIGUOUS zncert Stage 3 rows stay
unresolved; the verifier bake-off never runs; and `evs.g3` publish stays blocked on
`zsm1 feat/consolidation` landing.

### 1.3 zncert-w1 — the published product (7 gated Cloudflare Pages sites)

**What it is.** NCERT corpus → exact, labelled, gated HTML, at `/home/pk33/zncert-w1`.
Working id `v2-w1-p1-s1`, edition `v2-2026-10-01`. English is published first (470
chapters, 8,299 pages); Hindi is replicas-only (505 chapters, text untrustworthy).
Source: `zncert-w1/README.md`, `zncert-w1/docs/HANDOVER.md`.

**Verdict: LIVE and PUBLISHED (English). Hindi deliberately paused** (ADR-0006
"English first, deliberately"; Hindi OCR is broken — 0 of 107 chapters produced
Devanagari, ADR-0011). The gate is server-side: Supabase Edge Function `gate`, HTTP-only
cookie, bcrypt via `zn_login()` — 401 without cookie, 200 with; 243 books / 133 published
assets. Seven gated Cloudflare Pages sites: zncert2-metadata, zncert2-w1, zncert, zncert1,
zncert3, zncert4, zncert5.

**Depends on:** Supabase Postgres (metadata) + the Edge Function gate; Cloudflare Pages +
scoped token; pk31 as the secondary worker (WiFi −76 dBm, ~30% SSH failures); the laptop
as orchestration-only; the **local** Gitea remote `ssh://git@127.0.0.1:2222/zexp3/zncert2-w1.git`
(healthy — this is the pk33 container, not pk31); and the 14 GB `library/` (2,569 PDFs —
SOURCE, must stay on SSD).

**What breaks if it stays paused.** The published English surface keeps serving (the gate
is now server-side), but no new content ships; the 3 in-progress labelled chapters (Class
12 Physics, Accountancy, Economics) stall; and the legacy zncert2 catalogue stays unwired
behind the gate.

### 1.4 zsm1 — the study-material SSOT

**What it is.** The single source of truth for zsm study content — github `zexp1/zsm1`,
checked out in-tree at `efus3/zsm1`. A grade-free, subject-free K-12 study-material engine;
content trio (507 topics / 1,580 cards / 940 items) authored at 100% coverage behind the
F11 human-review gate. Publish surface is `https://eduz.study` (Cloudflare Pages),
byte-identical via a sha manifest. Source: `zsm1/README.md`, `zsm1/README.STATUS.md`.

**Verdict: FINALISED / GREEN — do not resume it.** Re-running the checkers (not trusting
the 2-month-stale `RESUME-HERE.md`) gives: depth 1961/1961, Indianization 0 faults over
30,164 artefacts, boundary 4/4 PASS, git tree clean, with a deliberate
`close-down sweep` commit `311036bb`. Source: `pk33-cleanup/pk-rework-assessment-3-30oct0134hrs.md`
§3.1, `pk33-cleanup/pk-observations-gotchas-audit-4-30oct0134hrs.md` §5.1.

**Depends on:** the zcons kernel (STANDARD profile); R2 for big derived artefacts
(sqlite/ndjson/zip/mp4 — git keeps text only); `zdash222/docs/zsm/PUBLISH-MANIFEST.tsv`
for byte-identical publish; GitHub origin `zexp1/zsm1` (reachable); Gitea origin
`http://pk31.local:3000/pk/zsm1.git` (**offline**, GitHub mirror covers it).

**What breaks if it stays paused.** Nothing — it was deliberately closed clean. The risk
is the opposite: "resuming" zsm1 re-does finished work and disturbs the one thing on the
box that is genuinely done and authoritative. The only real exposure is if the R2 derived
artefacts become unreachable, which would block byte-identical re-publish of eduz.study.

### 1.5 zk / graphify — the knowledge-graph layer (what it was FOR)

**What it was for.** ZK is the machine's own knowledge layer: "where is X / what says Y"
across every repo clone on the box, the laptop mirror, and `~/.claude` memory, without an
LLM. It is four things composed: (1) fts5 text search over `zfts` scopes; (2) ripgrep
definition sweep; (3) `graphify` code graphs per scope (tree-sitter, local, no API);
(4) optional bge-m3 semantic embeddings in sqlite-vec, served by the pk31 embed server.
Verbs: `search` / `where` / `explain` / `vsearch` / `index` / `embed` / `scopes` /
`status`. Source: `skills/zk/SKILL.md`.

**Verdict: LIVE but pathological — this is the "core knowledge-graph rework."** Every run
returns `ok=3 red=45` (45 of 48 scopes RED), averaging 738 s/pass against a 10-minute cron
that perpetually overlaps itself. The scope list contains the 2.2 GB Flutter SDK, two
`~/trash/` directories, and `claude-memory` repeated 11×. It pins a core at 100% and drove
the CPU package to 89–100 °C (crit 100 °C). It was paused on 2026-10-14 for exactly this
reason and the pause was later reverted. Source:
`pk33-cleanup/pk-rework-assessment-3-30oct0134hrs.md` §2, `GRAPHIFY-PAUSED-2026-10-14.md`,
`pk33-cleanup/pk-observations-gotchas-audit-4-30oct0134hrs.md` §1.1–1.3.

**Depends on:** `zdash222/bin/zk`; the scope list derived from
`zdash222/docs/registers/clones.tsv`; `graphify` (local, regenerable in ~125 s for the
embed index); and the **pk31 bge-m3 embed server (offline)** for semantic search.

**What breaks if it stays paused.** Semantic search (`vsearch`/`embed`) is already broken
while pk31 is offline; the local text/code search (fts5 + ripgrep) still works. Keeping the
current rebuild loop running un-reworked burns a full core at thermal-critical temperatures
for zero successful scopes — the worst of both worlds. The right shape is: pause the broken
loop, keep local search, prune the scopes, and restore embeddings only when pk31 returns.

### 1.6 Gitea seat (rustzydesk-gitea container, zexp3 org, 4 repos)

**What it is.** Self-hosted Gitea on pk33: container `rustzydesk-gitea`
(`gitea/gitea:latest`, Up), DOMAIN `localhost`, SSH_DOMAIN `pk33.local`, HTTP :3000, SSH
:2222, data persistent in `/home/pk33/gitea/data`. The zexp3 org holds exactly 4 repos,
verified on disk at `/home/pk33/gitea/data/git/repositories/zexp3/`:

| repo | commits | state |
|---|---|---|
| `rustzydesk` | 4 | a Flutter chat-first support-desk app (pin 3.24.5) |
| `zncert2-w1` | 3 | zncert-w1's working repo — its primary remote |
| `zncert-w1-tracking` | 1 | seed tracking repo (handover/status/verification) |
| `zncert2` | 0 | **EMPTY — still to seed** |

Source: `zncert-w1/docs/HANDOVER.md`; `git --git-dir` commit counts on 2026-11-02.

**Verdict: LIVE.** The container is up and zncert-w1's canonical self-hosted origin
(`ssh://127.0.0.1:2222/zexp3/zncert2-w1.git`) is healthy. HANDOVER's "seed
zncert-w1-tracking and zncert2" is half done: tracking is seeded; zncert2 is still empty.

**Depends on:** `/home/pk33/gitea/data` (persistent); `~/.gitea_token` (mode 600) for the
HTTP API; the docker runtime. It does **not** depend on pk31.

**What breaks if it stays paused.** zncert-w1 loses its self-hosted origin (GitHub mirror
`zexp2/zncert2-w1` remains); the rustzydesk support-desk work stops. Because it is already
up, the recovery action is verification + seeding, not a restart.

---

## 2. FLEET DEPENDENCIES

Fleet reality, verified read-only on 2026-11-02 (`tailscale status` + pings + ssh):

| node | tailscale IP | state | notes |
|---|---|---|---|
| pk33 | 100.127.179.91 | online | this seat |
| pk31 | 100.114.44.10 | offline, last seen 4d | **keystone** — unreachable (ping timeout) |
| pk32 | 100.83.201.29 | offline, last seen 23d | unreachable (ping timeout) |
| aw | 100.87.8.127 | offline 86d | **LAN ssh still works**: `ssh aw` → host `aw`, user `aw`, `~/awgbackups` **empty** |
| aw-1 | 100.94.142.78 | online | reachable at IP; tailnet policy denies SSH as pk33 |
| pk | 100.99.62.21 | online | `ssh pk` works → host `pk`, user `pk`, no `~/awgbackups` |

**What the topology implies for "re-initiate core workings."**

pk31 is the single keystone dependency. It is simultaneously: (a) the Gitea origin
(`pk31.local:3000`) for efus3, zsm1, zcontent1, and zdash222 — verified in each repo's
`git remote -v`; (b) the GPU zoo (Qwen3.8-27B, gemma-4-26B, bge-m3 embeddings,
Qwen2.5-VL) that powers the verifier bake-off and embeddings; (c) the zk embed server; and
(d) zncert-w1's secondary worker. Its 4-day outage blocks semantic search, the verifier
bake-off, Hindi OCR work, and the canonical Gitea remotes.

The mitigating fact: **git is not hostage to pk31.** All four pk31-origin repos have live
GitHub mirrors — `zexp1/efus3`, `zexp1/zsm1`, `zdash22x/zcontent1`, `zdash22x/zdash222` —
so history and working copies are recoverable right now from pk33.

Two corrections to the prior audits, from this pass:

1. **`pk` is a live fleet machine**, not a dead hostname. `pk33-cleanup/pk-rework-assessment-3`
   and audit #4 both treat the 60 `pk` registry rows as "a hostname you no longer have."
   Live `tailscale status` and `ssh pk` show host `pk` online at 100.99.62.21. Those 60
   rows belong to a real machine and must be reconciled, not discarded.
2. **`aw`'s `~/awgbackups` is empty.** The off-site backup destination is not actually
   populated today, so destructive pruning has no verified off-box fallback until a real
   backup is placed there.

**What must be verified before anything is turned back on:**

1. pk31's power/network state — it has been offline 4 days and must be physically booted;
   this is not a pk33 action.
2. GitHub mirrors are current relative to the pk31 origin (read-only `git fetch`/ahead-behind).
3. B2 (rclone `b2-remote:zsm-ncert`) and R2 are reachable from pk33 — B2 holds the NCERT
   markdown/PDFs, R2 holds zsm1's big derived artefacts.
4. Supabase + Cloudflare still serve the published zncert-w1 surface (gate returns 401/200).
5. Where the real off-site backup lives, since `aw:~/awgbackups` is empty.

---

## 3. RE-INITIATION PLAN (ordered — trigger / first observable signal / rollback)

Assumption: you cannot test everything at once, so each step gates the next and has a
single observable "it worked" signal and a way back.

### Step 0 — Precondition: make the box safe to work on (record + thermal + narrow delete)

- **Trigger:** none — this is the first act of recovery, before any "working" resumes.
- **Action:** snapshot the crontab and `efus3/_logs/agent-crons/registry.tsv`; add a
  thermal guard that pauses graphify above ~85 °C; disable the zk rebuild family via
  `pcl_selfcron.sh` (longest-tag-first, see Risks); delete only
  `~/.local/share/opencode/opencode.db{,-wal,-shm}` — **not** the directory, which holds
  four live API credential files.
- **First signal:** `sensors` settles below ~80 °C sustained and `top` no longer shows a
  graphify process at 100%; `df -h /` jumps from 215 G free toward ~480 G.
- **Rollback:** `pcl_selfcron.sh enable` per entry; the db deletion is the one irreversible
  move, which is why it is scoped to the db only and the keys + `snapshot/` are kept.

### Step 1 — Freeze zsm1 and prove it is GREEN

- **Trigger:** Step 0 (recovery must begin from a trusted content foundation).
- **Action:** re-run `tools/kcgraph/depth_status.py`, `indianization_status.py`,
  `g6_hindi_status.py`, `boundary_gate.py`; confirm `git status` clean.
- **First signal:** depth 1961/1961, Indianization 0 faults, boundary 4/4 PASS, clean tree
  — the numbers `RESUME-HERE.md` got wrong.
- **Rollback:** none needed — read-only verification; the `close-down sweep` commit stands.

### Step 2 — Gitea seat: verify + seed the empty repos

- **Trigger:** Step 1 (the local origin of truth must be trusted before anything is pushed).
- **Action:** confirm :3000 and :2222 respond; confirm 4 repos exist; seed `zexp3/zncert2`
  (still 0 commits) and complete `zexp3/zncert-w1-tracking`.
- **First signal:** Gitea serves over HTTP; `ssh://git@127.0.0.1:2222/zexp3/zncert2-w1.git`
  accepts a fetch/push; repo count = 4 with zncert2 non-empty.
- **Rollback:** the empty repos have no history to lose; the container is already Up, so
  a bad seed is simply re-pushed.

### Step 3 — zncert-w1 English surface: verify published + resume the 3 local chapters

- **Trigger:** Step 2 (zncert-w1's canonical origin is local Gitea).
- **Action:** curl the gate (401 without cookie, 200 with); confirm the 7 CF Pages sites;
  build the 3 in-progress labelled chapters (Class 12 Physics, Accountancy, Economics) via
  `bin/zw1-build`.
- **First signal:** a wrangler deploy of one chapter succeeds and the gate returns 200 for
  that slug.
- **Rollback:** re-deploy the last known-good chapter dir; `library/` stays on SSD (never
  pruned — it is SOURCE, per HANDOVER's `.gitignore` bulk-guard discipline).

### Step 4 — zcontent1 rework: commit the real TSVs, then build the KLO ledger (local-only)

- **Trigger:** Step 1 + the B2 markdown already restored to `/mnt/wdhdd/zsm-ncert/`
  (10,411 files, per zcontent1 README §6).
- **Action:** commit the 3 real TSVs (`CARD-SOURCE-CANDIDATES-{hindi,latin}`, `CURRENT.tsv`);
  then regenerate `STOCK-TAKE.tsv` / `KLO-LEDGER.tsv` / the KLO matrix.
- **First signal:** the 14 design gates (`bin/run_gates.py`) GREEN; findings index
  regenerates; matrix reconciles against both KLO ledgers.
- **Rollback:** additive-law git-revertable — every number carries its command; the
  `docs/history/` snapshots are the archive-before-regenerate safety net.

### Step 5 — Fleet gate: pk31 back online (the keystone)

- **Trigger:** pk31 physically powers on / `tailscale status` shows it online — external,
  not a pk33 action.
- **Action:** ping `100.114.44.10`; hit `pk31.local:3000`; `nvidia-smi` on pk31 for a free GPU.
- **First signal:** pk31 pings; its Gitea responds; the RTX A6000 shows capacity.
- **Rollback:** if pk31 stays offline, the plan splits — Steps 6–8 wait; local zk
  text/code search still works in the interim.

### Step 6 — zk rework: prune scopes → rebuild code graph (local) → restore embeddings (needs pk31)

- **Trigger:** Step 5 (embeddings) or, for the local half, Step 0.
- **Action:** edit `zdash222/docs/registers/clones.tsv` to drop the Flutter SDK, the two
  trash dirs, and 10 of 11 `claude-memory` rows; regenerate `scopes.tsv` (60 → ~47); rebuild
  `graphify` code graphs locally; restore `zk embed` only when the pk31 embed server is back.
- **First signal:** `zk status` GREEN (not RED); graphify-out regenerates without the
  Flutter SDK; `vsearch` returns results once embeddings are restored.
- **Rollback:** the pruned `clones.tsv` is a git edit (revertable); the embed index
  regenerates in ~125 s, so there is no reason to preserve the poisoned one.

### Step 7 — Verifier bake-off + zncert Stage 3 (the actual content work)

- **Trigger:** Step 6 (GPU zoo + embeddings restored).
- **Action:** run the bake-off (runs A–E) on the 109 trailed cards; pick a two-verifier
  pre-filter; resume `zsm↔zncert` Stage 3 matching (2,302 Devanagari rows already carry
  strand candidates — title-level matching, not strand discovery).
- **First signal:** a bake-off run reproduces; a two-verifier policy is chosen; the
  AMBIGUOUS count drops below 5,786 with a receipt per batch.
- **Rollback:** each receipt is git-revertable; the PL loop (draft → independent verify →
  correct → specialist → publish-only-approved) is unchanged.

### Step 8 — Hindi OCR diagnosis (last, deliberately)

- **Trigger:** Step 7 green + pk31 secondary worker healthy + pk's explicit go.
- **Action:** re-run PaddleOCR on a det model; compare against the English-clean path.
- **First signal:** >0 Devanagari chapters extracted (today it is 0 of 107).
- **Rollback:** Hindi stays paused — ADR-0006 "English first, deliberately" is the default.

---

## 4. WHAT STAYS PAUSED (and why)

- **zsm1 itself** — it is finalised and clean. Resuming it re-does finished work and
  disturbs the only genuinely-done component. (`pk-rework-assessment-3` §3.1, audit #4 §5.1)
- **The graphify/zk full-rebuild cron family** — 45/48 RED every pass, 738 s per 10-min
  cron, 89–100 °C, for zero successful scopes. Keep it paused until scopes are pruned and
  a thermal guard exists. (`pk-rework-assessment-3` §2, `GRAPHIFY-PAUSED-2026-10-14.md`)
- **Flutter SDK / trash / `claude-memory`×11 KG scopes** — remove permanently from the
  scope list; they are the reason the KG never goes green. (`audit #4` §1.3)
- **k3s** — crash-looping (62–73 restarts/pod), ~9% of a core, not needed for the
  re-initiation. Keep stopped unless Kubernetes is actually required. (`pk-load-audit-1`)
- **tzerp/zsm1-tzerp** — declared READ-ONLY ARCHIVE 2026-09-16; SSOT is zexp1/zsm1. Do not
  resume work there. (`zsm1/README.md`, `pk-prune-assessment-2` §3.5)
- **efus3-demoguest-merge** — an in-flight merge (11 GB, untracked `_logs/`). Hold until it
  is explicitly closed or abandoned. (`pk-prune-assessment-2` §2.4)
- **Hindi publication** — OCR is broken (0/107 chapters) and pk's own ordering is "English
  first." (`zncert-w1/README.md`, `docs/HANDOVER.md`)
- **evs.g3 publish** — pending `zsm1 feat/consolidation` landing on the SSOT export; a
  scheduling call, not a technical blocker. (`zcontent1/README.md` §9)
- **zemma fine-tune** — waits for ≥1,000 span-verified cards. (`zcontent1/README.md`)
- **opencode (the tool)** — retire, don't resume: delete only the 244 GB db, keep the four
  API keys and `snapshot/`. (`audit #4` §1.4)
- **The 131 GB `zsm-ncert-20260919/ncert-pdfs` corpus** — stays cold on HDD; B2 markdown
  suffices and pk said ignore DIKSHA. (`zcontent1/README.md` §6, `pk-prune-assessment-2` §2.1)

---

## 5. RISKS

- **Thermal recurrence.** 89–100 °C happened once, was fixed on 2026-10-14, and the fix was
  reverted. Needs a policy (thermal guard), not a one-off. (`audit #4` §1.1)
- **opencode credential caveat.** Deleting `~/.local/share/opencode/` wholesale destroys 4
  live API keys (`auth.json`, minimax/deepseek/mimo keys). Delete the db only. (`audit #4` §1.4)
- **`pcl_selfcron` substring collision.** `disable zk-index` silently also drops
  `zk-index-zdash222`, and the shrink guard permits it. Disable longest-tag-first and
  `verify` after each. (`audit #4` §1.5)
- **`GRAPHIFY-PAUSED-2026-10-14.md` footgun.** Its `sed` restore command would re-enable
  three jobs if run today. Restore through `pcl_selfcron.sh enable` per entry instead.
  (`audit #4` §5.2)
- **24 worktrees with unmerged `wip/pk33/*` branches (6.9 GB).** `git worktree remove` on an
  unmerged branch loses the tip. Run `git branch --no-merged` first. (`audit #4` §3.2)
- **pk31 is a single point of failure** for GPU, embeddings, and the Gitea origin. GitHub
  mirrors are the current mitigation; re-pointing the `gitea` remotes at the local pk33
  Gitea (or GitHub) would remove the dependency. (verified `git remote -v`, 2026-11-02)
- **No off-site backup in place.** `aw:~/awgbackups` is empty, so destructive pruning has
  no verified off-box fallback until a real backup is placed. (verified `ssh aw`, 2026-11-02)
- **`pk` is live, not stale.** The 60 `pk` registry rows belong to a reachable machine;
  discarding them (as audit #3 suggested) would erase a real host's cron provenance.
  (verified `tailscale status` + `ssh pk`, 2026-11-02)
- **zprotocols cron stages-but-never-commits** (the `||` grouping in `zctx-refresh`), so 19
  files accumulate in the index after every "clean." Fix the cron before treating the tree
  as cleanable. (`audit #4` §6.2)
- **`intake.append()` O(n) rescan degrades forever.** Re-enabling `fast_pulse`/`sau_keeper`
  without fixing max-seq caching just re-creates the CPU burn. (`pk-load-audit-1` §1.1)

---

*Seat B (Steward / Recovery Architect). Read-only throughout; the only write is this file.
Grounded in: efus3/README.md, zsm1/README.md + README.STATUS.md, zcontent1/README.md,
zncert-w1/README.md + docs/HANDOVER.md, skills/zk/SKILL.md, pk33-cleanup audits #1–#4 +
human_typed.md + README.md, GRAPHIFY-PAUSED-2026-10-14.md, and live read-only commands on
2026-11-02 (tailscale status, ping, ssh aw/pk, docker ps, git remote -v, gitea data).*
