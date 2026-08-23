---
compatibility: Claude Code
metadata:
  "Built and maintained": "Darrin Southern from CadenceUX"
  version: "1.0"
name: cadenceux-fms-docker-setup
description: |
  Installs and configures Claris FileMaker Server in Docker on macOS with Docker Desktop,
  using Claris's official arm64 Ubuntu 24.04 build. Use when asked to install, set up, run, or
  troubleshoot FileMaker Server on Docker — phrases like "FMS on Docker", "install FMS via
  Docker", "Dockerize FileMaker Server". Also trigger when FileMaker Server installer files
  (filemaker-server*.deb, or an fms_*.zip with a Docker folder) sit in the working directory,
  or on symptoms this runbook resolves: Admin Console showing a wrong disk-usage percentage,
  "Cannot decrypt the private key file" during certificate import, a missing sample database,
  Nginx CVE warnings, or FMS software vanishing after a container was recreated. Covers
  building from Claris's own Dockerfile (their install script can't run on macOS), the
  assisted-install wizard, a trusted local HTTPS cert via mkcert, and the container-vs-image
  data-loss trap. Not for FileMaker Pro schema/script/layout work. For OttoFMS on top, see
  cadenceux-ottofms-docker-setup.
---

# FileMaker Server on Docker — Setup

A verified, ordered runbook for getting Claris FileMaker Server running in a Docker container,
built from a real session that hit (and resolved) every trap documented here. It replaces the
generic three-year-old community walkthroughs that predate Claris's own arm64 build and don't
cover the macOS-specific failure modes below.

