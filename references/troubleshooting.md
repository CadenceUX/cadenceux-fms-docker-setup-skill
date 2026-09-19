# Troubleshooting reference — FileMaker Server on Docker

Organized by symptom. Each section explains *why* it happens, not just the fix — useful when
the exact wording of an error differs slightly from what's quoted here.

## "FileMaker Server disappeared" / commands say the binary doesn't exist

**Symptom:** `fmshelper`/`fmsadmin` missing, `dpkg -l | grep filemaker` shows nothing, but the
container is running and the FMS Admin Console won't load.

**Cause:** FileMaker Server was installed via `apt install *.deb` — this writes into the
*container's own writable layer*, not the Docker image and not any volume. If the container was
ever `docker rm`'d and recreated from the base `fmsdocker:prep` image (rather than from a
`docker commit`'d image with FMS already installed), the software is gone. The four named data
volumes (`fms-data`, `fms-cstore`, `fms-wpeconf`, `fms-logs`) survive this regardless, since
they're mounted, not baked in — so licenses, config, and databases are typically all still
intact even when the software itself is gone. Confirm with:

```bash
docker exec <container> bash -c "ls '/opt/FileMaker/FileMaker Server/CStore/' 2>&1"
```

If that directory has content (license file, keys, certs), the data is fine — only the
application needs reinstalling.

**Fix — reinstall on top of the existing data:**

```bash
docker cp /path/to/filemaker-server-*.deb <container>:/tmp/
docker exec <container> bash -c "apt-get update && apt install /tmp/filemaker-server-*.deb"
# choose "load previous configuration" when the wizard asks — this reuses the existing
# Data/CStore contents (admin account, license, sample DB staging) rather than starting fresh
```

Re-apply the Nginx patch too (Step 6 in SKILL.md) — that was also lost, since it's the same
writable-layer issue.

**Prevention (do this from the start, not just after a loss):**

```bash
docker commit --message "FMS installed - $(date '+%Y-%m-%d %H:%M')" <container> fmsdocker:installed
```

Re-run after every apt-level change. Recreate the container from `fmsdocker:installed` (not
`fmsdocker:prep`) going forward:

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
  fmsdocker:installed
```

## Admin Console reports a wildly wrong disk-usage percentage (e.g. 89-90% when volumes are barely used)

**Cause:** if a host installer folder was bind-mounted into the container (e.g.
`-v /path/on/mac:/install`) so `apt` could see the `.deb`, FMS's disk-usage health check scans
*all* mounts — including that bind mount — and can end up reporting the Mac's actual boot-disk
usage percentage instead of the container/FMS volumes' real usage, since the bind mount passes
through the host filesystem's own stats.

**Fix:** don't leave that bind mount attached long-term. Once the installer step is done,
either use `docker cp` for any further one-off file transfers, or recreate the container without
the `/install` mount (same `docker run` command as above, just omit that `--volume` line).

**Verify the real number** rather than trusting Admin Console's display:

```bash
docker exec <container> df -h "/opt/FileMaker/FileMaker Server/Data" / /install 2>&1
```

Whichever mount shows the inflated percentage is the bind mount to remove.

## "Cannot decrypt the private key file... with the password" (error 20408) during certificate import

**Cause:** `fmsadmin certificate import <cert> --keyfile <externally-generated-key>
--keyfilepass <pass>` has a real bug (or at minimum an unsupported path) specific to importing
an *externally-generated* private key. This was tested exhaustively — unencrypted key, PKCS#8
with AES, legacy PKCS#1 with 3DES, legacy PKCS#1 with AES-128 — every variant fails identically,
even when the exact same key/password pair verifies as completely valid via plain OpenSSL
(`openssl rsa -in <key> -passin pass:<pass> -check`) on both the Mac and inside the container.
Don't spend time iterating on cipher/format — it isn't a format problem.

**Fix — use FMS's own key instead of an external one:**

```bash
# 1. Let FMS generate its own CSR + key (interactive — run from a real terminal):
docker exec -it <container> bash
"/opt/FileMaker/FileMaker Server/Database Server/bin/fmsadmin" certificate create localhost --keyfilepass <passphrase>
# creates CStore/serverRequest.pem (CSR) and CStore/serverKey.pem

# 2. Pull the CSR out, sign it with your own CA (mkcert or otherwise), injecting SAN entries:
docker cp <container>:"/opt/FileMaker/FileMaker Server/CStore/serverRequest.pem" ./serverRequest.pem
CAROOT=$(mkcert -CAROOT)
printf 'subjectAltName=DNS:localhost,DNS:<other-names>,IP:127.0.0.1,IP:::1\n' > san.ext
openssl x509 -req -in serverRequest.pem -CA "$CAROOT/rootCA.pem" -CAkey "$CAROOT/rootCA-key.pem" \
  -CAcreateserial -out serverSigned.pem -days 730 -extfile san.ext

