---
compatibility: Claude Code
metadata:
  "Built and maintained": "Darrin Southern from CadenceUX"
  version: "1.5"
name: cadenceux-fms-docker-setup
description: |
  Installs, upgrades and configures Claris FileMaker Server in Docker on macOS with Docker
  Desktop, using Claris's official arm64 Ubuntu 24.04 build. Use when asked to install, set
  up, upgrade, update, run or troubleshoot FMS on Docker ("FMS on Docker", "Dockerize
  FileMaker Server", "update FMS in the container", "is my FMS current?"). Also trigger when
  FMS installer files (filemaker-server*.deb, or an fms_*.zip with a Docker folder) sit in the
  working directory, or on symptoms it resolves: a wrong Admin Console disk-usage percentage,
  "Cannot decrypt the private key file" on certificate import, a missing sample database,
  Nginx CVE warnings, or FMS vanishing after a container was recreated. Covers building from
  Claris's Dockerfile (their install script can't run on macOS), the install wizard, in-place
  upgrades with snapshot and volume backup, a trusted mkcert cert, and the
  container-vs-image data-loss trap. Not for FileMaker Pro schema/script/layout work. For
  OttoFMS on top, see cadenceux-ottofms-docker-setup.
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
> Linux release is [latest]. See *Upgrading FileMaker Server in place* below. Separately, this
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

## Step 6 — Patch Nginx (26.0.2 and earlier — check first on 26.0.3+)

**This step is version-dependent. Check the FMS build before running it.**

- **26.0.2 and earlier:** run it as written below.
- **26.0.3 and later:** Claris changed the approach. The 26.0.3 release notes say Linux installs
  no longer need `NginxUpdate.sh`, so Ubuntu's own security updates to Nginx apply normally, and
  the 26.0.3 package no longer ships `NginxUpdate.sh` at all. Seen directly during the 26.0.3
  upgrade: the installer's pre-install step printed *"Removing Nginx software source… pin-priority…
  signing key"* and deleted exactly what this step adds (`/etc/apt/sources.list.d/nginx.list`,
  `/etc/apt/preferences.d/99nginx`, the nginx.org keyring). **Don't re-add them on 26.0.3+
  without asking the developer** — the installer will just strip them again on the next upgrade,
  and it's working against Claris's supported path. On a fresh 26.0.3+ install, skip this step
  unless the wizard actually raises an Nginx CVE warning, and tell the developer which happened.
  See the upgrade section for the one follow-on decision this creates on containers that were
  patched under 26.0.2.

```bash
docker exec fms bash -c "
apt-get install -y curl gnupg2 ca-certificates lsb-release ubuntu-keyring &&
curl -fsSL https://nginx.org/keys/nginx_signing.key | gpg --dearmor -o /usr/share/keyrings/nginx-archive-keyring.gpg &&
echo \"deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/ubuntu \$(lsb_release -cs) nginx\" | tee /etc/apt/sources.list.d/nginx.list &&
printf 'Package: *\nPin: origin nginx.org\nPin-Priority: 900\n' | tee /etc/apt/preferences.d/99nginx &&
apt-get update &&
apt-get install -y nginx
"
docker restart fms
```

Two things that trip people up here — full explanation in the reference file, short version:
- This container has no `sudo` installed (you're already root via `docker exec`) — Claris's own
  `NginxUpdate.sh` script assumes `sudo` exists and silently no-ops without it. The commands
  above already have `sudo` stripped out.
- After the apt upgrade, restart the whole **container** (`docker restart fms`), not
  `systemctl restart nginx` inside it — that targets a different, disabled generic
  `nginx.service` unit that conflicts with FMS's own directly-spawned Nginx process (which
  already owns ports 80/443). A container restart makes `fmshelper` respawn its Nginx child
  against the newly-installed binary correctly.

Verify: `docker exec fms /usr/sbin/nginx -v` should report something well past 1.24.0.
Re-commit the image (Step 5) after this succeeds.

## Step 7 — Promote the sample database (if missing)

There are two different "missing" outcomes — check which one you have before fixing anything:

- **Stuck in staging** — the file exists hidden as `.Sample/en_FMServer_Sample` but was never
  promoted. Fixed below.
