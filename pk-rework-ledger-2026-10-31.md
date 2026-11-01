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

---

## Phase 4c — 2026-11-02: the intake append fix (the audit #1 CPU hog, actually fixed)

### The canonical home (pk ruling)

> **`pk33backupsv1` is the CANONICAL GitHub home for pk33's seat work.
> `awgbackups` is SKIPPED** — do not create, push to, or migrate anything there.

`awgbackups` has zero members and `pk-szt6` is not one, so it cannot be written
from this box (`CreateRepository` refused). It was created 2026-10-30T21:14:14Z
— 16 minutes before the first attempt — and has never been populated.
`pk-szt6/pk33-cleanup` is a **stale mirror** from 2026-10-30/31; it is superseded
by `pk33backupsv1/pk33-cleanup` and should not be pushed to.

Recorded in: both seats' `AGENTS.md` §0, and `pk33-cleanup/README.md`.

### The fix

Audit #1 identified the defect (a full-log rescan per append) but did not fix it.
On 2026-11-02 it was **actively burning a core every 10 minutes at 96 °C**.

| | |
|---|---|
| Root cause | `intake.append()` computed the next seq with `_scan_max_seq()` — a full read of the log. `append_many()` called `append()` per event, so a 76-event feed rescanned a 93 MB log **76 times**. The rc log is on an **HDD bind-mount**, so each scan was also a spin-disk read. |
| zcons1 commit | `944d51b` (pushed to `zexp1/zcons1`) |
| efus3 pointer | `ccec0c63e` |

**Measured on the real 93 MB rc log, old vs new, same input:**

    OLD: append_many(76 events) = 34.07 s
    NEW: append_many(76 events) =  0.00 s

**Output byte-identical** — 226,362 lines compared one by one: same seqs, same
kinds, same payloads, same provenance.

**Two changes, both keeping the ONE-WRITER LAW intact:**

1. `_tail_max_seq()` — the intake log is append-only and never rewritten, and
   every append takes the flock, so seqs increase **in file order** and the last
   parseable line carries the max. Reading the last 64 KB gives the same answer
   at O(tail). It is used **only when it proves the answer**: it returns `None`
   on a malformed or unordered tail, and the caller then pays the full scan.
   Correctness never depends on the heuristic.
2. `append_many()` scans **once** for the whole batch and assigns
   base+1, base+2, ... under the same flock the single append takes. The
   invariant is unchanged — seqs strictly increase, assigned and written under an
   exclusive lock, so a concurrent appender still observes every prior append
   and still gets a strictly larger seq. A batch is now **atomic** where it
   previously was not, which is strictly stronger.

**Tests:** `ssot_engine/tests/test_intake_append_perf.py` (new, 6 tests) pins the
batch semantics, the one-scan-per-batch property, tail/full-scan agreement, the
malformed-tail refusal, and atomicity. Suite: **22 passed** (16 pre-existing +
6 new).

**The wider `tests/` suite is unchanged by this commit** — 42 failed / 241 passed
**both before and after**. The failures are pre-existing and unrelated; verified
by re-running with the original `intake.py` restored and getting the identical
counts.

**Live proof (the same workload, one hour apart):**

| | tick | rc log written | elapsed |
|---|---|---|---|
| old code | 03:48:06 | 03:48:46 | **40 s** |
| fixed code | 03:58:03 | 03:58:03 | **< 1 s** |

The log grew by exactly 76 events (seq 226286 → 226361), strictly increasing,
unique, no collision. **CPU package: 96 °C → 34 °C.**

### A historical defect found while verifying (NOT caused by this fix)

The rc log contains **2 duplicate seqs**: 637 and 638, each appearing twice, from
**2026-07-02** — nine hours apart (`10:27` and `19:50`), both from
`emitter:iops_deploy_dev`. That is the old `append()` racing under the flock
**before** the current locking discipline, or a pre-flock writer. It is **three
months old** and predates every change in this ledger.

**Not fixed here** — the drain handles duplicates by checkpoint, and repairing
historical seqs in a durable log is a separate, deliberate operation. Recorded
as an open item.