**Verified environment:** macOS (Apple Silicon/arm64), Docker Desktop, FileMaker Server
26.0.2.219 (Claris's official `fms_*_Ubuntu24_arm64` package), Ubuntu 24.04 base image. The
same approach should generalize to Intel Macs and native Linux Docker hosts (the container
itself is architecture-neutral once you match the right FMS package to `uname -m`), but only
the arm64/macOS path has actually been run start-to-finish. Flag this to the developer if the
host differs, rather than assuming identical behaviour.

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
   If either is occupied, this has a safe, low-stakes default: remap the *host* side only (e.g.
   `8080:80`, `8443:443` — see Step 3) and **tell the developer which ports you chose and why**
   rather than silently deciding — they may prefer to free the conflicting port instead. Don't
   remap `2399`/`5003` without asking — FileMaker Pro clients expect those exact ports.

   **This conflict can be transient** — confirmed in practice: port 443 was occupied on first
   setup, then free again roughly a day later with nothing else about the Mac's configuration
   changed. Don't treat an earlier remap as permanent; re-check before assuming `8443` is still
   necessary, and if 443 is free, recreate the container publishing it directly (same
   disaster-recovery `docker run` command as Step 5, just swap `-p 8443:443` for `-p 443:443`)
   to drop the port number from every URL.

## Decisions that need the developer's input, not an assumed default

A few points below have no universally-correct answer — surface them explicitly rather than
picking silently, even though each has a reasonable default worth suggesting:

- **Single primary server, or primary + secondary/failover?** Default to a single primary
  unless the developer specifically wants to test multi-server/failover features — state that
  assumption and let them correct it, rather than asking as a blocking question every time
  (this one has a clear, low-risk default).
- **Data storage: named Docker volume vs. a host-visible folder** (Step 2) — genuinely ask.
  Named volumes are simpler and avoid macOS file-sharing permission issues; a host folder under
  `~/Desktop/...` or similar gives Finder-visible files for easy backup/browsing, at the cost of
  a small amount of extra setup. Don't default this one silently — it affects how the developer
  interacts with their own data going forward.
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

Use named Docker volumes, not host bind-mounts under `/opt`. Docker Desktop's default macOS
file-sharing only covers paths under `/Users` (and a few others) — a bind mount to `/opt/...`
either fails silently or requires extra Docker Desktop configuration. Named volumes sidestep
this entirely and are the simpler default. (A host folder under `~/Desktop/...` is a legitimate
alternative if the developer wants Finder-visible files — offer it, don't assume it.)

## Step 3 — Run the container

```bash
docker run -d \
  --name fms \
  --hostname fms \
  --privileged \
  --restart unless-stopped \
  --stop-timeout 135 \
  --memory 4096m \
  --cpus 2 \
  -p 8080:80 -p 8443:443 -p 2399:2399 -p 5003:5003 \
  --volume fms-data:"/opt/FileMaker/FileMaker Server/Data" \
  --volume fms-cstore:"/opt/FileMaker/FileMaker Server/CStore" \
  --volume fms-wpeconf:"/opt/FileMaker/FileMaker Server/Web Publishing/publishing-engine/conf" \
  --volume fms-logs:"/opt/FileMaker/FileMaker Server/Logs" \
  fmsdocker:prep
```

Notes on the flags:
- `--privileged` and a full init system (`/sbin/init`, systemd) are required for FMS's services
  to start correctly — Claris's Dockerfile already sets `CMD ["/sbin/init"]`.
- Port mapping: if 80/443 are already taken on the host, remap the *host* side only
  (`8080:80`, `8443:443` above) — keep `2399` (ODBC) and `5003` (FileMaker client protocol)
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
from the developer's own terminal, not through a non-interactive `docker exec`:

```bash
docker exec -it fms bash
# then, inside the container:
apt-get update && apt install /path/to/filemaker-server-*.deb
```

(If the `.deb` isn't already reachable inside the container, `docker cp` it in first, or bind-mount
the folder temporarily — see the disk-usage warning in the troubleshooting reference before
leaving any bind mount attached long-term.)

The wizard asks, in order: security check, license agreement, deployment type (primary
machine), Admin Console username/password/4-digit PIN, filter databases by privilege, remove
sample database, HTTPS tunneling, firewall (ufw) handling. Sensible defaults: accept the
security check and license, choose primary/standard deployment, decline HTTPS tunneling unless
specifically needed, and let the developer choose their own Admin Console credentials — don't
invent them.

**It will likely warn about an outdated bundled Nginx with known CVEs and ask whether to
continue.** Say yes — this is expected on Ubuntu 24.04's apt-provided Nginx (1.24.0), and gets
patched in Step 6.

## Step 5 — Lock in the install immediately

**Do this before anything else, even before configuring TLS.** FileMaker Server is installed via
`apt` directly into the *container's own writable layer* — not into the image, and not into any
volume. A future `docker rm` on this container (accidental, or during unrelated cleanup) silently
deletes the entire FMS software install — binaries, systemd units, everything — while leaving the
data volumes untouched, which is a deeply confusing partial-data-loss scenario to debug after the
fact.

```bash
docker commit --message "FMS installed - $(date '+%Y-%m-%d %H:%M')" fms fmsdocker:installed
```

Re-run this commit after *every* further apt-level change to the container (the Nginx patch in
Step 6, any OS package installs) — it's cheap, and it's the only thing standing between "restart
the container" and "reinstall everything" if the container ever gets removed. Keep the
disaster-recovery `docker run` command (identical to Step 3, but pointing at `fmsdocker:installed`
instead of `fmsdocker:prep`) somewhere the developer can find it — see
`references/troubleshooting.md` for the full recreate-from-image snippet.

## Step 6 — Patch Nginx

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

**The working path:** let FMS generate its own key, and only bring your own signed certificate.

```bash
# 1. Install mkcert (once)
brew install mkcert

# 2. Inside the container, have FMS generate its own CSR + key (needs the Admin Console
#    username/password interactively — run from the developer's own terminal):
docker exec -it fms bash
"/opt/FileMaker/FileMaker Server/Database Server/bin/fmsadmin" certificate create localhost --keyfilepass <a passphrase>
# answer y, then the Admin Console username/password when prompted
# this creates serverRequest.pem and serverKey.pem in CStore/ — leave serverKey.pem alone

# 3. Pull the CSR out and sign it with mkcert's CA, injecting your own SAN list
#    (Chrome/Safari require SAN entries — a CN-only cert is rejected regardless of match):
docker cp fms:"/opt/FileMaker/FileMaker Server/CStore/serverRequest.pem" ./serverRequest.pem
CAROOT=$(mkcert -CAROOT)
cat > san.ext <<'EOF'
subjectAltName=DNS:localhost,DNS:fms,IP:127.0.0.1,IP:::1
EOF
openssl x509 -req -in serverRequest.pem \
  -CA "$CAROOT/rootCA.pem" -CAkey "$CAROOT/rootCA-key.pem" -CAcreateserial \
  -out serverSigned.pem -days 730 -extfile san.ext

# 4. Copy the signed cert into a persistent volume path (NOT /tmp — see the note below) and import
docker cp serverSigned.pem "fms:/opt/FileMaker/FileMaker Server/CStore/serverSigned.pem"
docker exec fms chown fmserver:fmsadmin "/opt/FileMaker/FileMaker Server/CStore/serverSigned.pem"
# then, back in the interactive shell from step 2:
"/opt/FileMaker/FileMaker Server/Database Server/bin/fmsadmin" certificate import "/opt/FileMaker/FileMaker Server/CStore/serverSigned.pem" --keyfilepass <same passphrase>
# no --keyfile flag this time — FMS already has its own matching key from step 2
```

Restart the container when prompted, then have the **developer** (not the agent) run
`mkcert -install` in their own terminal — this modifies the macOS system trust store, which is
outside what an agent should do on the user's behalf even though the action itself is low-risk.
Same rule for any `/etc/hosts` edits for a local DNS alias (see the reference file for a subtle
trailing-newline trap there).

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
curl -sk -o /dev/null -w "%{http_code}\n" https://localhost:8443/fmi/webd                          # WebDirect
curl -sk -o /dev/null -w "%{http_code}\n" https://localhost:8443/fmi/odata/v4/                      # OData
curl -sk -o /dev/null -w "%{http_code}\n" https://localhost:8443/fmi/data/vLatest/productInfo       # Data API
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

## Verifying the whole install

```bash
docker exec fms systemctl is-active fmshelper          # expect: active
curl -sk -o /dev/null -w "%{http_code}\n" https://localhost:8443/admin-console/   # expect: 200
echo | openssl s_client -connect localhost:8443 -servername localhost 2>/dev/null | openssl x509 -noout -issuer   # expect: your mkcert CA, not FMS's self-signed default
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
