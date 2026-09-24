# Changelog

## v1.5 — 2026-09-24

Works through the eval findings left over from v1.4. It also runs the rollback for real, in an
isolated test container, so it's no longer marked "not yet exercised". Every new command was
run before release (details per item).

- **Rollback — verified and rewritten as numbered steps.** A test container (`--network none`,
  so it couldn't clash with the live server's ports or licence) was recreated from the dated
  `pre-upgrade` tag onto volumes restored from the U3 tars. It came back as 26.0.2.219 in about
  20 seconds, with `fmshelper` active, the database opened and the Data API OK; it was then
  removed. The restore was run twice, the second time over a non-empty volume, to prove the
  "empty first" part. The steps now:
  1. commit the failed state first
  2. record the settings, then stop and `rm` explicitly
  3. restore the data only when it's needed
  4. recreate from the dated tag
  5. verify, then point `:installed` back
- **New trap — checking a restore with `find -type f` undercounts.** The backups contain 13
  OttoFMS symlinks and a named pipe (`.passphrase`), and `-type f` skips both, which looked like
  missing files. Use `find ! -type d`.
- **Step 8 (mkcert) — rewritten.**
  - Homebrew only if it's already installed, otherwise the direct download. The old text said
    "don't default to Homebrew" and then led with `brew install mkcert`.
  - Every command is labelled developer-Mac, developer-container-tab or agent.
  - **Ordering bug fixed:** `mkcert -install`, which creates the CA, now runs first. Signing
    needs the CA files, so on a Mac that had never run mkcert, the old order failed at the
    signing step. There's also a check that the CA files exist.
  - The broken "see the note below" pointer now names the `/tmp` entry in troubleshooting.
  - The signing commands were tested with a throwaway CA; all four SAN entries were present.
- **Upgrades — skipping releases and bigger jumps.** Read the release notes for every release in
  between. Apply version-dependent steps according to which releases the jump crosses (the Nginx
  follow-on heading now says "from 26.0.2 or earlier to 26.0.3+"). Check Claris's supported
  upgrade paths. A major version, or a Dockerfile whose `FROM` changes, is flagged as an untested
  rebuild, not an in-place `apt` upgrade.
- **U1 — old package no longer on disk:** check the new Dockerfile's `FROM` line and package list
  against the running container. The package check goes through `xargs` because zsh doesn't
  word-split `$var`: the first attempt passed all 24 packages as one bogus name. Run verbatim
  from the skill in zsh against the live container, all 24 were present, and a fake package
  name was reported as missing.
- **U3 — backup location agreed with the developer** (absolute path under `/Users`) instead of
  `$PWD`, with a fallback image if `fmsdocker:prep` is gone.
- **Upgrades — agree the downtime first:** U3 and U5 each disconnect clients.
- **Storage decision:** named volumes are recommended. A host-folder bind mount is flagged as
  untested and likely to trigger the disk-usage false alarm; U3's tars are the way to get
  Finder-visible copies.
- **Troubleshooting — disk-usage fix:** now finds the bind mount with `docker inspect`, and says
  what to do when there isn't one. It commits and confirms before recreating, and no longer
  refers to an `/install` line the snippet doesn't have. The inspect command was checked against
  the live container.
- **Troubleshooting — certificate fix:** notes that mkcert's CA exists only after
  `mkcert -install`.
- **Verified-environment table:** new rollback row.
- **From an independent pre-release review:**
  - The rollback restore used U3's `$B`, which is empty in a later shell (`-v "":/src` fails).
    It now takes the backup folder path, and U3 prints that path and says to run its block in
    one call.
  - U2 said "four lines" but has five.
  - Step 4 said the upgrade section "recommends" Ubuntu's Nginx; that section deliberately
    presents two options.
  - The README still called Ubuntu's Nginx "CVE-affected".
  - The disk-usage verify command still hardcoded `/install`.
  - The docker-rm eval's expected answer predated the recreate-from-image-first fix.

## v1.4 — 2026-09-24

Fixes from an eval run of v1.3: four eval prompts, each run once with the skill and once
without, all plan-only. Every case passed; the with-skill run also flagged these gaps. No new
install or upgrade was run for this release; the new commands were checked read-only against
the live 26.0.3.309 container.

