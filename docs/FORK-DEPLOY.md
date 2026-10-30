# Deploying the port fork on gx10-a

How the fork checkout that serves Robert's traffic is brought to a newer `main` (this branch, with the engine as the
patch series `zig/patches/0011-...` to `0022-...` and this fork's defaults on), and how it is rolled back. GPU work
and anything on the Sparks is the coordinator's: these are the steps it runs. Nothing here changes the stock checkout.

## What serves now

| | |
| --- | --- |
| unit | `deepseek1017.service`, enabled: since 2026-10-22 `Type=simple`, `FOREGROUND=1`, `PREPARE=0`, `Restart=on-failure`, `RestartSec=60s`, `StartLimitBurst=3` in 30 min (so the unit **never builds or prepares**: the image must be on both Sparks before a start), `ExecStartPre` waits for the worker; `deepseek1017-watchdog.timer` restarts it after 3 failed `/health` checks, never while `~/.deepseek1017-hold` exists |
| drop-in | `/etc/systemd/system/deepseek1017.service.d/fork.conf`: `WorkingDirectory`, `ExecStart` (`start.sh`) and `ExecStop` (`stop.sh`) point at **`~/DeepSeek-port`**; `ExecStop` is resolved at stop time, so swap folders only while the unit is stopped |
| fork checkout | `~/DeepSeek-port` on gx10-a: an earlier tree of this branch (or of the upstream `spill-kvshard`) with the fork's own `scripts/local.sh` |
| image | `tensorfold-dsv41:exl3-tp2-067553852da3` on both Sparks since 2026-10-30 (Mia's main `0e7d421` + display reserve `0039`); the previous `exl3-tp2-2cacfb08137a` stays for a rollback |
| stock checkout | `~/DeepSeek-v4.1-Flash-1017-2x-DGX-Sparks`, untouched (the unit's own paths) |

This branch already carries both local fixes of the stock tree (`compiler-mirror.patch`: `ZIG_MIRRORS`;
`worker-helper-image.patch`: the worker's helper containers use `$IMAGE`), so the fork needs no working-tree edits
any more: only `scripts/local.sh`, which git ignores.

## Update: a new fork from this branch

Make the new tree beside the old one and swap the two folders, so the drop-in's path never changes and the old tree
stays whole for a rollback.

1. **On the Mac (no Spark involved).** The engine is in `zig/patches` already (the series `0011` to `0022`, then `0023` and
   upstream fixes `0024`–`0038` and ezoushen's display reserve `0039`). Export engine work made after it
   (on top of an engine base containing all current patches) as `0040-...`, commit it on `main`, then bundle the branch:

   ```
   cd ~/GitHub/deepseek1017-recipe-port
   scripts/port-patches.sh --base <engine-base> --branch <engine branch> --dry-run   # then without --dry-run
   bash scripts/port_test.sh && python3 -I tools/selftest.py      # the golden start.sh test skips here: step 4
   git add zig/patches && git commit -m "zig/patches: 0040-... = <engine branch> <hash>"
   git bundle create ~/port-recipe.bundle main
   scp ~/port-recipe.bundle gx10-a:
   ```

   Each series patch's header says what it carries; `head -1` of a `port-patches.sh` export names the engine branch
   and commit it was exported from.

2. **On gx10-a: hold, record what serves.**

   ```
   touch ~/.deepseek1017-hold                                         # the watchdog leaves the stack alone
   d=~/dsv41-rollback/fork-$(date +%Y%m%d-%H%M); mkdir -p $d
   docker inspect -f '{{.Config.Image}}' dsv41-1017-exl3-tf | tee $d/image
   ssh <worker> docker inspect -f '{{.Config.Image}}' dsv41-1017-exl3-tf      # the same tag
   docker tag "$(cat $d/image)" tensorfold-dsv41:fork-rollback
   ssh <worker> docker tag "$(cat $d/image)" tensorfold-dsv41:fork-rollback   # no prune can take it now
   systemctl cat deepseek1017.service > $d/unit.txt
   ```

3. **The new tree**, a clean checkout of the bundle, with the fork's own settings:

   ```
   git clone --quiet --branch main ~/port-recipe.bundle ~/DeepSeek-port.next
   cp -p ~/DeepSeek-port/scripts/local.sh ~/DeepSeek-port.next/scripts/local.sh
   [ -f ~/DeepSeek-port/.env ] && cp -p ~/DeepSeek-port/.env ~/DeepSeek-port.next/.env
   ls ~/DeepSeek-port.next/zig/patches/       # 0001 0004 0005 0008 0010, then 0011 to 0039: one port only
   ```

   Only one copy of the port may be in `zig/patches` (an older `0007-...` or `0011-spill-kvshard.patch` export and the
   series touch the same files and would not apply together). This branch turns spill v2, KV sharding and the display
   reserve on by default (`SPILL_GIB=64`, `KV_SHARD=1`, `DISPLAY_KV_MIB=1792`, an empty `POOL` derived as 5,218,304,
   or 2,101,248 without the reserve), each left out of a start it cannot run in, with a warning; `scripts/local.sh`
   needs no feature lines, and one set there refuses the start instead (a bad spill folder, say). Each Spark's
   `SPILL_DIR` / `WORKER_SPILL_DIR` must be on its own NVMe (docs/PORT-RECIPE.md).

4. **Test the new tree on gx10-a, then look before switching.** The Mac has only bash 3.2, so `scripts/port_test.sh`
   skips its start.sh DRY_RUN golden test there: that test, the only one comparing start.sh's full docker commands
   (all off, KV shard with and without an explicit pool, the defaults, a failing default spill folder against an
   explicit one, spill on) with MiaAI-Lab's main (`370daf0`, in the bundle's history), runs here,
   with bash 5 and on Linux, on a disk folder (never `/tmp` on a Spark):

   ```
   cd ~/DeepSeek-port.next && mkdir -p ~/sess/tmp
   f=~/sess/port_test-$(date +%Y%m%d-%H%M).txt
   TMPDIR=~/sess/tmp bash scripts/port_test.sh | tee $f | tail -3
   grep -cE '^ok   golden: ' $f; grep -E '^(FAIL|skip golden|skip start.sh)' $f    # 7 (8 with a fork's defaults on), and nothing
   ```

   **Do not swap the folders unless it ends in `N of N checks passed` with every golden case `ok`** (a `skip
   golden` line or a `FAIL` stops the deploy; record the result in the deploy notes). Then
   `STATE_DIR=~/sess/dry-state DRY_RUN=1 PREPARE=0 FOREGROUND=1 ./start.sh restart` prints both ranks' docker
   commands and changes nothing (its own `STATE_DIR`: the serving unit's start.sh holds `start.lock` for as long as it
   serves, so a DRY_RUN on the shared state dies with 'another ./start.sh is already running') (the image tag in them is the new one: the patch hash changed). With spill on, check
   the identity variables (`TF_DS_ENGINE_ID`, `TF_DS_RECIPE_PATCHES`, `TF_DS_MODEL_REVISION`, `TF_DS_ENGRAM_REVISION`;
   `TF_DS_RANK_CACHE_SHA` appears once its digest is cached, after the first real start). A real start with spill on
   refuses (before stopping anything) when either rank's weight digest cannot be had: the digest names the store, so
   a rank without it would open an empty one. `SPILL_WEIGHTS_SHA=0` runs without digests on purpose, under another
   store name too.

5. **The image first, then swap and start through the unit.** The unit runs `PREPARE=0`: a start whose image tag is
   missing dies after `ExecStop` has already taken :8888 down, and `Restart=on-failure` repeats that into the start
   limit. So put the new image (`source scripts/config.sh; echo $IMAGE` in the new tree) on both Sparks first:

   - **Off the Sparks (what 2026-10-30 did; the Sparks keep serving):** on an arm64 Linux VM (podman), pull the base by
     the digest the Sparks hold (`docker image inspect nvcr.io/nvidia/pytorch:26.07-py3` on gx10-a), then build with
     `scripts/prepare.sh`'s exact `docker build` arguments (its step 2: `TF_REPO`, `TF_REF`, `INPUTS_REF`, `ZIG_*`,
     `BASE_IMAGE`, `PATCHES_HASH=$(image_hash)`, the tag `$IMAGE`). `podman save --format docker-archive`, drop the
     base's layer files from the archive (keep any the new layers also name) and `docker load` the ~0.6 GB rest on each
     Spark: docker loads a layer from the archive only when its chain id is not in the store. Check the same image id
     on both Sparks.
   - **On gx10-a while :8888 is down:** `VERIFY_SHA=0 scripts/prepare.sh` in the new tree (~6 min of Zig, then the
     copy to the worker). Not while serving: it needs memory the serving ranks do not leave.

   Either way, first check the new image's `tf-dsv41-rank-cache <checkpoint> /tmp/x <rank> 2 --index-only | sha256sum |
   cut -c1-20` against the existing `rank<r>of2-<hash>.bin` names (a VM run on a copy of the 39 shards' headers, the
   tensor bytes left sparse, is enough): a different hash means `prepare.sh` must write ~106 GB a Spark before the start.
   Then, in an idle moment (no `tensorfold:requests_running`/`_waiting` and no new requests for a minute):

   ```
   touch ~/.deepseek1017-hold
   sudo systemctl stop deepseek1017.service
   mv ~/DeepSeek-port ~/DeepSeek-port.prev-<old commit> && mv ~/DeepSeek-port.next ~/DeepSeek-port
   S=~/.local/state/dsv41-tensorfold; cp -p $S/prepared $S/prepared.<old commit>
   (cd ~/DeepSeek-port && bash -c 'source scripts/config.sh; source scripts/nodes.sh; write_prepared_marker')
   sudo systemctl reset-failed deepseek1017.service && sudo systemctl start deepseek1017.service
   ```

   The marker matters only for a manual `./start.sh` (`PREPARE` unset); its format changed with Mia's `0e7d421`. On
   2026-10-30 :8888 was down 64 s (stop 7 s, load 5 s, start to healthy 51 s).

6. **Check, then release the hold:**

   ```
   cd ~/DeepSeek-port
   python3 -I tools/smoke.py && python3 -I tools/exact.py record --like <baseline.json> --out new.json && \
     python3 -I tools/exact.py compare <baseline.json> new.json
   curl -s localhost:8888/health | python3 -m json.tool | grep -A3 -E 'kv_shard|spill|"pair"'
   docker logs dsv41-1017-exl3-tf 2>&1 | grep -E 'dsv41: pool|dsv41 spill|CUDA budget|status 75'   # the last: below
   rm ~/.deepseek1017-hold
   ```

   For the full GPU runbook (every configuration, receipts, restore) use `tools/gpu_validate.sh` instead
   (`tools/PORT-VALIDATION.md`); it works in its own checkout under `~/sess` and never touches `~/DeepSeek-port`.

## A broken pair ends rank 0 (status 75)

The engine in this tree ends a broken pair instead of hanging on it: when rank 1 dies or hangs, or a spill-tier or
KV-transfer exchange breaks, rank 0 exits with status 75 (a dead rank 1 within seconds, a hung one after the 120 s
stall bound and at most the 15 s drain deadline) and `/health` stops answering (docs/PORT-RECIPE.md "When the rank
pair breaks"). Its start-up line says so (`... ends this server with status 75 ...`).

Since 2026-10-22 the unit is `Type=simple` with `FOREGROUND=1`, so systemd sees that exit and restarts the pair after
`RestartSec` (60 s); `deepseek1017-watchdog.timer` remains the backstop for a hang (3 failed `/health` checks 2 minutes
apart). The paragraphs below describe the earlier oneshot unit. Nothing changes on gx10-a for that: the unit, its drop-in, the watchdog and its timer stay as they
are, and the hold keeps the watchdog off during manual work as before. The restart's `./stop.sh` (the unit's
`ExecStop`) saves both ranks' logs before it removes the containers, so the newest
`~/.cache/dsv41-tensorfold/logs/*-rank0.log.gz` ends with the engine's `exiting with status 75` and its cause.

A restart within a minute would mean moving the unit to the recipe's template (docs/systemd.md: `Type=simple`,
`FOREGROUND=1`, `PREPARE=0`, `Restart=on-failure`, `RestartSec=60s`), which restarts on the exit code itself. That is a
change of the deployment, not of this tree, and it takes two more in the same step: the watchdog goes or changes (its
`pgrep ... start.sh` would always match the attached launcher and never act, and one supervisor is the rule), and
images are built by hand before a restart (`PREPARE=0`; step 5 builds through the unit today).

To see the whole path on the pair: `tools/gpu_validate.sh --configs pair-kill,pair-stop` (tools/PORT-VALIDATION.md)
kills or stops rank 1 under load, checks that every stream ends and rank 0's container is `exited 75`, then restarts
the pair itself.

## Rollback

Hold first (`touch ~/.deepseek1017-hold`), remove it at the end.

- **A feature off, same image:** set it off explicitly in `~/DeepSeek-port/scripts/local.sh` (`SPILL_GIB=0`,
  `KV_SHARD=0`, `DISPLAY_KV_MIB=0`, `POOL=`), `sudo systemctl restart deepseek1017.service`. With all three at `0` this
  is the port image with the port unused; removing the lines turns this fork's defaults back on.
- **The previous fork, exactly:** swap the folders back and restart. Its image is still on both Sparks (and tagged
  `fork-rollback`), so `prepare.sh` finds it and rebuilds nothing:

  ```
  mv ~/DeepSeek-port ~/DeepSeek-port.bad-$(date +%Y%m%d-%H%M) && mv ~/DeepSeek-port.prev-<stamp> ~/DeepSeek-port
  sudo systemctl restart deepseek1017.service
  ```

  `source scripts/config.sh; echo $IMAGE` in the restored tree must print the tag in `$d/image`.
- **Not the previous image from the new tree:** since `0e7d421` start.sh passes settings an image on the older pin
  refuses (`--graph-cap` to rank 1, among others). Roll back by swapping the folders, and restore the old tree's
  `prepared.<old commit>` marker.
- **Back to the stock server:** `sudo rm /etc/systemd/system/deepseek1017.service.d/fork.conf && sudo systemctl
  daemon-reload && sudo systemctl restart deepseek1017.service`. The stock checkout and its image were never changed.
- **Spilled state** stays in each Spark's spill folder (`<SPILL_DIR>/native-spill-<compat>/`), owned by you after a
  stop; a new image or checkpoint gets a new `<compat>` folder and never reads the old one. Reclaim it with the
  server stopped: `rm -rf <SPILL_DIR>/native-spill-*` on each Spark.

Old trees (`~/DeepSeek-port.prev-*`) and the `fork-rollback` tag can go once the new fork has served for a while
(`docker image rm tensorfold-dsv41:fork-rollback` removes the tag only while another tag names the same image).
