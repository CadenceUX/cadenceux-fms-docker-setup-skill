# Upgrading FileMaker Server in place

The upgrade runbook for an install built with SKILL.md (Steps 1–5).

## Contents
- Upgrading FileMaker Server in place
- U1 — Get the new package and compare it with the old one
- U2 — Snapshot the container's software
- U3 — Back up the four data volumes (a commit doesn't include them)
- U4 — Copy the package in and dry-run it
- U5 — Run the upgrade (interactive — developer's terminal)
- U6 — Verify
- U7 — Clean up, then commit
- One follow-on decision when upgrading from 26.0.2 or earlier to 26.0.3+ (Nginx)
- Rolling back


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

