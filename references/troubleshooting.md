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

**First, rule out the wrong shell.** `fmsadmin: command not found` is also what you get when
the command was typed into the macOS host shell instead of the container — see "`command not
found` / `No such file`" below. Check the prompt before assuming the software is gone. Inside the
container, confirm with `dpkg -l filemaker-server`, and call `fmsadmin` by full path
(`"/opt/FileMaker/FileMaker Server/Database Server/bin/fmsadmin"`) rather than relying on `PATH`.

**Fix 1 — recreate from a committed image, if one exists (no reinstall needed):**

```bash
docker images fmsdocker     # look for :installed, a :<build> tag, or a :pre-upgrade-* tag
```

If there is one, the software is already saved — recreate the container from it with the
disaster-recovery `docker run` below, attached to the same four volumes. Confirm with the
developer before removing the current container, and pick the tag that matches the FMS build the
data was last used with (newest `:installed` unless the developer says otherwise).

**Fix 2 — reinstall on top of the existing data (only when no committed image exists):**

1. **Back up the four volumes first**, exactly as in SKILL.md's U3 (stop the container, tar each
   volume with a throwaway `fmsdocker:prep` container, start it again). A reinstall that goes
   wrong over the only copy of the licence and databases is the worst version of this problem.
2. **Reinstall the same build the data was last used with**, not simply the newest zip on disk.
   Installing an older build over data from a newer one isn't a tested path. Ask the developer
   which build it was running; if they aren't sure, the log volume survived and records it on
   every start:
   `docker exec <container> bash -c "grep 'Starting Database Server' '/opt/FileMaker/FileMaker Server/Logs/Event.log' | tail -1"`
   gives e.g. `Starting Database Server 26.0.3 309(08-26-2026)...` — build 26.0.3.309 (verified
   2026-09-24). This needs a running container but not a working FMS install. With no container
   at all, read the log volume through a throwaway one:
   `docker run --rm -v fms-logs:/logs:ro fmsdocker:prep grep 'Starting Database Server' /logs/Event.log | tail -1`. To move to a newer build, reinstall the old one first, then follow SKILL.md's upgrade
   section.
3. Copy the package in (agent can do this) and hand the interactive install to the developer —
   the wizard needs a real TTY, so it must be `docker exec -it`, run in the developer's own named
   terminal tab (prompt `root@<hostname>:/#`):

   ```bash
   docker cp /path/to/filemaker-server-<build>-arm64.deb <container>:/tmp/
   # developer's terminal:
   docker exec -it <container> bash
   apt-get update && apt install /tmp/filemaker-server-<build>-arm64.deb
   ```

   Choose "load previous configuration" when the wizard asks. That reuses the existing
   Data/CStore contents (admin account, licence, sample DB staging) instead of starting fresh.
4. **Re-apply what lived in the writable layer**, which was lost along with the software. On
   26.0.2 or earlier, that includes the Nginx patch (SKILL.md Step 6); on 26.0.3+ read Step 6
   first, since Claris no longer wants it. Publishing settings in
   `Admin/conf/deployment.xml` aren't in any of the four volumes either — re-check WebDirect,
   Data API, OData and ODBC (SKILL.md Step 9) rather than assuming they came back.
5. Commit straight away (SKILL.md Step 5 — delete the `.deb` from `/tmp` and `apt-get clean`
   first).

**Prevention (do this from the start, not just after a loss):**

```bash
docker exec <container> bash -c "rm -f /tmp/*.deb && apt-get clean"
docker commit --message "FMS installed - $(date '+%Y-%m-%d %H:%M')" <container> fmsdocker:installed
```

Re-run after every apt-level change. Recreate the container from `fmsdocker:installed` (not
`fmsdocker:prep`) going forward. **Copy the current container's real settings rather than
trusting the example values below** — read them before removing anything:

```bash
docker inspect <container> --format 'name={{.Name}} host={{.Config.Hostname}} mem={{.HostConfig.Memory}} cpus={{.HostConfig.NanoCpus}} ports={{json .HostConfig.PortBindings}}'
docker inspect <container> --format '{{range .Mounts}}{{.Name}} -> {{.Destination}}{{"\n"}}{{end}}'
```

