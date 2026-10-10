# After the install: Steps 6–9

Optional and version-dependent steps that follow Step 5 in SKILL.md. Commit the container
(Step 5) after every apt-level change made here.

## Contents
- Step 6 — Patch Nginx (26.0.2 and earlier — check first on 26.0.3+)
- Step 7 — Promote the sample database (if missing)
- Step 8 — Set up a trusted local HTTPS certificate
- Step 9 — Enable WebDirect / Web Publishing (optional, but off by default)

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

