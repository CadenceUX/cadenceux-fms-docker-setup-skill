# Changelog

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