### The `pk` registry reconciliation

Seat B found `pk` is a **live machine** (audit #4 had called it "a hostname you
no longer have"). Reconciled rather than discarded:

| # | date | class | target | action | reason | reversible | recovery |
|---|---|---|---|---|---|---|---|
| 4.23 | 2026-11-01T20:05Z | config | `pk`'s own registry | 8 rows `DEAD-UNTRIAGED` → `ACTIVE`; pk commit `da607cb44` | all 8 are live in pk's crontab and their commands resolve to real paths on that host. `DEAD-UNTRIAGED` means "a human ruled on this, stop re-alarming" — no such ruling applies, they are running. | yes | `git revert da607cb44` |
| 4.24 | 2026-11-01T20:06Z | config | pk33's copy of the shared registry | the same 8 rows; pk33 commit `36804eccb` | the registry is shared — a stale copy re-seeds the wrong state | yes | `git revert 36804eccb` |
| 4.25 | 2026-11-01T20:08Z | config | the 6 pk rows marked `DISABLED` | **verified, no change** | all 6 are genuinely absent from pk's crontab — those labels stand | — | — |

**Also found on `pk` (not fixed from pk33 — it is pk's own estate):** 28 jobs run
on pk but are registered under machine `pk33`, so pk's `verify` cannot see them
and `heal` cannot repair them. Their commands point at `/home/pk/`, so they are
pk's own jobs, misfiled. Plus 57 `#PAUSED20260711` comment lines — the residue of
the July runaway, **stable** (111 → 111 lines over 60 s; the `heal` guards are
present). Full analysis: `~/pk33backupsv1/pk33-recovery-nov2026/docs/FLEET-AUDIT.md`.

### B2 freshness — the ncerts corpus IS redundant

Verified before treating the 131 GB local corpus as prunable:

    local  /home/pk33/zsm-ncert-20260919/ncert-pdfs : 1,025 files, 140,599,609,820 bytes
    B2     b2-remote:zsm-ncert/ncert-pdfs-20260919   : 1.025k objects, 140,599,609,820 bytes

**Byte-identical.** The corpus is safely prunable or shiftable — B2 is a live
off-box tier (unlike `aw:~/awgbackups`, which is **empty**).

### Not done

- **The 131 GB ncerts shift/delete** — verified redundant against B2, but the
  action itself is pk's call (shift to HDD vs delete outright).
- **The 22 clean worktrees** — now known to be safe (their tips are in the SSOT
  at `311036bb`), but the **2 dirty ones** must be committed or archived first
  (26 files, incl. `packages/api/src/durable-learn-store.ts`).
- **The broad cron pause**, the **zdash222 force-push** (origin is pk31, down),
  the **zprotocols cron** (needs pk's answer), and **pk's 28 mis-scoped rows**
  (pk's own estate).
- **The historical duplicate seqs 637/638** in the rc log.

---

## Phase 4d — 2026-11-02: everything committed and pushed

Final state of the push. Other machines (pk31, pk32, aw) are switched off, so
their remotes were not touched — noted, not worried about.

| # | date | class | target | action | reason | reversible | recovery |
|---|---|---|---|---|---|---|---|
| 4.26 | 2026-11-01T20:30Z | commit | `zexp1/efus3` | rebased 5 local commits onto `origin/main` and pushed: `5feadb128..04442a420` | the remote had moved (10 commits — the demoguest-merge superset); the rebase conflicted in `registry.tsv` | yes | `backup/pre-push-20261102-final` (local) + `git revert` |
| 4.27 | 2026-11-01T20:32Z | config | `efus3` registry — conflict resolution | took **my** side: the six zk rows stay `DISABLED` with the reason; the remote's 4 dropped pk33 rows were **restored** | the remote still had all six zk crons `ACTIVE` — taking the remote side would have re-armed the pause. The 4 rows the remote dropped were re-added (the conflict was a row-range overlap, not a real deletion) | yes | the commit is in the rebase history |
| 4.28 | 2026-11-01T20:35Z | config | `efus3` — `skills/afed-bfed/SKILL.md` | local edit **superseded** by the remote version; the local text was confirmed already relocated to `zdash22x/zbfed/docs/protocols/BOT-FIRST.md` | the remote moved the protocol to a dedicated kit — the local copy was the older form of the same content | yes | `stash@{1}` still holds the local edit |
| 4.29 | 2026-11-01T20:40Z | commit | `zdash22x/zdash222` | published the 82-commit rebase-repair line to the GitHub mirror as branch **`recovery/2026-10-31-rebase-repair`** — NOT to `main` | `main` on the mirror is 2 commits behind and a merge would need `--force-with-lease`; that is **pk's open decision #4**, so it was not taken unilaterally. A branch is durable and non-destructive | yes | `git push origin --delete recovery/2026-10-31-rebase-repair`; both tips also on `backup/pre-rebase-main-2026-10-31` and `backup/detached-line-2026-10-31` |
+| 4.30 | 2026-11-01T20:45Z | commit | `pk33backupsv1/pk33-cleanup` | added `references/agent-out/` — the canonical originals of the three seats' briefs + findings | the briefs were authored outside any repo; copies were already in both seat repos | yes | `git rm -r references/agent-out` |

### Push ledger — what is on GitHub

| repo | head | what |
|---|---|---|
| `pk33backupsv1/pk33-cleanup` | `85234ff` | audits #1–#4, the ledger (Phases 0–4d), `human_typed.md`, agent-out originals |
| `pk33backupsv1/pk33-recovery-nov2026` | `56dfa10` | RECOVER seat: AGENTS, protocols v1.1.1, directives v1.1.1, STATE, RECOVERY-PLAN, CORRECTIONS, SYNTHESIS, FLEET-AUDIT, handover 1.0.0 + 1.1.0, worktree-salvage |
| `pk33backupsv1/pk33-intelligence-recreate-nov2026` | `7225479` | RECREATE seat: AGENTS, INVENTORY, RECREATE-DESIGN, THROW-AWAY, references |
| `zexp1/efus3` | `04442a420` | the ledger, the registry fix (zk `DISABLED` + pk reconciled), the zcons1 pointer |
| `zexp1/zcons1` | `944d51b` | **the intake append fix** — 34.07 s → 0.00 s per feed |
| `zexp1/zsm1` | `311036bb` | SSOT, unchanged; plus `archive/wip/pk33/*` = the salvaged worktree tips |
| `zdash22x/zdash222` | `recovery/2026-10-31-rebase-repair` | the 82-commit rebase-repair line (not on `main` — pk's decision) |

### Left deliberately uncommitted (all live cron churn, not work)

| Repo | dirty | what |
|---|---|---|
| `efus3` | 211 | `_logs/**` ndjson, `pk-console` pulse files, `ssot_engine_efus3/{dc,rc}/intake.ndjson` — rewritten every cycle |
| `zcons1` | 10 | the zcons **self-spine** live state (`intake.ndjson`, `spine.sqlite`, `projection.json`, `_prober/`, `coverage/`, `shr/logs/`) |
| `zcontent1` | 621 | 562 `docs/history/CURRENT/*.tsv.gz` + embed logs |
| `zdash222` | 65 | `_logs/gates/**` + registers |

Committing these would re-dirty the tree within one cron cycle — the reason the
broad cron pause (RECOVERY-PLAN §3) is the step that makes the git cleanup
meaningful.

### One deliberate non-push

`zsm1-tzerp` (the read-only archive) has 1 unpushed commit — the
`README: read-only archive notice` — and its GitHub remote is **archived and
read-only** (403). The commit is retained locally and the same notice is in the
SSOT's README (`efus3/zsm1`, "SSOT notice (2026-09-16)"), so nothing is lost.

### Not touched (other machines switched off)

- `pk31` — down 4d. Hosts the primary Gitea (the `origin` for efus3/zsm1/
  zcontent1/zdash222) and the bge-m3 GPU/embed box. **The GitHub mirrors
  covered for it**, so the audit + recovery work was publishable anyway.
- `pk32` — down 23d.
- `aw` — down 86d on the tailnet; `~/awgbackups` is empty (no off-box tier there).
