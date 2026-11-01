# ADVERSARY A — pk33 rework attack

**As of:** 2026-11-02 ~21:59Z / ~03:29 IST  
**Scope:** adversarial verification of the claims in the cleanup/rework plan. No cleanup, move, deletion, commit, or cron change was performed. The requested findings file is the only file created. The zsm1 checkers were invoked without their optional output-writing flags. `pcl_selfcron.sh verify` has a built-in telemetry call at `scripts/pcl_selfcron.sh:2`; no cron or registry write was performed by me.

The box is changing during this review. The 244 GB database was deleted/recreated and the credential rebase happened before or during this pass. Findings below separate the historical 2026-10-30 audit state from the live 2026-11-02 state.

## REFUTED

### 1. “zsm1 is FINALISED and GREEN” is false as an unqualified release verdict

The audit states this verdict at `pk-rework-assessment-3-30oct0134hrs.md:17` and `:313`. Its own evidence table immediately contradicts the stronger wording: `:300-303` records Hindi as **RED / 99.3%** and the test suite as **not re-run**. `efus3/zsm1/RESUME-HERE.md:14` also names Hindi debt as unfinished.

I re-ran all four requested checkers from `/home/pk33/zexp1/efus3/zsm1`:

```bash
PYTHONDONTWRITEBYTECODE=1 python3 tools/kcgraph/depth_status.py
PYTHONDONTWRITEBYTECODE=1 python3 tools/kcgraph/indianization_status.py -v
PYTHONDONTWRITEBYTECODE=1 python3 tools/kcgraph/g6_hindi_status.py -v
PYTHONDONTWRITEBYTECODE=1 python3 tools/kcgraph/boundary_gate.py -v
git status --short
git rev-parse HEAD
git rev-parse origin/main
```

Results:

- depth: `1961/1961`, exit 0.
- Indianization: `0 faults`, 30,164 artefacts across 149 strands, exit 0.
- Hindi/G6: **exit 1**, `T1 cards: 144/145`, `evs.g3: MISSING`, `G6-100: RED — not yet`.
- boundary: L1–L4 PASS, 149 strands / 1,976 KCs / 4,801 bindings, exit 0.
- Git: clean; `HEAD` and `origin/main` are both `311036bb8339451c610aae0526dca69c8ca1f1d7`.

ZNCERT Stage 3 is also explicitly open: `docs/m216-corpus-ingestion/zncert-sync/STAGE3-PLAN.md:12` gives 7,995 genuine chapter rows, `:15` gives `CLEAN 2,144 / AMBIGUOUS 5,786 / NO-ZSM-STRAND-YET 217`, and `:44` says ratification is `0 of 2,208`.

**Correct verdict:** zsm1 is a clean, pushed, stable SSOT baseline with three strong green gates and a bounded open Hindi/Stage-3 tail. It is not fully complete. A pk-facing reviewer would still want:

1. an explicit ruling on the missing `evs.g3` Hindi card strand;
2. a fresh full `pnpm test` result, because the audit admits it was not run;
3. a disposition for the 5,786 AMBIGUOUS mappings and 217 no-strand rows;
4. a scope statement if “finalised” means “no more work on this baseline” despite G6-100 remaining RED.

### 2. The 24 worktrees do not carry unmerged branch tips, and removing them does not delete branches

The plan implies that the worktrees expose unmerged `wip/pk33/*` work (`pk-observations-gotchas-audit-4-30oct0134hrs.md:88-92`, `:229-250`).

Commands run in `/home/pk33/tzerp/zsm1-tzerp`:

```bash
git branch --no-merged main
git merge-base --is-ancestor <each-worktree-HEAD> main
git -C /home/pk33/zexp1/efus3/zsm1 merge-base --is-ancestor <each-worktree-HEAD> 311036bb
```

`git branch --no-merged main` lists only six refs: five `backup/*` refs pointing at `ba1baf21` and `wip/pk33/vps-deploy-harness`. None of the 24 child worktree branches appears. All 24 child tips are ancestors of the archive’s `main` **and** are present and ancestral in the SSOT `efus3/zsm1` at `311036bb`.