# 3. Import WITHOUT --keyfile — FMS already has the matching key from step 1:
docker cp serverSigned.pem "<container>:/opt/FileMaker/FileMaker Server/CStore/serverSigned.pem"
docker exec <container> chown fmserver:fmsadmin "/opt/FileMaker/FileMaker Server/CStore/serverSigned.pem"
"/opt/FileMaker/FileMaker Server/Database Server/bin/fmsadmin" certificate import "/opt/FileMaker/FileMaker Server/CStore/serverSigned.pem" --keyfilepass <same passphrase>
```

Restart the container when it asks. This path worked cleanly on the first attempt in testing —
if it still fails, something else is wrong (check the CSR and cert actually match with
`openssl x509 -noout -pubkey` vs `openssl rsa -pubout` on both files).

**To add more SAN names later:** re-sign the same CSR file (still sitting wherever you pulled it
to) with an expanded `subjectAltName` line and re-import with the same passphrase — no need to
redo `certificate create`.

**Two related rules, not bugs:**
- Chrome and Safari both reject a certificate with only a CN match and no SAN entries — always
  include SAN, even for `localhost`.
- `mkcert -install` and any `/etc/hosts` edit must be run by the developer themselves, in their
  own terminal — these modify macOS system/trust settings, which shouldn't be automated on the
  user's behalf even though the actions themselves are low-risk.

## Local DNS alias (`/etc/hosts`) silently doesn't resolve

**Symptom:** added a line like `127.0.0.1 fms-linux` to `/etc/hosts`, but the name still won't
resolve (`ping: cannot resolve`).

**Cause:** if the file's last existing line has no trailing newline, `echo "..." | sudo tee -a
/etc/hosts` appends directly onto the end of that line instead of starting a new one — e.g. a
pre-existing `## Local - End ##` marker (common with VPN clients that manage a hosts-file block)
becomes `## Local - End ##127.0.0.1 fms-linux`, which is not a valid entry.

**Verify:** `grep <name> /etc/hosts` — if the match is on the same line as unrelated text,
that's the problem.

**Fix:** don't risk a sed one-liner against unknown surrounding content — have the developer
open `sudo nano /etc/hosts`, position the cursor right before the IP address, press Enter to
split it onto its own line, then save (Ctrl+O, Enter, Ctrl+X).

## `/tmp` contents vanished inside the container

**Cause:** an *unclean* restart (Docker Desktop itself quit or killed — container exit code 137,
not an OOM kill) causes the container's systemd to treat the next start as a fresh boot, which
wipes `/tmp`. A clean `docker restart` does not have this problem.

**Fix:** don't rely on `/tmp` for anything that needs to survive a restart — put certificate
files, install artifacts, etc. in one of the persistent volume paths instead
(`CStore/`, `Data/`, or similar) even if it feels like a less "throwaway" location.

## Container is stopped/exited after Docker Desktop was quit or the Mac slept/restarted

**Not data loss.** Quitting Docker Desktop stops its whole Linux VM (and everything in it), but
containers and volumes on disk are untouched.

```bash
docker ps -a   # confirm it shows "Exited (137)", not removed entirely
docker start <container>
sleep 15
docker exec <container> systemctl is-active fmshelper   # expect: active
```

**Also check for a stray empty auto-named container** — Docker Desktop occasionally spins one
up on its own restart. Safe to identify and remove if it has zero volume mounts:

```bash
docker inspect <stray-name> --format 'Mounts={{.Mounts}}'   # empty [] = safe to remove
docker rm <stray-name>
```

## Nginx patch: `sudo` commands silently fail / do nothing

**Cause:** this minimal container has no `sudo` package installed — you're already root via
`docker exec`, so Claris's own `NginxUpdate.sh` (and other vendor scripts that assume `sudo`
exists) fail every `sudo`-prefixed line without necessarily erroring loudly.