- **Absent entirely** — seen on a second run (FMS 26.0.2.219, arm64): `Data/Databases/Sample/`
  was an empty directory, there was no `.Sample` staging file, and `find "/opt/FileMaker/FileMaker
  Server" -iname '*.fmp12'` returned nothing. The cause is **not established** — it may be how
  the wizard's "remove sample database" prompt was answered, or this Docker/arm64 package may not
  ship one by default. Don't state either as fact. Treat it as a possible outcome that isn't
  necessarily a broken install, tell the developer, and don't hunt for a fix that isn't there.
  Confirming needs a run that deliberately answers "keep sample" and checks immediately.

Check first:

```bash
docker exec fms bash -c "ls '/opt/FileMaker/FileMaker Server/Data/Databases/' 2>&1"
```

If `FMServer_Sample.fmp12` isn't there, it's likely stuck unpromoted in a hidden staging folder
— common when installing on top of pre-existing `Data` (e.g. "load previous configuration"
during a reinstall):

```bash
docker exec fms bash -c "
cp '/opt/FileMaker/FileMaker Server/Data/Databases/.Sample/en_FMServer_Sample' '/opt/FileMaker/FileMaker Server/Data/Databases/FMServer_Sample.fmp12'
chown fmserver:fmsadmin '/opt/FileMaker/FileMaker Server/Data/Databases/FMServer_Sample.fmp12'
chmod 660 '/opt/FileMaker/FileMaker Server/Data/Databases/FMServer_Sample.fmp12'
"
docker restart fms
```

Swap `en_` for the developer's preferred locale prefix if not English. Confirm it actually
opened by checking the log, not just that the file exists:

```bash
docker exec fms bash -c "tail -30 '/opt/FileMaker/FileMaker Server/Logs/Event.log' | grep -i sample"
```

Look for `Opened database "FMServer_Sample"`.

## Step 8 — Set up a trusted local HTTPS certificate

The Admin Console's default self-signed cert triggers browser warnings. Use `mkcert` for a
locally-trusted certificate instead — but the certificate **import** path has a real bug worth
knowing about upfront rather than discovering through trial and error.

**The broken path (don't waste time on it):** generating a key externally (via `mkcert` or
`openssl`, any cipher — unencrypted, PKCS#8/AES, legacy PKCS#1/3DES, legacy PKCS#1/AES-128, all
tested) and importing with `fmsadmin certificate import <cert> --keyfile <key> --keyfilepass
<pass>` reliably fails with `Cannot decrypt the private key file... with the password` (error
20408), even when the key/password pair is independently verified correct with OpenSSL. This
looks like a genuine bug or unsupported path specific to `--keyfile` external-key import.

**Getting mkcert (once per Mac).** Check first: `command -v mkcert`. If it's missing:

- **`brew` already installed** (`command -v brew`) → `brew install mkcert`.
- **No Homebrew** → don't install Homebrew just for this; a system-wide package manager is a much
  bigger ask than one binary. Use the direct download instead. It comes from the mkcert
  maintainer's own stable redirect (documented in mkcert's release notes) and needs no sudo and
  no package manager:

  ```bash
  mkdir -p ~/bin && cd ~/bin
  curl -fsSL "https://dl.filippo.io/mkcert/latest?for=darwin/arm64" -o mkcert
  chmod +x mkcert
  file mkcert          # confirm a real Mach-O executable of plausible size before running it
  ./mkcert -version
  ```

  Swap `darwin/arm64` for `darwin/amd64`, `linux/amd64` etc. to match the host.
  `/usr/local/bin` is root-owned on stock macOS, so the binary stays in `~/bin/`. Call it by full
  path (`~/bin/mkcert`) unless `~/bin` is actually on `PATH`. The commands below write `mkcert`;
  substitute the full path where needed.

**The working path:** let FMS generate its own key, and only bring your own signed certificate.
Three places run commands here — label every handoff with which one:

- **Developer's Mac terminal** (their own macOS shell): `mkcert -install` — it changes the macOS
  trust store, so the developer runs it, not the agent, even though it's low-risk.
- **Developer's container tab** (prompt `root@<hostname>:/#`, opened with `docker exec -it fms
  bash`): the two `fmsadmin` commands — both ask for the Admin Console username and password.
- **Agent** (non-interactive, on the Mac): everything else.

```bash
# 1. DEVELOPER, Mac terminal — creates mkcert's local CA if it doesn't exist yet, and trusts it.
#    This must come first: step 3 signs with the CA files this creates.
mkcert -install

