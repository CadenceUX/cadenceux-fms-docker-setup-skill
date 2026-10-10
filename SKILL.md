---
name: cadenceux-fms-docker-setup
description: >-
  Installs, upgrades and configures Claris FileMaker Server in Docker on macOS with Docker
  Desktop, using Claris's official arm64 Ubuntu 24.04 build: building from Claris's Dockerfile
  (their install script can't run on macOS), the install wizard, persistent volumes, a trusted
  mkcert certificate, WebDirect, Data API and OData, in-place upgrades with a snapshot and
  volume backup, and rollback. Use when asked to install, set up, upgrade, update, run or
  troubleshoot FMS on Docker ("FMS on Docker", "Dockerize FileMaker Server", "is my FMS
  current?"), when FMS installer files (filemaker-server*.deb, or an fms_*.zip with a Docker
  folder) are in the working directory, or on symptoms it resolves: a wrong Admin Console
  disk-usage figure, "Cannot decrypt the private key file" on certificate import, a missing
  sample database, Nginx CVE warnings, or FMS vanishing after a container was recreated. Not
  for FileMaker Pro schema, script or layout work; for OttoFMS on top, see cadenceux-ottofms-
  docker-setup.
license: CC-BY-4.0
compatibility: >-
  Claude Code on macOS (Apple Silicon) with Docker Desktop. Needs the developer's licence-
  linked FileMaker Server Linux download.
metadata:
  author: Darrin Southern, CadenceUX
  version: "1.6.0"
---

# FileMaker Server on Docker — Setup

A verified, ordered runbook for getting Claris FileMaker Server running in a Docker container,
built from a real session that hit (and resolved) every trap documented here. It replaces the
generic three-year-old community walkthroughs that predate Claris's own arm64 build and don't
cover the macOS-specific failure modes below.

**Verified environment:** macOS (Apple Silicon/arm64), Docker Desktop, Claris's official
`fms_*_Ubuntu24_arm64` package, Ubuntu 24.04 base image. What has actually been run:

| FMS build | Path | Runs |
|---|---|---|
| 26.0.2.219 | Fresh install, Steps 1–9 | 2 (second on macOS 15, 16GB RAM, ~7.75GB to Docker Desktop — no command changes needed) |
| 26.0.2.219 → 26.0.3.309 | In-place upgrade (see *Upgrading FileMaker Server in place*) | 1 (2026-09-24) |
| 26.0.3.309 | Switch from frozen nginx.org 1.30.4 to Ubuntu's `nginx` 1.24.0 after the upgrade | 1 (2026-09-24) |
| 26.0.3.309 → 26.0.2.219 | Rollback: dated snapshot + volumes restored from U3 tars (isolated test container, `--network none`) | 1 (2026-09-24) |

**A fresh install of 26.0.3 or later has not been run yet.** Steps 1–9 were written against
26.0.2.219; where 26.0.3 is known to behave differently (Step 6, Nginx) that's called out
inline, but treat the rest as expected-to-hold rather than verified for newer builds. The same
approach should generalise to Intel Macs and native Linux Docker hosts (the container itself is
architecture-neutral once you match the right FMS package to `uname -m`), but only the
arm64/macOS path has been run. Flag it to the developer if the host differs, rather than
assuming identical behaviour.

## Two conventions used in every command below

**Container name — confirm it, don't assume `fms`.** Commands here use `fms`, the name Step 3
creates. For any container that already exists, check first and substitute the real name:

```bash
docker ps -a --format '{{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
```

Seen in practice: a working FMS container named `docker` but with hostname `fms` — so its prompt
still read `root@fms:/#`. The in-container prompt shows the **hostname** (`root@<hostname>:/#`),
not the container name, so take the prompt string for handoffs from
`docker inspect <name> --format '{{.Config.Hostname}}'`. (FMS's own server name, as it appears in
`Event.log`, is a third, separate name — don't confuse it with either.)

**`[:port]` in URLs** means the host-side HTTPS port chosen in Step 3: nothing when host 443 is
published directly, otherwise `:8443` or whatever was picked. It's for `https://` URLs only —
`openssl s_client -connect` always needs an explicit port, so those commands use
`localhost:<HTTPS_PORT>` (e.g. `localhost:443`). For an existing container, read it
from the `Ports` column above rather than assuming.

## Is this FMS build current? (staleness check)

This runbook snapshots a vendor product that ships updates every few months. Run this check
when the developer asks whether their FMS is current, mentions an update or "latest", brings a
new `fms_*` zip, or before starting a fresh install from an installer they downloaded a while
ago. It's advisory — never a blocker.

Claris's updater feed is JSON and lists every release per platform. Validated 2026-09-24: it
reported 26.0.3 for Linux on the day that release was installed.

```bash
curl -sL --max-time 20 https://www.claris.com/cms/resources/downloads/updaters/product-updaters.txt \
| python3 -c "import json,sys,re; d=json.load(sys.stdin); v=sorted({re.search(r'\((\d+\.\d+\.\d+)\)',e['version']).group(1) for e in d if e['product'].strip()=='FileMaker Server' and e['platform']=='Linux' and re.search(r'\(\d',e['version'])}, key=lambda s:tuple(map(int,s.split('.')))); print(v[-1])"
```

Compare that with what the container is running — `docker exec fms dpkg -l filemaker-server`
(the feed gives `26.0.3`; dpkg adds the build number, `26.0.3.309`). The feed has no download
URL for Linux; the installer comes from the developer's own licence-linked Claris download, so
ask them for the zip rather than trying to fetch it. For what changed, read the release notes
at `https://help.claris.com/en/server-release-notes/content/index.html` — look specifically for
Linux, Nginx, Ubuntu and install/upgrade notes, since those are what touch this runbook.

If the feed is newer than the container, say so:

> ⚠️ **Newer FileMaker Server available** — the container runs [installed]; Claris's latest
> Linux release is [latest]. See *Upgrading FileMaker Server in place* (references/upgrading.md). Separately, this
> runbook was last verified against 26.0.3.309 — anything newer may behave differently.

If the fetch fails, say the check couldn't run and carry on — don't guess a version.

## Why not just follow Claris's own Docker installer script?

Claris ships a `Docker/` folder alongside the FMS installer containing `fms_Docker_Installer.sh`
— a full multi-container orchestration script. **Do not run it directly on macOS.** It calls
Linux-only host commands (`lsb_release`, `ip link` for MacVLAN networking) that don't exist on
macOS and will abort immediately (`set -e` is active throughout). Its bundled `Dockerfile`,
however, builds fine with a plain `docker build` regardless of host OS, since that runs inside
the Linux build context. This runbook builds the prep image from that Dockerfile directly and
configures the container by hand — simpler, more transparent, and it actually works on macOS.

## Pre-flight checks — verify these before running anything, don't just skim them

This whole section exists because a container build/run failure *after* several minutes of
downloading and `apt install`-ing is a much worse debugging experience than catching the same
problem in five seconds up front. Run every check below and resolve or explicitly flag each one
— don't proceed with an unresolved item silently.

1. **Docker Desktop is actually running.** `docker info` should return cleanly, not hang or
   error. If it hangs, Docker Desktop probably isn't started — say so and wait, don't retry in a
   loop.

2. **Docker Desktop has enough allocated resources for FMS.** FMS itself wants real memory —
   check what Docker Desktop currently has:
   ```bash
   docker info --format 'CPUs={{.NCPU}} Mem={{.MemTotal}}'
   sysctl -n hw.memsize | awk '{print $1/1024/1024/1024" GB total host RAM"}'
   ```
   As a rough guide, Docker Desktop should have at least 4GB allocated for a single-server dev
   setup (8GB+ more comfortable). If it's noticeably tight relative to the host's total RAM —
   especially on a 16GB Mac where Docker Desktop, other apps, and macOS itself are all
   competing — **surface this to the developer as a heads-up before proceeding**, rather than
   pushing forward and hoping. They may want to raise Docker Desktop's memory allocation in
   Settings → Resources first. This isn't usually worth blocking on outright — state the numbers
   and let them decide.

3. **Enough free disk space — check both the Mac and Docker's own usage.** The Ubuntu base
   image, FMS's `.deb` (400-500MB+), and the built layers together can run several GB; leave
   real headroom rather than cutting it close:
   ```bash
   df -h / | tail -1
   docker system df
   ```
   If free space on the Mac's own boot volume is under ~10GB, flag it before starting the build
   — a failed build partway through burns time and can leave partial images/layers to clean up.
   This is a genuine "ask before proceeding" case if space is tight, not just an FYI, since
   recovering from a mid-build disk-full failure costs more than a 10-second check up front.

4. **Host architecture matches the downloaded FMS package.** `uname -m` on the Mac, and confirm
   the FMS zip/folder name matches (`arm64` vs `amd64`). Claris ships a native arm64 Ubuntu
   build — no x86 emulation needed on Apple Silicon, but a mismatched package will build/install
   the wrong architecture.

5. **Locate the `.deb` and the `Docker/` subfolder** (containing `Dockerfile`,
   `fms_Docker_Installer.sh`, `Resources/`) inside the extracted FMS package. The `.deb` usually
   sits one level *above* the `Docker/` subfolder.

6. **Check what's already on host ports 80/443, 2399, 5003.** `docker run` will fail with
   "address already in use" if something's there — common on a Mac with other dev tools
   running. `lsof` without `sudo` can miss root-owned listeners and silently report nothing
   even when a port is genuinely occupied — confirm with an actual connection attempt too,
   not just `lsof` alone:
   ```bash
   lsof -iTCP:80 -sTCP:LISTEN -P 2>&1
   lsof -iTCP:443 -sTCP:LISTEN -P 2>&1
   nc -z -w 2 localhost 80 && echo "80 occupied" || echo "80 free"
   nc -z -w 2 localhost 443 && echo "443 occupied" || echo "443 free"
   ```
   To see *what* holds a port without `sudo`, ask it: `curl -sI http://localhost/ | grep -i
   '^server'`. On a Mac, port 80 is often the built-in macOS Apache (`Server: Apache/2.4.x
   (Unix)`) — seen 2026-09-24. That's why the verified container publishes `8080:80`, and it's
   why typing `http://localhost/admin-console` into a browser returns a 404 from Apache rather
   than anything from FMS. Always hand the developer the `https://` URL.

   If either is occupied, this has a safe, low-stakes default: remap the *host* side only (e.g.
   `8080:80`, `8443:443` — see Step 3) and **tell the developer which ports you chose and why**
   rather than silently deciding — they may prefer to free the conflicting port instead. Don't
   remap `2399`/`5003` without asking — FileMaker Pro clients expect those exact ports.

   **This conflict can be transient** — confirmed in practice: port 443 was occupied on first
   setup, then free again roughly a day later with nothing else about the Mac's configuration
   changed. Don't treat an earlier remap as permanent; re-check before assuming `8443` is still
   necessary, and if 443 is free, recreate the container publishing it directly — the
   disaster-recovery `docker run` in the troubleshooting reference, with `<HTTPS_PORT>` set to
   `443` — to drop the port number from every URL. Commit first (Step 5) and confirm with the
   developer, since this removes the current container.

7. **Check for existing FMS containers, volumes and images.** Anything left from an earlier
   attempt changes what a "fresh" install actually does:
   ```bash
   docker ps -a --format '{{.Names}}\t{{.Image}}\t{{.Status}}'
   docker volume ls --filter name=fms-
   docker images fmsdocker
   ```
   **Existing `fms-*` volumes are the dangerous one.** `docker volume create fms-data` on a name
   that already exists is a silent no-op, so Step 2 appears to succeed and the new install lands
   on the old licence, admin account and databases — the installer then offers "load previous
   configuration" on what the developer thinks is a clean start. If any exist, stop and ask the
   developer which they want: reuse them (a deliberate reinstall — see the troubleshooting
   reference), use new volume names, or remove them — and only after a backup (U3's procedure)
   and an explicit yes. Never remove a volume on your own judgement. An existing `fmsdocker:installed`
   image may mean there's nothing to install at all — see "FileMaker Server disappeared" in the
   troubleshooting reference.

## Decisions that need the developer's input, not an assumed default

A few points below have no universally-correct answer — surface them explicitly rather than
picking silently, even though each has a reasonable default worth suggesting:

- **Single primary server, or primary + secondary/failover?** Default to a single primary
  unless the developer specifically wants to test multi-server/failover features — state that
  assumption and let them correct it, rather than asking as a blocking question every time
  (this one has a clear, low-risk default).
- **Data storage: named Docker volumes vs. a host-visible folder** (Step 2) — ask, and
  recommend named volumes. They're the only option this runbook has run. A host folder under
  `~/Desktop/...` gives Finder-visible files, but it's a bind mount, and a bind mount is exactly
  what caused the Admin Console disk-usage false alarm (see the troubleshooting reference) — expect
  the same there. That path is untested here: no commands for it, and ownership/permissions for
  FMS's `fmserver` user haven't been checked. If what the developer actually wants is
  Finder-visible copies of the data, U3's backup tars give them that without a bind mount.
- **Container resource limits** (`--memory`, `--cpus` in Step 3) — compute a suggested value
  from the actual host resources checked in step 2 above, state the reasoning ("Docker Desktop
  has X allocated, suggesting Y"), and let the developer adjust rather than hardcoding a fixed
  number regardless of what the host can actually support.
- **Admin Console username/password/PIN** (Step 4) — never invent these. Either ask the
  developer directly, or if the assisted-install wizard is running interactively in their own
  terminal, let them type their own answers at the prompts.
- **If a container with the target name already exists** — before removing or recreating it,
  check what's actually in it first (`docker inspect <name> --format 'Mounts={{.Mounts}}'`,
  `docker ps -a`). An empty container with no mounts and nothing installed is low-stakes to
  replace; a container with named volumes attached, or one you didn't create this session, is
  not — confirm with the developer before removing anything that might hold state, even if it
  looks disposable at a glance.

## Step 1 — Build the image from Claris's Dockerfile

```bash
cd "<path to the extracted FMS package>/Docker"
docker build -t fmsdocker:prep .
```

This installs Ubuntu 24.04 + all of FMS's OS-level dependencies (nginx, apache2-bin, openssl,
init/systemd, etc.) per Claris's own Dockerfile — no changes needed to the Dockerfile itself.

**Never substitute `ubuntu:latest` for `ubuntu:24.04` anywhere in this process.** `:latest`
drifts over time — at time of writing it resolved to Ubuntu 26.04, which this FMS build does
not support (Claris's installer only recognizes 22.04/24.04 and will refuse to proceed or
behave unpredictably on anything else). Claris's own Dockerfile already pins `FROM ubuntu:24.04`
correctly — just don't override it.

## Step 2 — Create persistent volumes

```bash
docker volume create fms-data
docker volume create fms-cstore
docker volume create fms-wpeconf
docker volume create fms-logs
```

These succeed silently even when the volumes already exist — pre-flight check 7 is what catches
that, not this step.

Use named Docker volumes, not host bind-mounts under `/opt`. Docker Desktop's default macOS
file-sharing only covers paths under `/Users` (and a few others) — a bind mount to `/opt/...`
either fails silently or requires extra Docker Desktop configuration. Named volumes sidestep
this entirely and are the simpler default. For the host-folder alternative and why it's not the
recommended one, see *Decisions* above.

## Step 3 — Run the container

```bash
docker run -d \
  --name fms \
  --hostname fms \
  --privileged \
  --restart unless-stopped \
  --stop-timeout 135 \
  --memory <MEM> \
  --cpus <CPUS> \
  -p <HTTP_PORT>:80 -p <HTTPS_PORT>:443 -p 2399:2399 -p 5003:5003 \
  --volume fms-data:"/opt/FileMaker/FileMaker Server/Data" \
  --volume fms-cstore:"/opt/FileMaker/FileMaker Server/CStore" \
  --volume fms-wpeconf:"/opt/FileMaker/FileMaker Server/Web Publishing/publishing-engine/conf" \
  --volume fms-logs:"/opt/FileMaker/FileMaker Server/Logs" \
  fmsdocker:prep
```

Fill the four placeholders from the pre-flight results and the developer's answers — never
paste a fixed value regardless of the host:

| Placeholder | Where it comes from | Seen on verified runs |
|---|---|---|
| `<MEM>` | Decisions → resource limits, from pre-flight check 2 | `4096m` (Docker Desktop had ~7.75GB) |
| `<CPUS>` | Same | `2` |
| `<HTTP_PORT>` | Pre-flight check 6: `80` if free, else e.g. `8080` | `8080` (macOS Apache held 80) |
| `<HTTPS_PORT>` | Pre-flight check 6: `443` if free, else e.g. `8443` | `8443` on the first run, `443` later |

State the values you chose and why before running it.

Notes on the flags:
- `--privileged` and a full init system (`/sbin/init`, systemd) are required for FMS's services
  to start correctly — Claris's Dockerfile already sets `CMD ["/sbin/init"]`.
- Port mapping: if 80/443 are already taken on the host, remap the *host* side only
  (e.g. `8080:80`, `8443:443`) — keep `2399` (ODBC) and `5003` (FileMaker client protocol)
  mapped 1:1 without remapping, since FileMaker Pro clients expect the standard port and can't
  easily be told to use a different one.
- If `docker run` fails with "address already in use", that's the host port conflict from the
  pre-flight check — adjust the host-side port and retry.

Verify systemd actually came up before proceeding:

```bash
docker exec fms bash -c "ps -p 1 -o pid,comm"
```

Expect `systemd` as PID 1. If it isn't, something about the image build or run flags is wrong —
don't proceed to install until this checks out.

## Step 4 — Install FileMaker Server

This step is interactive (a curses-style assisted-install wizard) and needs a real TTY. Run it
from the developer's own terminal, not through a non-interactive `docker exec`.

**When handing this step to the developer, name the exact terminal — every time.** Say which
tab or window to use and what its prompt should look like: inside the container it reads
`root@fms:/#`, while the developer's normal macOS shell reads something like `hostname:~ user$`.
If the agent opened a terminal tab already `docker exec -it`'d into the container, say "use the
tab showing `root@fms:/#`". Otherwise a developer with several terminals open will type the
command into their usual macOS shell — see "`command not found` for a Linux command" in the
troubleshooting reference for the symptom.

First copy the package in (agent can do this — it's non-interactive):

```bash
docker cp "<path to the extracted FMS package>/filemaker-server-<build>-arm64.deb" fms:/tmp/
```

Use `docker cp`, not a bind mount of the installer folder. A bind mount left attached is what
makes Admin Console report the Mac's own disk usage (see the disk-usage entry in the
troubleshooting reference), and removing it later means recreating the container.

Then hand this to the developer:

```bash
docker exec -it fms bash
# then, inside the container (prompt root@<hostname>:/#):
apt-get update && apt install /tmp/filemaker-server-<build>-arm64.deb
```

The wizard asks, in order: security check, license agreement, deployment type (primary
machine), Admin Console username/password/4-digit PIN, filter databases by privilege, remove
sample database, HTTPS tunneling, firewall (ufw) handling. Sensible defaults: accept the
security check and license, choose primary/standard deployment, decline HTTPS tunneling unless
specifically needed, and let the developer choose their own Admin Console credentials — don't
invent them.

**On 26.0.2 it warns about an outdated Nginx with known CVEs and asks whether to continue.**
Say yes. The warning is about Ubuntu 24.04's apt-provided Nginx, going by its version number
(1.24.0). Ubuntu backports security fixes into its 1.24.0 package without changing that upstream
number, so a version check alone can't tell a patched package from an unpatched one. Claris's
guidance changed between releases: 26.0.2 shipped `NginxUpdate.sh` to replace it (Step 6), and
26.0.3 dropped that script so Ubuntu's own updates apply instead — which is why the upgrade
section later lays out switching back to Ubuntu's package as one of two options. Follow
whatever the installed build expects. Whether 26.0.3+ still shows this warning on a fresh install hasn't been observed yet.

**Two lines the installer prints at the end don't apply inside Docker — don't act on them:**

- *"To configure FileMaker Server, open Admin Console at `https://172.17.0.2/admin-console`"* —
  that IP is the container's internal Docker bridge address and is **not reachable from the host
  browser**. Use `https://localhost[:port]/admin-console`. Copying the printed URL, finding it dead, and concluding the
  container is broken is an easy mistake.
- *"add current user to the group fmsadmin and then restart your system"* — bare-metal install
  boilerplate. There is no "current user" outside the container in this model and nothing needs
  a host restart. Safe to ignore.

## Step 5 — Lock in the install immediately

**Do this before anything else, even before configuring TLS.** FileMaker Server is installed via
`apt` directly into the *container's own writable layer* — not into the image, and not into any
volume. A future `docker rm` on this container (accidental, or during unrelated cleanup) silently
deletes the entire FMS software install — binaries, systemd units, everything — while leaving the
data volumes untouched, which is a deeply confusing partial-data-loss scenario to debug after the
fact.

Clear out the installer and apt's cache first — a commit captures everything in the writable
layer, and on the 26.0.3 upgrade leaving the ~540MB `.deb` in `/tmp` bloated the image by that
much:

```bash
docker exec fms bash -c "rm -f /tmp/filemaker-server-*.deb && apt-get clean"
docker commit --message "FMS installed - $(date '+%Y-%m-%d %H:%M')" fms fmsdocker:installed
```

Re-run this commit (after the same cleanup) after *every* further apt-level change to the container (the Nginx patch in
Step 6, any OS package installs) — it's cheap, and it's the only thing standing between "restart
the container" and "reinstall everything" if the container ever gets removed. Keep the
disaster-recovery `docker run` command (identical to Step 3, but pointing at `fmsdocker:installed`
instead of `fmsdocker:prep`) somewhere the developer can find it — see
`references/troubleshooting.md` for the full recreate-from-image snippet.

## Steps 6–9: after the install

Run these after Step 5, in order, and commit the container again after each apt-level change.
Each is version-dependent or optional, so check before running:

- **Step 6 — Patch Nginx** (26.0.2 and earlier; check first on 26.0.3+)
- **Step 7 — Promote the sample database** (if missing)
- **Step 8 — Trusted local HTTPS certificate** (mkcert)
- **Step 9 — Enable WebDirect / Web Publishing** (off by default; also Data API, OData, ODBC)

Full commands and checks: [references/post-install.md](references/post-install.md).

## Upgrading FileMaker Server in place

Snapshot the container, back up the four volumes, dry-run the new package, upgrade, verify,
then commit. Rollback is a re-tag of the snapshot. Agree the downtime with the developer
first, and don't use Admin Console's own update prompt. Full runbook (U1–U7, the Nginx
follow-on, rolling back): [references/upgrading.md](references/upgrading.md).

## Verifying the whole install

From the host, use `https://localhost[:port]/admin-console` — never the container IP the
installer prints (see Step 4).

```bash
docker exec fms systemctl is-active fmshelper          # expect: active
curl -sk -o /dev/null -w "%{http_code}\n" https://localhost[:port]/admin-console/   # expect: 200
echo | openssl s_client -connect localhost:<HTTPS_PORT> -servername localhost 2>/dev/null | openssl x509 -noout -issuer   # expect: your mkcert CA, not FMS's self-signed default
```

## If something's already gone wrong

See `references/troubleshooting.md` for the full detail behind every gotcha mentioned above,
organized by symptom — including the disaster-recovery `docker run` command for recreating the
container from a committed image, the disk-usage false-alarm fix, the Docker-Desktop-quit
recovery steps, and the `/etc/hosts` trailing-newline trap.

## Relationship to other skills

- **cadenceux-ottofms-docker-setup** — the next step once this skill's base install is working.
  Installing OttoFMS on top of this container hits several of the *same* traps this skill
  documents (no `sudo` in the container, apt-install-into-writable-layer), which is why that
  skill exists separately rather than duplicating this one's content — it assumes this skill's
  steps are already done and picks up from there.
- Not related to the `fmp-dev-*` skills or `claris-filemaker-pro` — those cover FileMaker Pro
  schema/script/layout authoring and calculation/script-step syntax, not server infrastructure.
  Don't route FileMaker Pro development questions here.

## Licence

This skill is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

Built and maintained by [Darrin Southern](https://www.linkedin.com/in/darrin-southern/) from [CadenceUX](https://cadenceux.com.au).