**Fix — either works:**
- Strip `sudo` from each command (root doesn't need it), or
- `apt-get install -y sudo` first, then run the vendor script completely unmodified.

**After any Nginx upgrade,** restart the whole container — not `systemctl restart nginx`
inside it. FMS spawns its own Nginx process directly (via `fmshelper`, using
`/opt/FileMaker/FileMaker Server/NginxServer/conf/fms_nginx.conf`), already bound to 80/443. A
plain `systemctl restart nginx` targets a *different*, disabled, generic `nginx.service` unit
that conflicts on those same ports and fails to bind — that failure is expected and not a sign
anything is actually broken. A full `docker restart <container>` makes `fmshelper` respawn its
own Nginx child against the newly-installed binary correctly.

## `command not found` / `No such file` for a Linux command during an interactive step

**Symptom:** `apt-get: command not found`, `systemctl: command not found`, or `No such file or
directory` on a path under `/opt/FileMaker/...` — during Step 4, 8 or 9.

**Cause:** almost always the command was typed into the developer's **macOS host shell** instead
of inside the container. Those commands and paths only exist in the container, so the error
reads as a generic failure rather than an obvious "wrong shell" signal. Seen in practice when an
agent opened a terminal tab already `docker exec -it`'d into the container (prompt
`root@fms:/#`) but the developer typed into a different, pre-existing tab (prompt like
`hostname:~ user$`).

**Fix:** check the prompt string before debugging anything else. If it isn't `root@fms:/#`, run
`docker exec -it fms bash` and retry there. When handing off an interactive step, name the exact
tab and prompt to use rather than saying "run this in your terminal."

## Sample database not showing up

See SKILL.md Step 7 for the fix. Root cause: the assisted installer stages locale-named sample
database files (`en_FMServer_Sample`, no extension) in a hidden `Data/Databases/.Sample/`
folder, and the final "promote to a live, openable file" step doesn't always run — most
reliably skipped when installing on top of pre-existing `Data` rather than a truly fresh
install. Confirm the fix worked via `Event.log`, not just file presence — `ls`-ing the file
existing doesn't confirm FMS actually opened it.

**A different case — the sample database is absent entirely**, not just unpromoted: no
`.Sample/` staging file, no `.fmp12` anywhere (`find "/opt/FileMaker/FileMaker Server" -iname
'*.fmp12'` returns nothing), and `Data/Databases/Sample/` is an empty directory. Seen once on FMS
26.0.2.219 arm64. The root cause is **not established** — it may be how the wizard's "remove
sample database" prompt was answered, or a version-specific default for this package. It isn't
necessarily a broken install, and there is nothing to promote. Don't assert either explanation;
confirming needs a run that deliberately answers "keep sample" and checks immediately.

## WebDirect / `/fmi/webd` returns 502 Bad Gateway

**Cause:** every publishing component (Web Publishing Engine/WebDirect, Custom Web Publishing,
the Data API, OData) is **disabled by default**, even after a completely successful
assisted-install. Nginx proxies the request correctly, but nothing is listening on the other
end, hence the 502 — this is not a broken install, it's an unconfigured feature.

**Confirm the actual state** rather than assuming from how the install wizard went. Match on
the whole `<component>` tag, not just `name="wpe"` — the `enabled` attribute comes *before*
`name` in the XML, so a naive grep on `name="wpe"` alone won't actually show you whether it's
enabled:

```bash
docker exec <container> bash -c "cat '/opt/FileMaker/FileMaker Server/Admin/conf/deployment.xml'" | grep -oE '<component[^>]*name=\"(wpe|odata|fmdapi|xdbc)\"[^>]*>'
```

**Fix:** enable it via Admin Console (a Configuration toggle) or the CLI:

```bash
docker exec -it <container> bash
"/opt/FileMaker/FileMaker Server/Database Server/bin/fmsadmin" enable wpe
```

**Startup delay is real but genuinely variable — confirmed across two separate restarts, not
just theorized.** First restart: OData and the Data API didn't come up for 34-45 seconds after
the database engine started. Second restart, same container, same `enabled="yes"` config
throughout: both were up within ~2-3 seconds. There's no fixed number to wait — poll with
retries over roughly a minute rather than checking once and concluding either "it's broken" or
"it's fine":

```bash
for i in 1 2 3 4 5 6; do
  sleep 10
  curl -sk -o /dev/null -w "%{http_code}\n" https://localhost:<port>/fmi/webd
done
```

**This cuts both ways.** If a developer manually starts a component after finding it
apparently "off," that is *not* evidence the setting actually reverted — check
`deployment.xml`'s `enabled` attribute first. If it already says `yes`, the config never
changed; what looked like a dropped setting was almost certainly this startup-timing
variability caught at the wrong moment, not a real regression. This was confirmed directly: a
deliberate restart with `enabled="yes"` unchanged going in came back up within seconds on that
attempt, cleanly ruling out a persistence bug.

Same disabled-by-default pattern applies to `xdbc` (ODBC/JDBC), `fmdapi` (Data API), and
`odata` in the same `deployment.xml` — check and enable whichever the developer actually needs.