# 2. DEVELOPER, container tab (root@<hostname>:/#) — FMS generates its own CSR + key:
"/opt/FileMaker/FileMaker Server/Database Server/bin/fmsadmin" certificate create localhost --keyfilepass <a passphrase>
# answer y, then the Admin Console username/password when prompted
# creates serverRequest.pem and serverKey.pem in CStore/ — leave serverKey.pem alone

# 3. AGENT, Mac — pull the CSR out and sign it with mkcert's CA, injecting a SAN list
#    (Chrome/Safari require SAN entries — a CN-only cert is rejected regardless of match):
docker cp fms:"/opt/FileMaker/FileMaker Server/CStore/serverRequest.pem" ./serverRequest.pem
CAROOT=$(mkcert -CAROOT)
ls "$CAROOT/rootCA.pem" "$CAROOT/rootCA-key.pem"   # both must exist — if not, step 1 hasn't run
printf 'subjectAltName=DNS:localhost,DNS:fms,IP:127.0.0.1,IP:::1\n' > san.ext
openssl x509 -req -in serverRequest.pem \
  -CA "$CAROOT/rootCA.pem" -CAkey "$CAROOT/rootCA-key.pem" -CAcreateserial \
  -out serverSigned.pem -days 730 -extfile san.ext

# 4. AGENT, Mac — copy the signed cert into the CStore volume, not /tmp (an unclean restart
#    wipes /tmp — see "/tmp contents vanished" in the troubleshooting reference):
docker cp serverSigned.pem "fms:/opt/FileMaker/FileMaker Server/CStore/serverSigned.pem"
docker exec fms chown fmserver:fmsadmin "/opt/FileMaker/FileMaker Server/CStore/serverSigned.pem"