- **Troubleshooting — "FileMaker Server disappeared" rewritten.** The old reinstall fix ran the
  interactive wizard through a non-interactive `docker exec` (no `-it`), so it couldn't work.
  It now:
  - rules out the wrong shell first
  - prefers recreating from a committed image, so no reinstall is needed
  - backs up the volumes before any reinstall
  - reinstalls the same build the data last ran, read from `Event.log`'s
    `Starting Database Server <version> <build>` line (verified)
  - uses `docker exec -it` in a named terminal tab
  - re-checks the Step 6 Nginx patch and the publishing settings, since `deployment.xml` isn't in
    any of the four volumes
  - deletes the `.deb` before committing
- **U2 — rollback tag.** `STAMP` is set once and reused for the commit and the tag. Building the
  timestamp twice broke the tag whenever the minute changed between the two commands.
- **Step 5 — delete the `.deb` and run `apt-get clean` before committing**, the same lesson as U7.
  A fresh install no longer bakes the ~540MB installer into `fmsdocker:installed`.
- **Step 4 — Nginx wording.** No longer calls Ubuntu's 1.24.0 plainly "outdated/CVE". It explains
  the warning goes by version number, that Ubuntu backports fixes without changing that number,
  and that Claris's guidance changed between 26.0.2 and 26.0.3. This removes the conflict with
  the upgrade section's advice to switch back to Ubuntu's package.
- **Step 4 — `docker cp` into `/tmp` first.** The temporary bind-mount suggestion is removed; a
  lingering bind mount is what causes the disk-usage false alarm.
- **Step 3 — no hardcoded host values.** Memory, CPUs and the HTTP/HTTPS host ports are now
  placeholders, with a table saying where each comes from and what the verified runs used.
  Every URL uses the `[:port]` convention.
- **Troubleshooting — recreate snippet.** Uses the same placeholders, and says to read the
  current container's real settings with `docker inspect` (checked against the live container)
  before removing it.
- **New pre-flight check 7 — existing containers, volumes and images.** `docker volume create`
  on an existing name silently does nothing, so a "fresh" install could land on old data.
  Existing volumes need an explicit decision, and a backup before any removal.
- **New "Two conventions" section.** Confirm the real container name with `docker ps` instead of
  assuming `fms`; the handoff prompt comes from the container's hostname, not its name (seen: a
  container named `docker` with hostname `fms`). Also defines `[:port]`.
- **From a second, targeted re-check of v1.4:**
  - Pre-flight check 6 pointed to a `docker run` in Step 5 that doesn't exist; it now points to
    the troubleshooting snippet.
  - U6's `openssl s_client -connect localhost[:port]` had no port when 443 is published directly,
    so it now uses `localhost:<HTTPS_PORT>`. The `[:port]` convention is now stated as
    `https://`-URLs only.
  - Troubleshooting now covers recreating when the container is already gone (where to recover
    the settings), and reading the build from the log volume with a throwaway container.
- evals: new case for leftover `fms-*` volumes.

## v1.3 — 2026-09-24

The Nginx follow-on decision from v1.2 has now been run on the verified container: after the
26.0.3 upgrade, it was switched from the frozen nginx.org 1.30.4 build to Ubuntu's
`nginx` 1.24.0-2ubuntu7.18.

- **Upgrade section — "Switching to Ubuntu's `nginx` (verified)".** Dry run with
  `apt-cache madison`, then an install with a temporary `policy-rc.d` (exit 101) so Ubuntu's
  package can't start the generic `nginx.service` against FMS's own Nginx on 80/443, plus
  `--force-confdef/--force-confold`. Restart the container, confirm FMS's master process runs the
  new `/usr/sbin/nginx` with `fms_nginx.conf`, run the U6 checks, commit. Records the expected
  harmless output and that Ubuntu's package replaces the unused `/etc/nginx/` defaults.
- **New trap:** remove `policy-rc.d` as its own step. On the verified run, `systemctl is-active
  nginx` returned `inactive` (exit 3) under `set -e`, which skipped the `rm`. Left in place, it
  silently blocks every later service start from apt.
