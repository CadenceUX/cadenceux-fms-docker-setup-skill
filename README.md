# cadenceux-fms-docker-setup

A Claude Code skill for installing and configuring Claris FileMaker Server inside a Docker
container. Verified end-to-end on macOS (Apple Silicon) with Docker Desktop, using Claris's
official arm64 Ubuntu 24.04 build, and hardened through three rounds of real corrections since
first written, a second independent install run (v1.1), and a real in-place upgrade from
26.0.2 to 26.0.3 (v1.2), including the move to Ubuntu's own Nginx that 26.0.3 expects (v1.3), then tightened from an eval run of the skill against a no-skill baseline (v1.4).

Built and maintained by [Darrin Southern](https://www.linkedin.com/in/darrin-southern/) from [CadenceUX](https://cadenceux.com.au).

## What it does

When this skill is active, Claude will:

- Build the FMS image from Claris's own `Dockerfile` directly, rather than their bundled
  `fms_Docker_Installer.sh` script, which can't run on macOS at all
- Pin `ubuntu:24.04` explicitly rather than the drifting, unsupported `ubuntu:latest`
- Run the container with the flags FMS's services actually need: `--privileged`, systemd as
  PID 1, correct port mapping, named volumes for persistent data
- Walk through the interactive assisted-install wizard
- Flag the container-vs-image trap before it costs you data: FMS installs into the
  container's writable layer, not the image or any volume, so an accidental `docker rm`
  silently wipes the software while leaving data intact. The skill commits an image at the
  right points so that mistake is recoverable rather than a full reinstall
- Patch Ubuntu's CVE-affected bundled Nginx on 26.0.2 and earlier, correctly, in a container
  with no working `sudo` — and know when not to, since 26.0.3 changed Claris's approach
- Promote the sample database when it doesn't auto-appear (a common silent gap on reinstalls)
- Set up a locally-trusted HTTPS certificate via `mkcert`, including a documented workaround
  for a real bug in `fmsadmin certificate import`'s external-key path
- Check whether the container's FMS build is current against Claris's own updater feed
- Upgrade FMS in place to a new release — snapshot the container, back up the data volumes,
  dry-run the package, run the upgrade, verify, and commit — so a bad upgrade can be rolled back
- Enable WebDirect, OData, and the Data API — all disabled by default even after a clean
  install — and correctly interpret their genuinely variable startup delay after a restart,
  rather than concluding a working config has failed

## Scope

This is a server-infrastructure skill: getting FileMaker Server itself running correctly in
Docker. It is deliberately not a FileMaker Pro schema, script, or layout authoring skill. See
the `fmp-dev-*` and `claris-filemaker-pro` skills for that side of the platform.

For installing OttoFMS on top of this once the base server is working, see the companion skill
`cadenceux-ottofms-docker-setup` (not yet published to a separate repo).

## Installation

**Easiest — double-click (macOS):** download the `.skill` file from the
[Releases page](../../releases) and double-click it. Claude Desktop registers the `.skill`
extension and opens its install flow directly. (The `.skill` file is the release zip with a
different extension. Not yet confirmed on Windows.)

**Fallback — upload the zip:** in Claude.ai, go to **Customize → Skills** and upload the
release `.zip`. This is the path for the web app and any platform where the double-click
association isn't available.

## Why this exists

Existing community walkthroughs for FileMaker Server on Docker predate Claris's arm64 build
and don't cover several macOS/Apple-Silicon-specific failure modes that actually show up in
practice. This skill is a from-scratch runbook, built by working through a real install
start to finish rather than written from general knowledge, with every gotcha here having
actually happened and been resolved once already.

See `references/troubleshooting.md` for the full detail behind each one, organized by symptom
so you can jump straight to a fix rather than reading the whole runbook when something's
already gone wrong.

## Contributing

Issues and PRs welcome, particularly reports of behaviour on Intel Macs or native Linux Docker
hosts (only the Apple Silicon/macOS path has been run start-to-finish so far), Windows
`.skill` double-click confirmation, and any new failure mode worth documenting for the next
person.

## Licence

CC BY 4.0. Use it, fork it, adapt it. Attribute it.

Built and maintained by [Darrin Southern](https://www.linkedin.com/in/darrin-southern/) from [CadenceUX](https://cadenceux.com.au).