# 5. DEVELOPER, container tab — import it. No --keyfile: FMS already has the matching key from step 2.
"/opt/FileMaker/FileMaker Server/Database Server/bin/fmsadmin" certificate import "/opt/FileMaker/FileMaker Server/CStore/serverSigned.pem" --keyfilepass <same passphrase>
```

Restart the container when prompted (`docker restart fms`). Any `/etc/hosts` edit for a local
DNS alias is also the developer's to make, in their Mac terminal (see the reference file for a
subtle trailing-newline trap there).

To add more SAN names later (e.g. a custom local hostname), re-sign the *same* CSR with an
expanded SAN list and re-import with the same passphrase — no need to redo `certificate create`.

Re-commit the image (Step 5) once the certificate is confirmed working.

## Step 9 — Enable WebDirect / Web Publishing (optional, but off by default)

If the developer wants WebDirect, Custom Web Publishing, the Data API, or OData, don't assume
any of them are on just because the install finished cleanly — **every publishing component is
disabled by default** even after a full, successful assisted-install. Trying to load
`/fmi/webd` against a fresh install returns a `502 Bad Gateway` — Nginx is proxying correctly,
but there's nothing listening on the other end, because the Web Publishing Engine process was
never started.

Confirm the actual state rather than guessing from the install wizard's behaviour:

```bash
docker exec fms bash -c "cat '/opt/FileMaker/FileMaker Server/Admin/conf/deployment.xml'" | grep -oE '<component[^>]*name=\"wpe\"[^>]*>'
```

If it shows `enabled="no"`, turn it on — either in Admin Console (a toggle under Configuration),
or via the CLI (interactive, needs a real TTY and Admin Console credentials, same pattern as
Step 4):

```bash
docker exec -it fms bash
# then, inside the container:
"/opt/FileMaker/FileMaker Server/Database Server/bin/fmsadmin" enable wpe
```

**Startup delay is real but variable — don't trust a single immediate check, from either
direction.** Confirmed across two separate restarts: OData and the Data API sometimes come up
within ~2-3 seconds of the database engine starting, and sometimes take 30-45+ seconds — same
`enabled="yes"` config, same container, just different timing between restarts. There's no
fixed number worth hardcoding as "wait N seconds." Poll with retries over roughly a minute
before concluding anything is actually broken:

```bash
curl -sk -o /dev/null -w "%{http_code}\n" https://localhost[:port]/fmi/webd                          # WebDirect
curl -sk -o /dev/null -w "%{http_code}\n" https://localhost[:port]/fmi/odata/v4/                      # OData
curl -sk -o /dev/null -w "%{http_code}\n" https://localhost[:port]/fmi/data/vLatest/productInfo       # Data API
```

This cuts both ways — it also means a developer who manually started a component after finding
it apparently "off" isn't proof anything had actually reverted. Before concluding a setting
didn't persist, check `deployment.xml`'s `enabled` attribute first (see above) — if it already
says `yes`, the config was fine and what looked like a dropped setting was almost certainly just
this startup-timing variability, not an actual regression. Restarting again and polling with
retries (rather than a single immediate check) is the way to tell the two apart.

The same `enabled="no"`-by-default pattern applies to the other components listed in
`deployment.xml` (`xdbc` for ODBC/JDBC, `fmdapi` for the Data API, `odata` for OData) — check
and enable whichever the developer actually needs; don't assume any of them came on with the
base install.

## Upgrading FileMaker Server in place

Verified once, 2026-09-24: 26.0.2.219 → 26.0.3.309 on the arm64/macOS setup above, with the
mkcert certificate, WebDirect/Data API enabled and OttoFMS installed — all survived. The
upgrade is the same `apt install` of the new `.deb` over the running install; what makes it a
Docker job is doing it so it can be rolled back, and not losing it afterwards.

**Don't use Admin Console's own update prompt for this.** Stick to the steps below so the
snapshot and volume backup exist before anything changes.

**Agree the downtime first.** U3 stops the container, and U5 restarts FMS — connected FileMaker
clients get disconnected both times. How long depends on data size (the verified run had tiny
volumes, so the U3 stop was brief; the U5 outage wasn't timed). Ask the developer when that's
acceptable rather than starting straight away.

### U1 — Get the new package and compare it with the old one

The developer supplies the `fms_<version>_Ubuntu24_arm64.zip` (licence-linked download — see the
staleness check). Unzip it next to the previous package and diff what matters:

```bash
diff fms_<old>/Docker/Dockerfile fms_<new>/Docker/Dockerfile        # changed → the prep image may need rebuilding; stop and plan that first
diff "fms_<old>/Assisted Install.txt" "fms_<new>/Assisted Install.txt"
ls fms_<old> fms_<new>                                               # helper scripts added/removed
```

26.0.2 → 26.0.3: Dockerfile identical; `Assisted Install.txt` gained two accessibility keys;
`NginxUpdate.sh` was dropped (see Step 6). Also compare the package's dependencies against the
installed ones once the `.deb` is in the container (U4).

**Old package no longer on disk?** Compare the new Dockerfile against the running container
instead. Its `FROM` line must match the container's OS (`grep ^FROM fms_<new>/Docker/Dockerfile`
vs `docker exec fms grep ^VERSION_ID /etc/os-release`), and every package in its
`apt-get install` list should already be installed:

```bash
awk '/apt-get install --no-install-recommends -y/{f=1;next} f{line=$0; gsub(/[\\&]/,"",line); n=split(line,a," "); for(i=1;i<=n;i++) print a[i]; if($0 ~ /&&/) exit}' fms_<new>/Docker/Dockerfile \
| docker exec -i fms xargs dpkg-query -W -f='${db:Status-Abbrev} ${Package}\n' 2>&1 | grep -v '^ii '
```

No output means everything is installed; any line printed is a missing package. (Verified
2026-09-24 against the 26.0.3.309 Dockerfile: all 24 present, and a deliberately fake name was
reported. The list goes through `xargs` because zsh — macOS's default shell — doesn't split an
unquoted `$var` into words, which silently turns a package list into one bogus name.) Anything
missing means the prep image is out of date for this release — stop and plan that before going
further.

**Skipping a release, or a bigger jump.** Only a single step (26.0.2 → 26.0.3) has been run.
For a jump across several releases:

- Read the release notes for *every* release in between, not just the newest — changes like
  26.0.3's Nginx one land in the release that introduced them.
- Apply version-dependent steps by what the jump *crosses*. Going from 26.0.2 or earlier to
  anything 26.0.3 or later means Step 6 no longer applies and the Nginx follow-on decision below
  does.
- Check Claris's installation guide for the new release for any supported-upgrade-path limits
  before assuming a direct jump is allowed. This runbook doesn't know them.
- **A new major version, or a Dockerfile whose `FROM` changes**, may not upgrade in place with
  `apt` at all. That would mean building a new prep image and installing into a new container
  on the same (backed-up) volumes — a path that hasn't been run. Tell the developer it's
  untested and plan it with them rather than improvising.

### U2 — Snapshot the container's software

```bash
STAMP=$(date '+%Y%m%d-%H%M')
docker exec fms bash -c "rm -f /tmp/*.deb && apt-get clean"
docker commit --message "FMS <old build> before upgrade - $STAMP" fms fmsdocker:pre-upgrade-$STAMP
docker tag fmsdocker:pre-upgrade-$STAMP fmsdocker:installed
echo "rollback tag: fmsdocker:pre-upgrade-$STAMP"
```

Set `STAMP` once and reuse it. Building the timestamp separately for the commit and the tag
breaks the tag whenever the minute rolls over between the two commands. Shell variables don't
survive between separate agent tool calls, so run all of these lines in one call and note the
printed tag name for rollback. The dated tag is the rollback point. It stays put when `:installed` moves on after the upgrade.

### U3 — Back up the four data volumes (a commit doesn't include them)

Databases, licence, certificates and admin account live in the volumes, not the image. Stop the
container so no database is open mid-copy, archive each volume with a throwaway container from
the already-present `fmsdocker:prep` image (no extra image pull), then start it again.

Choose the backup folder with the developer — somewhere on the Mac they'll find again, and under
`/Users` so Docker Desktop can mount it (the verified run used a `backups/` folder next to the
installer packages). Use an absolute path. If `fmsdocker:prep` is gone (`docker images
fmsdocker`), any image with `tar` works for the throwaway container, e.g. `fmsdocker:installed`.

```bash
B="<absolute backup folder>/pre-<new version>-$(date '+%Y%m%d-%H%M')"; mkdir -p "$B"
docker stop -t 120 fms
for v in fms-cstore fms-data fms-logs fms-wpeconf; do
  docker run --rm -v $v:/src:ro -v "$B":/dst fmsdocker:prep tar -czf /dst/$v.tgz -C /src .