`git worktree remove <path>` removes the checkout and its administrative metadata; it does not delete the branch ref stored in the common `.git`. Therefore the committed 24 tips are not the loss risk. A blind directory deletion or `--force` remove is different.

The real risk is two dirty worktrees:

- `agent-a25343b681975143d` on `wip/pk33/durable-learn-store`: 20 status entries; 15 tracked modifications and 5 untracked files. Across those paths, 14 differ from the SSOT, 3 do not exist in the SSOT, and 3 already match it. The unique untracked work includes `packages/api/src/durable-learn-store.ts`, `packages/api/src/learn-persistence.ts`, `packages/api/test/durable-learn.test.ts`, and two `_lanes/durable-learn-store/` files.
- `agent-a7c1b95e55c30d315` on `wip/pk33/api-v1-wave3`: one modified file, `docs/m217-zsm-curriculum/assessment-simulation-report.md`, with 8 insertions and 8 deletions.

Command:

```bash
for wt in $(git worktree list --porcelain | awk '/^worktree /{sub(/^worktree /,""); print}'); do
  git -C "$wt" status --porcelain=v1 --untracked-files=all
done
```

**Correct action:** the 22 clean child worktrees are prunable after confirming their branch refs remain; preserve or commit the two dirty worktrees first. Do not use “unmerged branch tips” as the justification.

### 3. “`graphify-out` is untracked in every repo” is false

The claim is at `pk-prune-assessment-2-30oct0134hrs.md:59-65`.

Commands:

```bash
find /home/pk33 -xdev -type d -name graphify-out -exec du -s -B1 {} +
find /home/pk33 -xdev -type d -name graphify-out -print | while read d; do
  git -C "$(dirname "$d")" ls-files -- "$(basename "$d")"
done
```

There are **53** `graphify-out` directories, not 43. Two have tracked files:

- `Desktop/notespk/npk003-skills-hooks-etc/npk0031-claude-work-backup-skills-hooks-etc-20261009/claude-home/graphify-out`
- `Desktop/notespk/npk003-skills-hooks-etc/npk0032-claude-baseline-skills-hooks-etc/claude-home/graphify-out`

Each has 28 tracked paths and occupies 520,192 bytes. Deleting them will dirty the notespk repository even though Git can restore them. The main SSOT’s `.gitignore:95-96` and `scripts/zfts.py:43` do support “derived” for that repository, but that evidence was incorrectly generalized to every directory.

The efus3 snapshot count has also moved: there are now 19 dated directories, including `2026-10-30`, not only the 18 dated Sep 16–Oct 13. Command:

```bash
find /home/pk33/zexp1/efus3/graphify-out -mindepth 1 -maxdepth 1 \
  -type d -name '2026-*' | wc -l
```

### 4. “244 GB is reclaimable” is no longer a pending action

The original audit cited `261,891,379,200` bytes at `pk-prune-assessment-2-30oct0134hrs.md:27`, which is **243.90 GiB** (261.89 decimal GB). I cannot re-stat that historical file because it has already been deleted; repeated source records and the current disk delta corroborate that deletion. Current read-only measurement:

```bash
stat -c '%s %y %n' /home/pk33/.local/share/opencode/opencode.db
du -s -B1 /home/pk33/.local/share/opencode
du -s -B1 /mnt/hdd1t/pk33-opencode-salvage
du -s -B1 /home/pk33/pk33-opencode-salvage
df -hT /
```

Current results:

- fresh `opencode.db`: 249,856 bytes;
- entire opencode state directory: 164,737,024 bytes;
- SSD root: 420 GiB used, 450 GiB available;
- HDD salvage: 6,887,141,376 bytes;
- duplicate SSD salvage: 6,887,141,376 bytes.

The original reclaim has happened. Repeating “reclaim 244 GB” in the active plan is stale. The residual SSD opportunity is the duplicate 6.41 GiB salvage tree, subject to retaining the verified HDD copy.

Also, audit #2’s instruction to delete the **whole opencode directory** (`:55`, `:321`) was unsafe as written because the directory contained credentials. Audit #4 corrected the target at `pk-observations-gotchas-audit-4-30oct0134hrs.md:103-117`.