(`Memory` is in bytes and `NanoCpus` is CPUs × 10⁹; `0` means no limit was set.)

**If the container is already gone**, there's nothing to inspect. Ask the developer for the
ports, memory and CPUs they used, or look for them where they may have been recorded: a saved
`docker run` command, shell history (`grep 'docker run' ~/.zsh_history`, with their OK — it's
their history), or Docker Desktop's container list if it still shows the removed one. The
volume names are recoverable from `docker volume ls --filter name=fms-`. Say which values were
confirmed and which were assumed before running anything.

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
  fmsdocker:installed
```

## Admin Console reports a wildly wrong disk-usage percentage (e.g. 89-90% when volumes are barely used)

**Cause:** if a host installer folder was bind-mounted into the container (e.g.
`-v /path/on/mac:/install`) so `apt` could see the `.deb`, FMS's disk-usage health check scans
*all* mounts — including that bind mount — and can end up reporting the Mac's actual boot-disk
usage percentage instead of the container/FMS volumes' real usage, since the bind mount passes
through the host filesystem's own stats.

**Fix:** don't leave that bind mount attached long-term — use `docker cp` for one-off file
transfers instead. Removing a mount means recreating the container:

1. Find the bind mount: `docker inspect <container> --format '{{range .Mounts}}{{.Type}} {{.Source}} -> {{.Destination}}{{"\n"}}{{end}}'`
   — the `bind` line is the one to drop. If there's **no** `bind` line, this isn't the cause; say
   so rather than recreating anything, and check `docker system df` and Docker Desktop's disk
   limit instead.
2. Commit first (SKILL.md Step 5, including the `.deb` clean-up) so the software is safe, and
   confirm with the developer — this removes the container.
3. Record its settings with the `docker inspect` commands under "FileMaker Server disappeared",
   then `docker stop -t 135 <container> && docker rm <container>`.
4. Recreate with the disaster-recovery `docker run` from that section — it has only the four named
   volumes, so the bind mount is simply not carried over.

**Verify the real number** rather than trusting Admin Console's display:

```bash
docker exec <container> df -h "/opt/FileMaker/FileMaker Server/Data" / "<bind-mount destination>" 2>&1
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

# 2. Pull the CSR out, sign it with your own CA (mkcert or otherwise), injecting SAN entries.
#    mkcert's CA files only exist once `mkcert -install` has run (developer, Mac terminal):
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

## Admin Console loads in one browser but `http://localhost/admin-console` gives a 404 in another

**Symptom:** the Admin Console works in one place (an agent's browser pane, a bookmark) but the
developer's own browser shows a 404 or a plain "It works!" page at
`http://localhost/admin-console/signin`. Seen 2026-09-24.

**Cause:** the URL is `http://`, i.e. port 80 on the Mac. On macOS that's often the built-in
Apache, not the container — which is why the container publishes `8080:80` in the first place.
Confirm: `curl -sI http://localhost/ | grep -i '^server'` shows `Apache/2.4.x (Unix)`.

**Fix:** use `https://localhost[:port]/admin-console/` (see SKILL.md's conventions for `[:port]`). If the
developer wants plain `http://localhost` to stop landing on Apache, `sudo apachectl stop` does
it — a system change, so it's their call and their terminal, not the agent's.

## Committed image is hundreds of MB bigger than expected

**Cause:** the installer `.deb` was still in the container's `/tmp` (or apt's cache was full)
when `docker commit` ran — the commit captures everything in the writable layer. Seen on the
26.0.3 upgrade: 6.6GB image vs 6.06GB once redone.

**Fix:** `docker exec fms bash -c "rm -f /tmp/*.deb && apt-get clean"`, then commit again to the
same tags. The new commit is smaller because a commit is always the container's full diff from
its base image, not a layer on top of the previous commit. Remove the now-untagged oversized
image with `docker rmi <image id>` once nothing uses it.

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