done
docker start fms
for f in "$B"/*.tgz; do echo "$(basename $f): $(tar -tzf $f | wc -l) entries"; done
echo "backup folder: $B"
```

Run the whole block in one call (it relies on `$B` throughout), and note the printed backup
folder — a rollback needs that exact path later.

Check `fms-data.tgz` actually lists the `.fmp12` files before moving on
(`tar -tzf "$B/fms-data.tgz" | grep -i fmp12`). Confirm the Admin Console answers again after
the start (`curl -sk -o /dev/null -w "%{http_code}\n" https://localhost[:port]/admin-console/`).

### U4 — Copy the package in and dry-run it

```bash
docker cp fms_<new>/filemaker-server-<new build>-arm64.deb fms:/tmp/
docker exec fms bash -c "apt-get update -qq && apt-get install -s /tmp/filemaker-server-<new build>-arm64.deb | grep -E '^(Inst|Remv)|upgraded'"
```

Expect only `Inst filemaker-server [<old>] (<new> …)` and nothing under `Remv`. If the dry run
wants to remove packages or pull in a large set of new ones, stop and show the developer before
installing. (26.0.2 → 26.0.3: one package upgraded, nothing removed or added — the new
dependency list only swapped OpenCV for `libopenjp2-7`, already present.)

### U5 — Run the upgrade (interactive — developer's terminal)

```bash
docker exec -it fms bash -c "apt install /tmp/filemaker-server-<new build>-arm64.deb"
```

Same handoff rules as Step 4: name the exact terminal tab. The licence agreement prompt is the
developer's to answer — never answer it for them. The installer then preserves the existing
deployment on its own; on 26.0.3 it printed *"Current Admin Console account information will be
preserved"* and *"Preserving current … deployment configuration"*, kept the existing MachineID,
and kept a copy of the old keystore as `CStore/keystore_26.0.2.backup`.

**Output that looks alarming but isn't:**

- `prerm: … /usr/bin/crontab: No such file or directory` — the old package's removal script tries
  to delete its entry from root's crontab, and the container has no `cron`. There was nothing to
  delete; the upgrade carries on.
- *"Removing Nginx software source / pin-priority / signing key"* (26.0.3+) — expected; see Step 6.
- The container-bridge Admin Console URL and "add current user to fmsadmin … restart" — same
  bare-metal boilerplate as Step 4. Ignore both.
- The progress bar stops at 80% and returns to the prompt. That's normal — check with `dpkg`
  (U6), not the bar.

### U6 — Verify

```bash
docker exec fms bash -c "dpkg -l filemaker-server | tail -1; dpkg --audit; systemctl is-active fmshelper"
curl -sk -o /dev/null -w "%{http_code}\n" https://localhost[:port]/admin-console/signin
curl -sk https://localhost[:port]/fmi/data/vLatest/productInfo
echo | openssl s_client -connect localhost:<HTTPS_PORT> -servername localhost 2>/dev/null | openssl x509 -noout -issuer
docker exec fms bash -c "ls '/opt/FileMaker/FileMaker Server/Data/Databases/'"
```

Expect: `ii` with the new build, no `dpkg --audit` output, `active`, 200, a Data API `OK`, the
mkcert issuer (not FMS's default), and the databases still present. If OttoFMS is installed,
check `https://localhost[:port]/otto/` too. Then have the developer sign in to Admin Console
and confirm the databases are open and the version shown matches.

### U7 — Clean up, then commit

**Delete the `.deb` from `/tmp` *before* committing.** Committing first bakes the ~540MB
installer into the image — seen on the 26.0.3 run (6.6GB image vs 6.06GB after redoing it). A
re-commit after deleting does shrink it, because each commit captures the container's full
diff from its base image, not a diff on top of the previous commit.

```bash
docker exec fms bash -c "rm -f /tmp/filemaker-server-*.deb && apt-get clean"
docker commit --message "FMS <new build> upgraded in place - $(date '+%Y-%m-%d %H:%M')" fms fmsdocker:<new build>
docker tag fmsdocker:<new build> fmsdocker:installed
```

The disaster-recovery `docker run` in the troubleshooting reference now restores the new build.

### One follow-on decision when upgrading from 26.0.2 or earlier to 26.0.3+ (Nginx)

A container patched under 26.0.2 (Step 6) keeps the newer nginx.org build after the upgrade
(1.30.4 on the verified run), but 26.0.3 removed the nginx.org repository, so that build now
receives **no further updates**: Ubuntu's own `nginx` (1.24.0 with backported security fixes)
has a lower version number, so apt never replaces it. Check with
`docker exec fms apt-cache policy nginx`. Don't pick for the developer — present both:

- **Follow Claris's 26.0.3 model — switch to Ubuntu's `nginx`.** Gets Ubuntu's ongoing security
  fixes. CVE scanners reading the version string may still flag "1.24.0", even though Ubuntu
  backports fixes into it. Verified 2026-09-24 (procedure below).
- **Keep the nginx.org build:** newer today, but frozen — and re-adding the repository gets
  stripped again by the next FMS upgrade. Not run.

#### Switching to Ubuntu's `nginx` (verified)

The rollback point is the `fmsdocker:<new build>` commit from U7, as long as nothing has changed
since — otherwise commit first. Dry-run, taking the exact version from apt rather than typing it:

```bash
docker exec fms bash -c 'apt-get update -qq; apt-cache madison nginx'   # the Ubuntu line, e.g. 1.24.0-2ubuntu7.18
docker exec fms bash -c 'apt-get install -s --allow-downgrades nginx=<ubuntu version> | grep -E "^(Inst|Remv)|DOWNGRADED"'
```

Expect `nginx` downgraded, `nginx-common` newly installed, nothing removed. Then install with
**service starts blocked**. Ubuntu's package tries to (re)start the generic `nginx.service`,
which would fight FMS's own Nginx for ports 80/443 (the same conflict as Step 6's
`systemctl restart` warning). A `policy-rc.d` returning 101 is the standard Debian/Docker way to
stop package scripts starting services:

```bash
docker exec fms bash -c '
printf "#!/bin/sh\nexit 101\n" > /usr/sbin/policy-rc.d && chmod +x /usr/sbin/policy-rc.d
DEBIAN_FRONTEND=noninteractive apt-get install -y --allow-downgrades \
  -o Dpkg::Options::=--force-confdef -o Dpkg::Options::=--force-confold nginx=<ubuntu version>
rm -f /usr/sbin/policy-rc.d'
docker exec fms ls /usr/sbin/policy-rc.d   # must fail with "No such file" — left behind, it silently blocks every future service start from apt
```

**Always remove `policy-rc.d` as its own step, never behind `set -e` or `&&` after a status
check.** Seen on the verified run: `systemctl is-active nginx` correctly returned `inactive`
(exit code 3), a `set -e` script aborted on it, and the `rm` after it never ran.

Expected output, all fine: `policy-rc.d returned 101, not running 'restart nginx.service'`, and
*"Not attempting to start NGINX, port 80 is already in use"*. Ubuntu's package replaces the
default files under `/etc/nginx/` (`nginx.conf`, `mime.types`, `fastcgi_params`); FMS doesn't use
them — its Nginx runs from `NginxServer/conf/fms_nginx.conf`.

Then check, restart and verify:

```bash
docker exec fms bash -c 'systemctl is-enabled nginx; /usr/sbin/nginx -v; dpkg --audit'   # disabled; nginx/1.24.0 (Ubuntu); no output
docker restart -t 135 fms
docker exec fms bash -c 'ps -eo args | grep "[n]ginx: master"'   # /usr/sbin/nginx -c …/fms_nginx.conf
```

Run the U6 checks, plus `grep 'Opened database' …/Logs/Event.log | tail` with a timestamp after
the restart. On the verified run the Admin Console was back in about 20 seconds; Data API,
WebDirect, OttoFMS and the mkcert certificate all came back unchanged. Finish with
`apt-get clean` and a commit (e.g. `fmsdocker:<new build>-ubuntu-nginx`, then retag
`:installed`).

### Rolling back

Verified 2026-09-24 in an isolated test container (26.0.3.309 → 26.0.2.219): recreated from the
dated `pre-upgrade` tag onto volumes restored from the U3 tars, it came back as 26.0.2.219 in
about 20 seconds, with `fmshelper` active, the database opened and the Data API answering. The
test ran with `--network none` beside the live server, so the steps below were proven on copies;
a rollback of the real container hasn't been needed yet.

**Rolling back replaces the running container, and possibly the data — confirm with the developer
before starting, and say which of the two cases below applies.**

1. **Keep the failed state first.** Commit the current container before removing it, so what
   went wrong can still be looked at later:
   ```bash
   docker exec fms bash -c "rm -f /tmp/*.deb && apt-get clean"
   docker commit --message "failed upgrade state - $(date '+%Y-%m-%d %H:%M')" fms fmsdocker:failed-$(date '+%Y%m%d-%H%M')
   ```
2. **Record the settings** with the `docker inspect` commands in the troubleshooting reference
   (ports, memory, CPUs, mounts), then stop and remove the container:
   ```bash
   docker stop -t 135 fms && docker rm fms
   ```
3. **Data too? Restore the volumes** from U3 — only if the upgrade changed or damaged data, not by
   default: the upgrade itself preserves the volumes. The container must be stopped (it is, after
   step 2). For each of the four volumes:
   ```bash
   docker run --rm -v <vol>:/dst -v "<U3 backup folder>":/src fmsdocker:prep sh -c "rm -rf /dst/* /dst/.[!.]* ; tar -xzf /src/<vol>.tgz -C /dst"
   ```
   `<U3 backup folder>` is the absolute path U3 printed — the `$B` variable from U3 won't exist in
   a later shell. This empties the volume before extracting, so it also works over a volume that still holds
   the newer data. Check it with `find ! -type d`, not `find -type f`: the backups include
   symlinks (OttoFMS's file-manager links in `fms-data`) and a named pipe (`.passphrase` in
   `fms-cstore`), which `-type f` skips — that looked like missing files on the verified run
   until a proper comparison showed every entry restored.
4. **Recreate** with the disaster-recovery `docker run` in the troubleshooting reference, using the
   recorded settings and the `fmsdocker:pre-upgrade-<STAMP>` tag U2 printed in place of
   `fmsdocker:installed`.
5. **Verify** with U6, expecting the *old* build, then point `:installed` back at it:
   `docker tag fmsdocker:pre-upgrade-<STAMP> fmsdocker:installed`.

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