- "Keep the nginx.org build" is still offered but marked not run.
- Verified-environment table gains the Nginx switch row.

## v1.2 — 2026-09-24

Adds in-place version upgrades, built from a real 26.0.2.219 → 26.0.3.309 upgrade of a running
container (arm64, macOS, Docker Desktop, with mkcert certificate, WebDirect/Data API and OttoFMS
already in place — all survived).

- **New section — "Upgrading FileMaker Server in place" (U1–U7).** Compare the new package with
  the old one (Dockerfile, `Assisted Install.txt`, helper scripts); snapshot the container to a
  dated `pre-upgrade` tag; back up the four volumes with the container stopped, using the
  existing `fmsdocker:prep` image for the tar step; `docker cp` the `.deb` in and dry-run it with
  `apt-get install -s`; run the interactive upgrade in a named terminal tab; verify; delete the
  `.deb` *before* committing. Documents the harmless-but-alarming installer output (`crontab: No
  such file`, the 80% progress bar, Nginx source removal). Rollback steps included and marked
  not yet exercised.
- **New section — staleness check.** Active pattern against Claris's updater feed
  (`product-updaters.txt`, JSON), filtered to FileMaker Server / Linux; signal validated live the
  day 26.0.3 was installed. Advisory message when the container or the runbook is behind.
- **Step 6 (Nginx) is now version-dependent.** 26.0.3's release notes drop `NginxUpdate.sh`,
  the package no longer ships it, and its installer removes the nginx.org source, pin and key
  that Step 6 adds. On 26.0.3+ don't re-add them without asking. The upgrade section covers the
  resulting decision for containers patched under 26.0.2: the nginx.org build is left installed
  but frozen, since Ubuntu's `nginx` has a lower version number — both options laid out, neither
  run yet.
- **Step 4:** the Nginx CVE warning is now scoped to 26.0.2; not yet observed on a fresh 26.0.3.
- **Pre-flight port check:** identify what holds a port with `curl -sI`; on macOS, port 80 is
  often the built-in Apache — the reason for `8080:80`, and why `http://localhost/admin-console`
  404s in a browser.
- **Troubleshooting — two new entries:** `http://` Admin Console 404 (macOS Apache on port 80),
  and an oversized committed image (`.deb` left in `/tmp`).
- **Verified environment** is now a table of what has actually been run, and states plainly that
  a fresh 26.0.3+ install hasn't been.
- Description: adds upgrade/update triggers; tightened to stay under 1024 characters.
- evals: new upgrade case.

## v1.1 — 2026-09-19

Folds in findings from a second, independent start-to-finish install (FMS 26.0.2.219 arm64, macOS
15, Apple Silicon, 16GB RAM, ~7.75GB to Docker Desktop) run by another agent. The core sequence —
Dockerfile build, volumes, `docker run` flags, `docker commit` locking, Nginx CVE patch, mkcert
CSR/sign/import — worked exactly as documented with no command changes, so those sections are
untouched. The gaps were assumptions the skill made that didn't hold:

- **Step 8 — no-Homebrew route.** Added a direct-binary download of mkcert (maintainer's own
  redirect, no sudo, no package manager) as the first-line option when `brew` is absent, with a
  `file` sanity check and the `~/bin` full-path caveat, instead of implying Homebrew is required.
- **Step 4 — installer output that misleads inside Docker.** The printed Admin Console URL is the
  container's internal bridge IP (unreachable from the host); use `https://localhost[:port]/...`.
  The "add user to fmsadmin group / restart your system" line is bare-metal boilerplate and safe
  to ignore. Same URL callout added to "Verifying the whole install".
- **Step 4 — handoff wording.** Interactive steps must name the exact terminal tab and prompt
  string (`root@fms:/#` vs the macOS shell) every time; a mismatch cost several turns in the
  source run.
- **Step 7 / troubleshooting — sample database.** Separated "stuck in staging" from "absent
  entirely" (empty `Sample/` dir, no `.fmp12` anywhere). The second case is flagged as
  **root cause not established** — wizard answer vs version-specific default — rather than
  asserting either; confirming needs a run that deliberately keeps the sample.