### 5. “`~/.secrets3` is names only” is false

The current recovery guide says `.secrets3` is names-only (`pk33-recovery-nov2026/AGENTS.md:152`; `docs/STATE.md:158`).

Read-only inspection found actual populated values, without printing them:

```bash
awk -F= '/^[[:space:]]*[A-Za-z_][A-Za-z0-9_]*=/{n++; if(length($0)>index($0,"=")) v++} END{print n,v}' \
  /home/pk33/.secrets3/zdvc/90-generated/references/zmuxpk7_llms_api_keys.env
stat -c '%a %U:%G %n' \
  /home/pk33/.secrets3 \
  /home/pk33/.secrets3/zdvc/90-generated/references/zmuxpk7_llms_api_keys.env
```

The inventory file has 27 nonempty assignments and mode `0664`; its directories are mode `0775`. The duplicate `pk-doppler-vault-records/MINIMAX.env` is also populated and mode `0664`. The home directory is mode `0750`, so this is not internet-readable, but the file-level permissions are not consistent with a secrets store.

The same read-only check found actual values in:

- `~/.secrets2/{MMX,CURSOR,MINIMAX,DEEPSEEK,B2}_TEMP.env`;
- `~/.secrets2/digitalocean-eduz1.env`;
- `~/.secrets3/r2-backup.env`;
- `~/.secrets3/zdvc/90-generated/references/zmuxpk7_llms_api_keys.env`;
- the `.secrets3/zdvc/99-salvage` provider records.

### 6. The opencode growth guard is not installed as automation

`pk33-recovery-nov2026/AGENTS.md:135` summarizes the guard as “installed,” but `docs/RECOVERY-PLAN.md:64` says only “written and dry-run,” and `:70-71` still lists scheduling it as TODO.

Read-only checks found a script but no execution path:

```bash
rg -n 'opencode' /home/pk33/zexp1/efus3/_logs/agent-crons/registry.tsv
crontab -l | rg -i opencode
systemctl --user list-timers --all --no-pager | rg -i opencode
systemctl list-timers --all --no-pager | rg -i opencode
```

All four produced no opencode match. A fresh DB already exists at 249,856 bytes, and line 8 of `scripts/opencode-db-guard.sh` itself says the growth mechanism is unchanged. Without scheduling, deletion did not fix recurrence.

### 7. The NCERTS corpus has 28 top-level language directories, not 29

Command:

```bash
find /home/pk33/zsm-ncert-20260919/ncert-pdfs -mindepth 1 -maxdepth 1 \
  -type d -printf '%f\n' | sort
```

There are 28. Three alias pairs inflate the apparent language count: `gujarati/gujrati`, `maithili/maithli`, and `odia/oriya`; there is also `unknown`. The “1,025 zip files” count is correct.

## OVERSTATED

### Storage units and “GB” labels

Several durable GiB figures are described as decimal GB without distinguishing the units:

- opencode: 261,891,379,200 bytes = 243.90 **GiB**, not 244 decimal GB.
- NCERTS: 140,599,609,820 logical bytes = 130.94 **GiB** / 140.60 decimal GB; “131” is a good rounded GiB figure.
- graphify outputs: 8,041,869,312 bytes = 8.04 decimal GB / 7.49 **GiB**; “8.0 GB” is correct only in decimal units.
- worktrees: 7,370,317,824 bytes = 7.37 decimal GB / 6.86 **GiB**; “6.9” is a good rounded GiB figure.

Commands:

```bash
stat -c '%s' <file>
find ... -exec du -s -B1 {} +
du -s -B1 .claude/worktrees
find /home/pk33/zsm-ncert-20260919/ncert-pdfs -type f -printf '%s\n' |
  awk '{s+=$1;n++} END{print n,s,s/1073741824}'
```

### “No live consumer” for NCERTS is true but not sufficient to justify a shift

No cron line, systemd timer, or process currently references the local corpus. Commands returned no matches:

```bash
crontab -l | rg -i 'zsm-ncert|ncert-refresh|vlm-bakeoff|ncert-pdfs'
systemctl list-timers --all --no-pager | rg -i 'ncert|zcontent|vlm'
ps -eo pid,cmd | rg -i 'zsm-ncert|ncert-refresh|vlm-bakeoff|ncert-pdfs'
```

However, the local copy is not a one-of-a-kind cache that merely needs archiving. Current B2 read-only inspection gives the same object count and payload:

```bash
find /home/pk33/zsm-ncert-20260919/ncert-pdfs -type f -printf '%s\n' |
  awk '{s+=$1;n++} END{print n,s}'
rclone size b2-remote:zsm-ncert/ncert-pdfs-20260919 --json
```

Both report 1,025 objects and 140,599,609,820 bytes. The corpus refresh summary (`zcontent1-ncert-refresh/docs/ncert-refresh/CORPUS-20260919-SUMMARY.md`) records three independently checked copies. A move to HDD conserves a local copy, but is not the least-space action. After confirming B2 retention and the manifest, the SSD copy can be removed rather than duplicated onto HDD.

### “All six zk jobs are DISABLED” is operationally true, but the pause is not durable

The pause currently holds, but not at the level the words imply. See MISSED §1.

### The 24-worktree average is a rough estimate

`7,370,317,824 / 24 = 307,096,576` bytes per worktree, not the stated “~290 MB each.” This is immaterial to the reclaim total.

### The opencode directory is “inactive,” not inert

There is no running process or service now, but the binary is still at `/home/pk33/.opencode/bin/opencode` and on `PATH`, the credential symlinks remain, and a fresh DB was already recreated. An inactive tool can regenerate state again; only disabling or guarding it makes that non-recurring.

## CONFIRMED

### Historical opencode deletion and salvage

The 244 GiB-ish DB was real and deleting it was the dominant space win. The source records the old object at `261,891,379,200` bytes (`pk-prune-assessment-2-30oct0134hrs.md:27`) and the deletion at `pk33-recovery-nov2026/docs/STATE.md:25-56`.

The meaningful session data was retained:

- `messages.ndjson`: 6,484,634,248 bytes;
- `parts.ndjson`: 402,491,148 bytes;
- HDD and SSD copies are both 6,887,141,376 bytes.

This preserves the real content without preserving the event-table bloat. The unresolved defect is scheduling the guard, not the historical deletion.

### NCERTS size and redundancy are real

The local SSD copy is 1025 files / 140,599,609,820 logical bytes and B2 currently exposes the identical count and byte total. The code references `/mnt/wdhdd/zsm-ncert/...` in the bakeoff tools, not `/home/pk33/zsm-ncert-20260919`, and the two mining repositories have no active schedule or process.

This confirms that moving or deleting the local copy will not break a currently running job. It does not excuse dropping the manifest and checksums.

### Graphify output is largely derived, even though the universal claim was too broad

The measured total is 8,041,869,312 bytes across 53 directories. In efus3, `.gitignore:95-96` explicitly excludes `graphify-out`, and `scripts/zfts.py:43` classifies it as a derived code graph. `scripts/gen_mem.sh:80-86` consumes graph output if it already exists, but `--fast` documents skipping the rebuild.

The material outputs can be regenerated with `graphify update`. The two tracked notespk copies need a normal `git status`/restore path, and retaining the newest snapshot per active repo would be more conservative than deleting all 53.

### The 24 worktrees occupy 6.86 GiB

Both facts survive:

```bash
du -s -B1 .claude/worktrees
find .claude/worktrees -mindepth 1 -maxdepth 1 -type d | wc -l
```

Measured: `7,370,317,824` bytes and 24 directories. All committed tips are safe in the archive main and SSOT. Only the two dirty checkouts are unique work.

### Three zsm1 gates are genuinely GREEN

- `depth_status.py`: 1961/1961 period-ready.
- `indianization_status.py`: 0 faults over 30,164 artefacts.
- `boundary_gate.py`: L1–L4 all PASS.
- Git is clean and the local main commit exactly matches `origin/main`.

These are real strengths. The objection is against summarizing the entire project as complete while `g6_hindi_status.py` exits nonzero and Stage 3 remains open.

