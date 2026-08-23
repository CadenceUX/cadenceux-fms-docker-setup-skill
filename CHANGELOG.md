# Changelog

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
