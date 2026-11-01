# pk33-cleanup

Audit set + human-typed prompt record for the pk33 cleanup and rework planning, 29–30 Oct 2026.

## What this is

| File | What it covers |
|---|---|
| `pk-load-audit-1-30oct0134hrs.md` | CPU/load assessment — why the box *looked* hot but wasn't CPU-starved |
| `pk-prune-assessment-2-30oct0134hrs.md` | Storage prune + HDD-shift assessment (~460 GB recoverable) |
| `pk-rework-assessment-3-30oct0134hrs.md` | The start point: cron recording, KG rework, finalised-vs-live map, dirty trees, 7-phase plan |
| `pk-observations-gotchas-audit-4-30oct0134hrs.md` | **20 tagged findings for second opinion** — gotchas, risks, stale records, and what was NOT verified |
| `human_typed.md` | Verbatim record of all 9 human prompts + response summaries + intent timeline + what was NOT done |

Read them in order. Audit #4 is written to be read standalone.

## 📍 Repo location — `pk33backupsv1` is the CANONICAL home (pk ruling, 2026-11-02)

> **`pk33backupsv1` is canonical. `awgbackups` is SKIPPED** — do not create,
> push to, or migrate anything there.

This repo was **shifted** from `~/awgbackups/pk33-cleanup` to
`~/pk33backupsv1/pk33-cleanup` and its remote repointed to the
**`pk33backupsv1`** org.

The original `awgbackups` org could not be written:

```
$ gh repo create awgbackups/pk33-cleanup
GraphQL: pk-szt6 does not have the correct permissions to execute `CreateRepository`
```

`awgbackups` exists (created 2026-10-30T21:14:14Z) but has **zero members** —
`pk-szt6` is not one of them. Everything on this box authenticates as
`pk-szt6` (one `gh` token, three SSH keys all resolving to that identity, no
other credentials in env, secret stores, or shell history). The
**`pk33backupsv1`** org works and is where the working folders now live.

**Current remote:**

```
git@github.com:pk33backupsv1/pk33-cleanup.git
```

### The pk33backupsv1 org — the working seat structure

| Repo | What it is |
|---|---|
| [`pk33backupsv1/pk33-cleanup`](https://github.com/pk33backupsv1/pk33-cleanup) | **this repo** — the audit set, the execution ledger, `human_typed.md` |
| [`pk33backupsv1/pk33-recovery-nov2026`](https://github.com/pk33backupsv1/pk33-recovery-nov2026) | **RECOVER** seat — zsm1, zcontent1, zncert-w1, zcons1, the Gitea seat, the fleet. `AGENTS.md`, protocols v1.1.1, directives v1.1.1, `docs/STATE.md`, `docs/RECOVERY-PLAN.md`, `docs/CORRECTIONS.md`, `docs/SYNTHESIS.md`, handover 1.0.0 |
| [`pk33backupsv1/pk33-intelligence-recreate-nov2026`](https://github.com/pk33backupsv1/pk33-intelligence-recreate-nov2026) | **RECREATE** seat — the code KG, ctx packs, project graphs, KG index, embeddings. `docs/INVENTORY.md`, `docs/RECREATE-DESIGN.md`, `docs/THROW-AWAY.md` |

Local paths:

```
~/pk33backupsv1/pk33-cleanup/
~/pk33backupsv1/pk33-recovery-nov2026/
~/pk33backupsv1/pk33-intelligence-recreate-nov2026/
```

### What happened on 2026-11-02

| | |
|---|---|
| **+244 GB freed on the SSD** | 77% -> 49% used; the 244 GB `opencode.db` event log deleted after its 6.9 GB of real content was salvaged to the HDD |
| the credentials survived | `~/.secrets1` created; the opencode credential files became symlinks into it |
| the pause became durable | the six `DISABLED` zk registry rows were **committed** (`efus3` `4e4599808`) — they were previously uncommitted, so one `git checkout` would have re-armed the pause |
| the guard got a scheduler | `opencode-db-guard` installed via `pcl_selfcron.sh` (registry now 104 entries) |
| three seats attacked the plan | and **found three audit errors and three missed problems** — see `pk33-recovery-nov2026/docs/SYNTHESIS.md` |

## Scope and safety

**No changes were made to the audited system.** Across the whole session:

- no cron disabled, enabled, healed, or edited (`pcl_selfcron.sh` was run with `list` and `verify` only — both read-only)
- no file deleted, moved, or renamed on the audited system (the 244 GB `opencode.db` is still present)
- no git commit/branch/stash/reset in any of the ten audited repos (read-only git commands only)
- no service stopped or reconfigured; k3s, docker, and all containers left running
- no config file edited (`/etc/fstab`, crontab, systemd units untouched)

The only writes were these five documents, this repo, and one throwaway test repo that was created during diagnosis and immediately deleted (`pk-szt6/pk33-cleanup` — recreated properly as this repo).

## The four documents in brief

**Audit #1 — load.** The box was 85–90% idle; load average 24–28 was process churn, not saturation (zero D-state throughout). Two hot Python processes root-caused to `intake.append()` rescanning the full 91 MB `rc/intake.ndjson` per event — measured 10.0 s for 10 appends. Docker healthchecks boot a full BEAM VM every 10–15 s (caught at 244% CPU, 128 `erts_sched` threads, ~70k ctx switches/sec). 32 `idle_inject` kernel threads burn ~1 core. k3s crash-looping, 62–73 restarts per pod.

**Audit #2 — prune/shift.** ~460 GB prunable, ~389 GB pure bloat. Largest: a 244 GB SQLite event log in an inactive tool (event table ~239 GB; `message.updated.1` rows average 526 KB, max 96 MB). Second: 131 GB ncert zip corpus, 1,025 files, no live consumer. The HDD bind-mount pattern is **already proven in production** — `ssot_engine_efus3/{dc,rc}` are already on it via `/etc/fstab`.

**Audit #3 — the start point.** The cron-recording mechanism **already exists**: `pcl_selfcron.sh disable` preserves the registry row as `state=DISABLED`. The real CPU hog is the KG rebuild — `graphify` at 100% with the CPU package at **89–100 °C** (crit 100), graphifying the Flutter SDK; a hand-written pause note from **2026-10-14** shows it was paused for exactly that reason and **later reverted**. zsm1 is **finalised** (re-ran its checkers: Indianization 0 faults, depth 1961/1961, boundary 4/4, git clean) — its `RESUME-HERE.md` is 2 months stale. ~250 of ~893 dirty tree entries are cron churn, not work.

**Audit #4 — gotchas.** 20 findings tagged [OBS] / [GOTCHA] / [RISK] / [STALE] / [UNVERIFIED], each with a "second opinion wanted on" question, plus an explicit *method, limits, and what I did NOT verify* section. Newly caught in that pass: the opencode dir holds **4 live API credential files** (so the delete target is narrower than audit #2 said); `disable zk-index` would **silently also drop** `zk-index-zdash222` (a `grep -vF` substring collision the shrink guard permits); the zprotocols ctx files are **cron-staged but never committed**; and `GRAPHIFY-PAUSED-2026-10-14.md`'s restore command would **re-enable three jobs** if run today.

---

*Created 2026-10-30 by pk-szt6 on pk33. Measurement was read-only throughout.*