### The zk cron pause currently prevents automatic heal resurrection

Current read-only checks:

```bash
bash scripts/pcl_selfcron.sh verify
awk -F '\t' '$1=="pk33" && $2 ~ /^zk-/{print $2,$8}' \
  _logs/agent-crons/registry.tsv
crontab -l | rg 'pcl-selfcron:zk-|graphify|zk_graph|zskill zk'
```

All six relevant registry states are `DISABLED`; no live tagged zk line exists. `verify` reports `103 registered, 1 dead`; the dead row is unrelated `zdash222-wifi-5g-guard`.

`scripts/pcl_selfcron.sh:233-240` skips `DISABLED` rows in `heal`. `zfable/ops/warden.sh:20-27` calls `verify`, then `heal` when any dead row exists; because `zdash222-wifi-5g-guard` remains dead, heal is still being called every five minutes, and it still skips the six zk rows. The mechanism was exercised in the relevant failure mode and did not resurrect them.

### The credential rebase improves the four moved files’ access control

Current measurement:

```bash
stat -c '%a %U:%G %n' /home/pk33/.secrets1 \
  /home/pk33/.secrets1/{llm.env,opencode_auth.json,minimax-api-key,deepseek-api-key,mimo-api-key}
namei -l /home/pk33/.secrets1
```

`~/.secrets1` is mode `0700`; every file in it is mode `0600`; its parent chain is sane. The opencode-facing four entries are symlinks to those files. This is directionally better than leaving values mixed into a mode-0775 application data directory.

The read-only comparison confirms:

- the minimax value in `.secrets2/MINIMAX_TEMP.env` equals `.secrets1/minimax-api-key`;
- the deepseek value in `.secrets2/DEEPSEEK_TEMP.env` differs from `.secrets1/deepseek-api-key`.

So the rebase improved the four files’ immediate permissions. It did not produce a single secure secrets store, revoke any old copies, encrypt values at rest, or eliminate duplicate provider credentials.

## MISSED

### 1. The DISABLED state is uncommitted, so ordinary Git recovery can re-enable all six jobs

This is the most serious pause defect.

Command:

```bash
git diff -- _logs/agent-crons/registry.tsv
git show HEAD:_logs/agent-crons/registry.tsv |
  awk -F '\t' '$1=="pk33" && $2 ~ /^zk-/{print $2,$8}'
```

The working registry has all six as `DISABLED`. The committed `HEAD` version has all six as `ACTIVE`. The registry is tracked and currently dirty. Any of these can undo the pause:

1. `git checkout -- _logs/agent-crons/registry.tsv` or an equivalent hard reset/restore;
2. restoring `~/registry.pre-rework.tsv`, which also contains all six as `ACTIVE`;
3. a rebuild from the current committed registry followed by `pcl_selfcron.sh heal`;
4. an explicit `pcl_selfcron.sh enable <name>` (`scripts/pcl_selfcron.sh:331-390`);
5. manually invoking the zk binaries outside cron, which the pause does not prohibit;
6. restoring the saved crontab: `~/crontab.pre-rework.20261031T215718Z.txt` still contains two live zk lines and four paused copies;
7. running the old hand-written restore command at `~/GRAPHIFY-PAUSED-2026-10-14.md:21`, which uncomments the four graphify-paused lines.

**Verdict:** `DISABLED` is durable against the current automatic `heal` path, but it is not yet durable against normal repository recovery or the saved backups. Commit the registry state with the human reason, and treat any registry restore as a cron-state restoration requiring `verify`.

### 2. The opencode growth guard is documented as “installed” but has no scheduler

This is a direct recurrence risk and a documentation contradiction. The script exists and is sensible (`pk33-recovery-nov2026/scripts/opencode-db-guard.sh:29-66`), but `crontab`, systemd timers, and the selfcron registry contain no reference to it. The plan still lists scheduling it as TODO (`docs/RECOVERY-PLAN.md:66-72`). The fresh DB has already been recreated. The next disk-full event remains possible.

### 3. There are two 6.41 GiB opencode salvage copies