- **Troubleshooting — new entry:** `command not found` / `No such file` for a Linux command
  during an interactive step almost always means the host shell, not the container.
- Verified-environment note updated to record the second successful run.

## v1.0 — 2026-08-11 (updated 2026-08-11, pre-release)

Initial release. Built directly from a real, start-to-finish FileMaker Server 26.0.2.219
install on Docker (macOS/Apple Silicon, Docker Desktop, Claris's official arm64 Ubuntu 24.04
build), capturing every failure mode actually hit during that session rather than written from
general knowledge.

Covers: building from Claris's own Dockerfile instead of their macOS-incompatible
`fms_Docker_Installer.sh`, pinning `ubuntu:24.04`, correct container run flags, the interactive
assisted-install wizard, the container-vs-image data-loss trap (with `docker commit`
mitigation), patching Ubuntu's CVE-affected bundled Nginx without a working `sudo`, promoting
an unpromoted sample database, and a full trusted-local-HTTPS-cert workflow via `mkcert`
including the documented workaround for a real bug in `fmsadmin certificate import`'s
external-key path.

**Same-day correction (pre-release, amended in place, no version bump — unshipped):** the
initial draft documented commands and gotchas thoroughly but didn't instruct a future session
to actually pause and confirm at genuine decision points, or hard-check system/drive
requirements before acting. Added a "Pre-flight checks" section that checks Docker Desktop's
allocated resources, host disk space, architecture match, and port conflicts before any build
starts (with explicit guidance on when to just state a default vs. when to actually ask), plus
a "Decisions that need the developer's input" section covering data-storage location, resource
limits, credentials, single-vs-multi-server, and confirming before removing any existing
container that might hold state.

**Second correction (2026-08-12, amended in place, no version bump — still unshipped):** added
Step 9, "Enable WebDirect / Web Publishing (optional, but off by default)" — discovered that
every publishing component (WPE/WebDirect, Custom Web Publishing, the Data API, OData) is
disabled by default even after a fully successful install, producing a 502 on `/fmi/webd` that
looks like a broken install but isn't. Documents the `deployment.xml` check, the
Admin-Console-or-CLI enable path, and the real startup delay before the Java Web Publishing
Engine actually becomes reachable (confirmed via `Event.log` timing, not assumed). Matching
entry added to `references/troubleshooting.md`. Also hardened the port pre-flight check —
`lsof` without `sudo` can miss root-owned listeners and under-report a genuine conflict, so a
plain `nc -z` connection test was added alongside it — and noted that a port 80/443 conflict
found during initial setup isn't necessarily permanent (confirmed in practice: port 443 was
occupied at setup, free again about a day later with nothing else changed), so it's worth
re-checking rather than assuming an earlier remap to `8443` has to stay that way forever.

**Third correction (2026-08-12, amended in place, no version bump — still unshipped):**
replaced the theorized "wait a minute or two" timing guidance for WPE/OData/Data API with a
confirmed-variable finding from two deliberate, back-to-back restart tests on the same
container: first restart, OData/Data API took 34-45 seconds to come up; second restart, same
`enabled="yes"` config unchanged, both were up within 2-3 seconds. There is no fixed number
worth hardcoding — SKILL.md and troubleshooting.md now both say to poll with retries over
roughly a minute instead. Also documented the inverse case directly (a developer manually
starting a component after finding it "off" is not evidence the setting reverted — check
`deployment.xml` first; if it already says `yes`, it was a timing artifact, not a regression),
since this was tested live and confirmed rather than assumed. Fixed a bug in the
`deployment.xml`-checking grep command in both files — it matched on `name="wpe"` alone, but
the `enabled` attribute appears *before* `name` in FMS's actual XML output, so the original
command never actually showed enabled/disabled state despite looking like it did.

**Published to GitHub (2026-08-23):** now living at `github.com/CadenceUX/cadenceux-fms-docker-setup-skill`, with this v1.0 as the first tagged release. Everything above happened before publication, hence three corrections landing inside one still-unbumped v1.0 — genuinely the first time this skill has shipped anywhere.
