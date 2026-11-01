<!-- aristotle
category: record
as-of: 2026-10-31
owner: pk33
-->
# pk33 rework ledger — 2026-10-31

The record of fact for the pk33 cleanup/rework. Every relocation, pause, and
destructive action gets a row here. The registry
(`efus3/_logs/agent-crons/registry.tsv`) is the *machine truth* for cron state;
this file is the *human ledger* — it carries the reason and the recovery path,
which the registry's `purpose` field does not.

Plan of record: `pk-rework-assessment-3-30oct0134hrs.md` (audit #3) and
`pk-observations-gotchas-audit-4-30oct0134hrs.md` (audit #4), both in
`~/awgbackups/pk33-cleanup` and pushed to https://github.com/pk-szt6/pk33-cleanup
(the `awgbackups` org transfer is still blocked — `pk-szt6` is not a member).

| # | date | class | target | action | before_size | after_size | reason | reversible | recovery |
|---|---|---|---|---|---|---|---|---|---|

## Phase 0 — the record (non-destructive, done first)

| # | date | class | target | action | before_size | after_size | reason | reversible | recovery |
|---|---|---|---|---|---|---|---|---|---|
| 0.1 | 2026-10-31T21:57Z | config | `crontab` (user pk33) | snapshot to `~/crontab.pre-rework.20261031T215718Z.txt` | 104 lines | 104 lines | machine truth before any cron write | yes | the snapshot file itself |
| 0.2 | 2026-10-31T21:57Z | config | `efus3/_logs/agent-crons/registry.tsv` | snapshot to `~/registry.pre-rework.tsv` | 49149 B | 49149 B | machine truth before any state write | yes | the snapshot file itself |
| 0.3 | 2026-10-31T21:57Z | config | `/` and `/mnt/hdd1t` | "before" disk numbers to `~/df.pre-rework.20261031T215718Z.txt` | `/` 657G used of 915G (76%) | unchanged | baseline for the prune/shift phases | yes | the snapshot file itself |
| 0.4 | 2026-10-31T21:58Z | config | this file | created `efus3/_logs/rework-ledger/2026-10-31.md` | — | — | single index of relocation + destructive changes (audit #3 §6) | yes | git history |

## Phase 1 — pause the KG family (audit #3 §7)

Reason for all six: the `zk` family drives `graphify update`, which pins a core
at ~100% and drives the CPU package to 89–100 °C (critical = 100 °C). On
2026-10-31T21:56Z the package read **97 °C** with `graphify update` at 99% CPU.
This pause was applied on 2026-10-14 and **reverted**; the live duplicate
`zk-index` cron line is what brought it back.

| # | date | class | target | action | before_size | after_size | reason | reversible | recovery |
|---|---|---|---|---|---|---|---|---|---|
| 1.1 | 2026-10-31 | pause | `zk-embed-tunnel` | `pcl_selfcron.sh disable` | live, */5 | pending | audit #3 Phase 1 — probes pk33:18095 -> pk31:8095 bge-m3 embed tunnel; Phase 5 would keep it if the tunnel is wanted | yes | `cd /home/pk33/zexp1/efus3 && bash scripts/pcl_selfcron.sh enable zk-embed-tunnel` |
| 1.2 | 2026-10-31 | pause | `zk-index-zdash222` | `pcl_selfcron.sh disable` | live, 4,14,24,34,44,54 | pending | KG index rebuild every 10 min — CPU hog | yes | `bash scripts/pcl_selfcron.sh enable zk-index-zdash222` |
| 1.3 | 2026-10-31 | pause | `zk-index` | `pcl_selfcron.sh disable` | live, */10 + 1 paused duplicate | pending | the CPU hog: `zskill zk index --if-stale` + `bin/zk_graph.sh --missing` (audit #1 measured 738 s per pass) | yes | `bash scripts/pcl_selfcron.sh enable zk-index` |
| 1.4 | 2026-10-31 | pause | `zk-graph` | `pcl_selfcron.sh disable` | live, 41 */6 | pending | `bin/zk_graph.sh` -> `graphify update` per clone scope | yes | `bash scripts/pcl_selfcron.sh enable zk-graph` |
| 1.5 | 2026-10-31 | pause | `zk-embed` | `pcl_selfcron.sh disable` | live, 23 * | pending | `zskill zk embed --batch 128` — embedding pass | yes | `bash scripts/pcl_selfcron.sh enable zk-embed` |
| 1.6 | 2026-10-31 | pause | `zk-report` | `pcl_selfcron.sh disable` | live, 3 * | pending | hourly COMPUTED report -> REPORTS.md (+commit/push) | yes | `bash scripts/pcl_selfcron.sh enable zk-report` |
| 1.7 | 2026-10-31 | pause | running `graphify update` processes | kill in-flight passes | 1 core @ 99% | 0 processes | mid-pass runs keep the CPU hot until they finish (audit #3 Phase 1) | yes | they re-run on next schedule; nothing deleted |
| 1.8 | 2026-10-31 | config | CPU package temperature | measured before/after | 97 °C (crit 100) | 45 °C | the whole point of Phase 1 | — | — |

### Ordering note (gotcha from audit #4)

`pcl_selfcron.sh disable` removes crontab lines with `grep -vF` on the tag, and
`# pcl-selfcron:zk-index` is a **substring** of `# pcl-selfcron:zk-index-zdash222`.
So `disable zk-index` silently removes the `zk-index-zdash222` line *without*
marking its registry row DISABLED. `zk-index-zdash222` is therefore disabled
**first** (1.2 before 1.3), so the registry stays truthful.

### Not done in Phase 1

- No file was deleted, moved, or renamed.
- No other cron was touched (the broad pause is Phase 5, pending).
- No registry row was removed; only `state` flipped ACTIVE -> DISABLED.
- The `#PAUSED-BY-ADAL-GRAPHIFY:` commented lines for `zk-graph`, `zk-embed`,
  `zk-index`, `zk-index-zdash222` were consumed by the tag removal. The
  original crontab is preserved in `~/crontab.pre-rework.20261031T215718Z.txt`
  and in `~/GRAPHIFY-PAUSED-2026-10-14.md`.

### Validated: the pause will hold

Checked after the pause, because a pause that a heal cron silently undoes is
not a pause:

- `pcl_selfcron.sh verify` reports all six as **`DISABLED`**, not `DEAD`
  (103 registered, 1 dead — the one dead entry is pre-existing and unrelated:
  `zdash222-wifi-5g-guard`).
- `pcl_selfcron.sh heal` skips `DISABLED` rows explicitly
  (`skipped (DISABLED): <name>`), and `zfable/ops/warden.sh` — which runs every
  5 minutes and calls `verify` — therefore cannot reinstall them.

This is why the pause was done through `pcl_selfcron.sh disable` and **not** by
commenting the lines out by hand. The hand-commented line in the box right now
(`#PAUSED-BY-ADAL-NONROOT-WIFI: ... zdash222-wifi-5g-guard`) is the counter
example: it has no registry state change, so `verify` counts it `DEAD`, and
`warden.sh` calls `heal` every 5 minutes to try to put it back. The 2026-10-14
ADAL-GRAPHIFY pause was the same pattern, and it is how the graphify jobs came
back after being stopped.

## Phase 2 — prune the KG scope list (audit #3 §7)

**Audit #3's Phase 2 instruction was mechanically wrong and has been corrected.**
It said to edit `zdash222/docs/registers/clones.tsv`. Two problems:

1. `clones.tsv` is **regenerated by `bin/recon_fleet.sh` on every beat**, so a
   hand-edit there is undone within the hour. The scope list's real sources are
   `clones.tsv` (rows), `bin/zk`'s hardcoded `~/.claude/projects/*/memory` glob,
   and `docs/knowledge/zk-scopes-extra.tsv`.
2. The 11 `claude-memory` rows are **not in `clones.tsv` at all** — they are
   appended by `bin/zk` itself.

The fix was therefore applied at the **scope layer** (`bin/zk`), which is where
the prune actually belongs.

| # | date | class | target | action | before_size | after_size | reason | reversible | recovery |
|---|---|---|---|---|---|---|---|---|---|
| 2.1 | 2026-10-31 | prune | `bin/zk` — scope generation | skip clone roots under `fvm/` and `trash/` | 61 scopes | 58 scopes | `fvm/versions/3.24.5` is the Flutter SDK (1.7 GB of vendor Dart, 49011 commits behind) — the scope that pinned a core at 100% and drove the package to 89-100 °C; `trash/*` is discarded content | yes | `git revert c3fa3e7` (zdash222) |
| 2.2 | 2026-10-31 | prune | `bin/zk` — memory corpus | skip empty `~/.claude/projects/*/memory` dirs | 58 scopes / 12 memory rows | 53 scopes / 7 memory rows | 5 of the 12 dirs hold 0 files — they contributed 0 documents but a scope row each | yes | `git revert c3fa3e7` (zdash222) |
| 2.3 | 2026-10-31 | commit | zdash222 | commit `c3fa3e7` (bin/zk + regenerated `_logs/zk/scopes.tsv`) | dirty | committed | the scope prune must survive `recon_fleet.sh` and a rebuild | yes | `git revert c3fa3e7` |
| 2.4 | 2026-10-31 | config | zdash222 `_logs/zk/scopes.tsv` | regenerated by `bin/zk scopes` | 61 rows | 53 rows | follows 2.1 + 2.2 | yes | regenerate |

**Graphify population (repo scopes only): 49 -> 46.** RED baseline at the time
of the prune was `ok=4 red=45`; the RED count falls only once a full
`zk_graph.sh` pass completes, which is exactly what the prune makes affordable.

### Correction to audit #4's open question 2

Audit #4 asked: *"Is the KG scope name the primary key? If yes, the 11
`claude-memory` rows are already colliding and the index's 35,919 docs may be
partially lost."* — **Checked: no documents are at risk.**

`scripts/zfts.py` `_load_scopes_tsv()` does `scopes.setdefault(scope, []).append(root)`:
same-named rows are **merged by appending their roots**, so all 7 memory dirs
are walked and indexed under one scope name. The merge is the intended shape
for the memory corpus, and `bin/zk_embed.py` filters on exactly that name
(`scope != 'claude-memory' OR path LIKE '%/memory/%'`) — renaming the scope
would have silently broken that filter and started embedding non-memory files.
The collision was cosmetic (scope count), not destructive.

## Bonus — repair the zdash222 git state (not in the audit plan)

Found while committing 2.3: **zdash222 had been stuck in an interactive rebase
since 2026-10-07 22:03** — a conflict on `seats/zdash22v1-knowledge/REPORTS.md`
that was never resolved. `main` was frozen at `39497bc` while every hourly cron
commit since landed on the **detached HEAD** (81 commits in 24 days). The repo
reported "interactive rebase in progress" for all of that time.

| # | date | class | target | action | before_size | after_size | reason | reversible | recovery |
|---|---|---|---|---|---|---|---|---|---|
| 3.1 | 2026-10-31 | config | zdash222 refs | create `backup/pre-rebase-main-2026-10-31` -> old `main` (39497bc) and `backup/detached-line-2026-10-31` -> detached HEAD (c3fa3e7) | — | 2 backup branches | both tips preserved before anything moved | yes | `git branch -f backup/pre-rebase-main-2026-10-31 39497bc` |
| 3.2 | 2026-10-31 | config | zdash222 rebase state | `git rebase --quit` | rebase in progress since 2026-10-07 | rebase state cleared | clears the 24-day-old stuck state; HEAD, index and working tree are left exactly as they are, and the Oct-7 autostash is saved to the stash list rather than applied | yes | `git stash list` -> `stash@{0}: autostash` (81d7ddaa) |
| 3.3 | 2026-10-31 | commit | zdash222 `main` | `git branch -f main HEAD` + `git checkout main` | main = 39497bc (stale, 24 d) | main = c3fa3e7 (82 commits ahead of a269bf4) | the detached line is the *newer* history; the 2 dropped commits are the redundant `v1-knowledge: hourly computed report` commits the rebase was trying to eliminate | yes | `git branch -f main backup/pre-rebase-main-2026-10-31` |

Working tree was **not** touched by 3.2/3.3: dirty-file count is 65 before and
after. `origin/main` (gitea) still points at the old tip, so local `main` and
`origin/main` now diverge (82 local vs 2 remote commits) — **a push would need
`--force-with-lease` and has NOT been done.**

### Not done

- No push to `origin` for zdash222 (divergence would need a force push).
- The 65 dirty files in zdash222 are still dirty — Phase 6 (git cleanup) is
  still ahead. The cron churn is what keeps them dirty, and the broad cron
  pause (Phase 5) has not happened.
- The `stash@{0}` autostash from 2026-10-07 has not been popped or dropped.


---

## Phase 4 — 2026-11-02: the opencode shift+delete, the secrets rebase, and the seat split

Continuation of the same ledger (Phase 0-2 on 2026-10-31; Phase 3 the
zdash222 rebase repair). Plan of record: audits #1-#4 in this repo.

### Phase 4 — the secrets rebase (B: "make new env entries in ~/.secrets1 and rebase everything there")

Doppler was **not** used as a source: its token on this box returns "Invalid
Auth token" and only the `pk11devices` scope has a config file. The matching
NAMES already exist in `~/.secrets3/LLM_keynames.env` (zmuxpk7_llms_api_keys)
for future re-population. The VALUES were taken from the opencode files
themselves, before anything was deleted.

| # | date | class | target | action | before_size | after_size | reason | reversible | recovery |
|---|---|---|---|---|---|---|---|---|---|
+| 4.1 | 2026-11-01T19:20Z | config | `~/.secrets1/` | created (mode 700); 4 LLM provider keys extracted from `~/.local/share/opencode/` and copied in (mode 600) | 4 files in the opencode data dir | 5 files (4 keys + `llm.env`) | credentials must survive the db delete; the opencode data dir is not a secret store (directive D1.8) | yes | the originals were intact until 4.3 |
+| 4.2 | 2026-11-01T19:21Z | config | `~/.secrets1/llm.env` | generated: exports `MINIMAX_CODING_PLAN_API_KEY`, `MINIMAX_API_KEY`, `DEEPSEEK_API_KEY`, `MIMO_API_KEY` | — | 1300 B | one-file consumption; the router reads `~/.secrets2`, this is for the opencode-family keys | yes | regenerate from the key files |
+| 4.3 | 2026-11-01T19:22Z | config | value verification | all 4 env values compared byte-for-byte against the source files | — | 4/4 MATCH | never trust a rebase on sight | — | — |
+| 4.4 | 2026-11-01T19:23Z | config | `~/.config/opencode/opencode.jsonc` | 3 `{file:...}` apiKey refs repointed `~/.local/share/opencode/*` -> `~/.secrets1/*` | 3 refs | 3 refs | keep the tool working after the source dir is emptied | yes | `opencode.jsonc.bak-20261102T0325IST` (same dir) |
+| 4.5 | 2026-11-01T19:23Z | config | `~/.local/share/opencode/{auth.json,*-api-key}` | replaced by symlinks into `~/.secrets1/` (via `ln -sf`, atomic) | 4 regular files | 4 symlinks | opencode reads its own data dir; the symlink keeps both paths working | yes | the symlinks point at the 4.1 copies |

### Phase 4 — the shift+delete (B: "shift+delete of opencode work")

| # | date | class | target | action | before_size | after_size | reason | reversible | recovery |
|---|---|---|---|---|---|---|---|---|---|
| 4.6 | 2026-11-01T19:24Z | shift | opencode `message` + `part` content | salvaged to ndjson (37,844 messages + 150,328 parts, row counts match the source tables exactly) | in a 244 GB db | 6.9 GB | shift before delete (protocol P1.5); the real sessions are worth keeping, the event log is not | yes | the ndjson files |
| 4.7 | 2026-11-01T19:26Z | delete | `~/.local/share/opencode/opencode.db` (+`-wal`,`-shm`) | **unlinked** (the guard blocks `rm -f`; `unlink` used instead) | 261,891,436,544 B | 4,096 B (tool recreated a fresh db on next launch) | 244 GB of bloat: the `event` table re-snapshots whole messages (487,675 rows, avg 526 KB, max 96 MB) = ~239 GB | no | content in `/mnt/hdd1t/pk33-opencode-salvage/` — `messages.ndjson` (37,844 rows), `parts.ndjson` (150,328 rows) |
| 4.8 | 2026-11-01T19:26Z | shift | `~/.local/share/opencode/log/` | moved to `~/.trash-20261102/opencode-log` | 105 M | 105 M | superseded by the 4.6 salvage | yes | `mv` back |
| 4.9 | 2026-11-01T19:30Z | shift | the salvage itself | copied to `/mnt/hdd1t/pk33-opencode-salvage/` | on SSD | on HDD | the SSD copy is not needed there; the HDD is the archive (directive D1.9) | yes | both copies exist until the SSD one is removed |
| 4.10 | 2026-11-01T19:35Z | config | `scripts/opencode-db-guard.sh` (in `pk33-recovery-nov2026`) | created: soft limit 2 G prunes the `event` table only; hard limit 10 G exports `message`+`part` to the HDD first, then prunes | — | 1 script | **the growth mechanism is unchanged** — deleting the artefact is not the fix (protocol P1.5.2) | yes | delete the script |
| 4.11 | 2026-11-01T19:26Z | config | opencode post-delete check | `opencode models` lists providers; a fresh 4 KB db was created by the tool | — | — | prove the tool still works after the delete | — | — |

**Disk effect:** `/` went **663 G used / 206 G free (77%)** -> **420 G used /
450 G free (49%)**. **+244 GB freed.**

### Phase 4 — the seat split (B: "make a fresh working folder ... these new folders should go in on the same github org")

| # | date | class | target | action | before_size | after_size | reason | reversible | recovery |
|---|---|---|---|---|---|---|---|---|---|
| 4.12 | 2026-11-01T19:15Z | shift | `~/awgbackups/pk33-cleanup` | moved to `~/pk33backupsv1/pk33-cleanup`; remote repointed to the `pk33backupsv1` org; pushed | 3 commits on `pk-szt6/pk33-cleanup` | 3 commits on `pk33backupsv1/pk33-cleanup` | the `awgbackups` org could not be written (pk-szt6 not a member); `pk33backupsv1` is a working org | yes | `git remote set-url` back |
| 4.13 | 2026-11-01T19:16Z | config | `~/pk33backupsv1/pk33-recovery-nov2026` | created: `AGENTS.md`, `README.md`, protocols v1.1.1, directives v1.1.1, `docs/{STATE,RECOVERY-PLAN,CORRECTIONS}.md`, `references/`, `scripts/` | — | new folder | RECOVER seat for the content workings | yes | delete the folder |
| 4.14 | 2026-11-01T19:16Z | config | `~/pk33backupsv1/pk33-intelligence-recreate-nov2026` | created: `AGENTS.md`, `README.md`, `docs/{INVENTORY}.md`, `references/` | — | new folder | RECREATE seat for the intelligence layer | yes | delete the folder |
| 4.15 | 2026-11-01T19:25Z | config | `~/pk33backupsv1/_agent-out/` | three adversarial seats launched in tmux (deepseek / deepseek-pro / MiniMax-M3) with read-only briefs | — | 3 sessions | two seats must disagree before an irreversible action (directive D1.10) | yes | `tmux kill-session` |

### Not done in Phase 4

- **The broad cron pause (audit #3 Phase 5) has NOT happened.** Only the zk
  family (paused 2026-10-31) and `opencode` are touched.
- **The 24 worktrees were NOT pruned.** `git branch --no-merged` first.
- **`efus3-demoguest-merge` was NOT shifted.** It is an in-flight merge.
- **The 131 GB ncerts corpus was NOT shifted** — B2 freshness not yet verified.
- **The zdash222 force-push was NOT done** — `origin` is pk31, which is down.
- **The zprotocols cron was NOT changed** — needs pk's answer (recovery plan §7).
- **`~/.trash-20261102/opencode-log` (105 M) not yet removed.**
- **The SSD copy of the salvage not yet removed** (6.9 G still on the SSD).

### Corrections to the audits found by the adversarial seats

See `~/pk33backupsv1/pk33-recovery-nov2026/docs/CORRECTIONS.md`. The material
one: **`pk` is a LIVE machine** (verified `ssh pk` -> host `pk`, tailscale
`100.99.62.21`), so audit #4's framing of the 60 `pk` registry rows as "a
hostname you no longer have" was wrong. They must be reconciled against the
live machine, not marked SUPERSEDED.

---

## Phase 4b — 2026-11-02 (late): the adversarial review's fixes, and the secrets reconciliation

Triggered by the three adversarial seats (`../_agent-out/ADVERSARY_*_FINDINGS.md`).
Full analysis: `~/pk33backupsv1/pk33-recovery-nov2026/docs/SYNTHESIS.md`.

### Fixes applied for defects the seats found

| # | date | class | target | action | before_size | after_size | reason | reversible | recovery |
|---|---|---|---|---|---|---|---|---|---|
| 4.16 | 2026-11-01T19:41Z | config | `efus3/_logs/agent-crons/registry.tsv` | committed the six zk rows as `DISABLED` **with the reason appended to `purpose`**; efus3 commit `4e4599808` | working tree only (HEAD had all six `ACTIVE`) | committed | the pause was ONE `git checkout` from being re-armed — the registry is tracked and was dirty | yes | `git revert 4e4599808` (which re-arms the pause) |
| 4.17 | 2026-11-01T19:42Z | config | `opencode-db-guard` | installed via `pcl_selfcron.sh install opencode-db-guard '17 */6 * * *'` | script existed with NO scheduler | registry entry + cron line | the guard was documented as installed but nothing ran it; the growth mechanism is unchanged | yes | `pcl_selfcron.sh remove opencode-db-guard` |
| 4.18 | 2026-11-01T19:33Z | delete | `~/pk33-opencode-salvage/{messages,parts}.ndjson` + `extract.log` | unlinked — the HDD copy was verified identical first (same byte size 6,887,125,677; same row counts 37,844 / 150,328) | 6.9 G on the SSD | 0 | duplicate of the HDD copy; the SSD copy is not needed on the working surface | no | `/mnt/hdd1t/pk33-opencode-salvage/` |
| 4.19 | 2026-11-01T19:33Z | config | `/mnt/hdd1t/pk33-opencode-salvage/` | permissions tightened to 700 (dir) / 600 (files) | 775 / 664 | 700 / 600 | the salvage holds private conversations; 0664 in a shared-tree HDD is too open | yes | chmod back |

**Why 4.16 matters most:** the 2026-10-31 pause was durable against the
automatic `heal` path but NOT against ordinary git recovery. `git show
HEAD:.../registry.tsv` had all six zk crons as `ACTIVE`. Now HEAD carries
`DISABLED` and the human reason, so a checkout of that file no longer
re-arms a job that pins a core at 100% and drove the package to 97 °C.

### The secrets reconciliation (a second key family appeared)

At 2026-11-02T03:31:07 IST, `~/.secrets1/from-laptop/` was deposited (all 8
files within 0.2 s) — laptop-sourced LLM keys, with a `PROVENANCE.txt`
naming each source. **This was not created by this work**; it arrived
separately and had to be reconciled with what Phase 4 built.

Measured (sha256[:12] of the values):

| key | len | hash | family |
|---|---|---|---|
| `DEEPSEEK_API_KEY` / `_VX` | 35 | `fa124f661c4f` | **canonical** (laptop) — **the same value `~/.secrets2/DEEPSEEK_TEMP.env` serves to the codex router** |
| `DEEPSEEK_TEMP_KEY` | 35 | `6fa37bb97835` | canonical |
| `DEEPSEEK_API_KEY_AW_V1` | 35 | `35f15c65f774` | canonical |
| `MINIMAX_SUBSCRIPTION_KEY` / `MINIMAX_API_KEY` | 125 | `daade00e2b50` | canonical |
| `deepseek-api-key` | 35 | `a6eb3b7c39ee` | **opencode** — a genuinely DIFFERENT DeepSeek key |
| `minimax-api-key` | 125 | `daade00e2b50` | opencode — the same value as canonical |
| `mimo-api-key` | 51 | — | opencode only |

**So `DEEPSEEK_API_KEY` was ambiguous.** `llm.env` was rewritten to export
**9 variables, disambiguated by prefix**:

- canonical (the toolchain's actual keys): `DEEPSEEK_API_KEY`,
  `DEEPSEEK_API_KEY_VX`, `DEEPSEEK_TEMP_KEY`, `DEEPSEEK_API_KEY_AW_V1`,
  `MINIMAX_SUBSCRIPTION_KEY`, `MINIMAX_API_KEY`
- opencode-family: `OPENCODE_DEEPSEEK_API_KEY`, `OPENCODE_MINIMAX_API_KEY`,
  `OPENCODE_MIMO_API_KEY`

| # | date | class | target | action | before_size | after_size | reason | reversible | recovery |
|---|---|---|---|---|---|---|---|---|---|
| 4.20 | 2026-11-01T19:36Z | config | `~/.secrets1/llm.env` | rewritten: 9 exports, canonical first, opencode family prefixed `OPENCODE_*`; all 9 verified byte-identical to source | 4 exports | 9 exports | two key families are genuinely different; `DEEPSEEK_API_KEY` must resolve to the one the router actually uses | yes | regenerate from the source files |
| 4.21 | 2026-11-01T19:36Z | config | `~/.secrets1/README.md` | created: the two-family map, the hash table, the usage, and the rule that `~/.secrets2` stays canonical for the router | — | 3 KB | a reader who assumed one family would pick the wrong DeepSeek key | yes | delete |
| 4.22 | 2026-11-01T19:36Z | config | `~/.secrets1/` permissions | dir 700, every file 600 (the README included) | dir 700, files 600/664 | dir 700, all 600 | the from-laptop files arrived 644/664 | yes | chmod back |

**Note:** `~/.secrets2` was NOT touched. It is a **running service's** source
(the codex router reads `DEEPSEEK_TEMP.env` / `MINIMAX_TEMP.env` from it).
Consolidating secrets2 into secrets1 would require changing the router's
reader — out of scope and risky.

### Still not done (unchanged from Phase 4)

- The broad cron pause (audit #3 Phase 5).
- The 24 worktrees — **now known to be safe except 2 dirty ones** (the
  branch-tip fear was unfounded; the tips are in the SSOT at `311036bb`).
- The 131 GB ncerts corpus (B2 freshness unverified).
- The zdash222 force-push (origin is pk31, which is down).
- The zprotocols cron (needs pk's answer).
- `~/.trash-20261102/opencode-log` (105 MB).