The docs say the content was moved to HDD, but both `/mnt/hdd1t/pk33-opencode-salvage` and `/home/pk33/pk33-opencode-salvage` exist at exactly 6,887,141,376 bytes. The recovery plan itself still lists removal of the SSD copy as TODO (`docs/RECOVERY-PLAN.md:67-69`). After comparing row counts and hashes, this is a straightforward additional SSD reclaim.

The HDD salvage files are mode `0664` inside a mode `0775` directory. They contain six-plus GB of prior conversations, not merely harmless logs. Their retention and access controls deserve an explicit decision.

### 4. The worktree hazard is dirty files, not unmerged refs

The actionable split is:

- 22 clean worktrees: prune normally;
- `durable-learn-store`: preserve/commit/test 14 SSOT-different paths plus 3 SSOT-missing files;
- `api-v1-wave3`: preserve/commit an 8-line report edit.

All 24 committed branch tips are already in the SSOT. A reviewer who follows the audit’s framing could delete the wrong material or preserve 6.8 GB of clean checkout copies unnecessarily.

### 5. NCERTS is already redundant in B2, so a 131 GiB HDD copy is not the best action

Current B2 reachability and exact object-size agreement were not checked by the audit. The local files are byte-for-byte representable from the remote corpus today. If policy accepts B2 as an archive, delete the SSD copy after checksum comparison or retain only a small manifest. Moving 131 GiB to the HDD saves SSD space but spends HDD space on duplicate data.

### 6. The “secrets1” consolidation did not retire the older credential stores

The box still has:

- `~/.secrets2/*_TEMP.env` with populated router/provider/B2/DO credentials;
- `~/.secrets3` with populated generated provider/env records, contrary to its names-only description;
- four SSH private keys under `~/.ssh`;
- GitHub auth in `~/.config/gh/hosts.yml`;
- B2 auth in `~/.config/rclone/rclone.conf`;
- Google credentials in `~/.config/gcloud/credentials.db`;
- Claude auth in `~/.claude/.credentials.json`;
- Cursor auth in `~/.config/cursor/auth.json`;
- Codex/chatgpt/router configs under `~/.codex`;
- Gitea’s private key at `~/gitea/data/gitea/jwt/private.pem`;
- multiple project `_local/*/.env.local` files, mostly mode `0600`;
- `.bash_history`, mode `0600`, containing secret-variable names and potentially command-line values.

The `.secrets1` permissions are better, but the overall posture is not materially improved until duplicate values are revoked/deduplicated and the `.secrets3` generated files are either migrated or tightened. The minimax value is duplicated across `.secrets1` and `.secrets2`; the deepseek keys are different, meaning there is no single source of truth for provider identity.

### 7. The unrelated dead cron keeps the healer active

`zdash222-wifi-5g-guard` is `DEAD` in current `verify` output. It has no `DISABLED` registry state, so `warden.sh` continues to call `heal` every five minutes. This does not undo the zk pause because heal skips `DISABLED`, but it leaves the re-enable machinery cycling and increases the chance that a future registry rollback is acted on promptly.

### 8. The audit’s “everything else is interim” framing hides release evidence still missing

The clean Git tree proves the working-tree state is deliberate. It does not prove tests, Hindi completion, Stage 3 ratification, or data-quality acceptance. The correct reviewer question is not “is zsm1 done?” but “which arcs are intentionally exempted from done?” The current documents give three green gates and one red gate without an explicit pk-approved exemption.

## Bottom line

The strongest parts of the plan survive:

- the NCERTS size and lack of active consumer;
- the graphify reclaim magnitude, with two tracked exceptions;
- the worktree storage magnitude;
- the three green zsm1 gates;
- the current prevention of automatic zk heal resurrection;
- the immediate permission improvement of the four keys moved into `~/.secrets1`.

The plan should not proceed on three assumptions:

1. do not call zsm1 fully GREEN/COMPLETE while G6 is RED and Stage 3 is open;
2. do not treat the six crons as durably paused until the registry delta is committed and backup restoration is protected;
3. do not treat opencode as fixed until the guard is scheduled and the SSD salvage duplicate is resolved.
