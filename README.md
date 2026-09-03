# SiYuan — Docker Compose

[![Deployment Verification](https://github.com/heyvaldemar/siyuan-docker-compose/actions/workflows/deployment-verification.yml/badge.svg?branch=main)](https://github.com/heyvaldemar/siyuan-docker-compose/actions/workflows/deployment-verification.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

This repository deploys **SiYuan** — a privacy-first personal knowledge base with block-level editing — with its workspace on the host filesystem and the UI protected by an access code.

## Getting started

```bash
# 1. Clone
git clone https://github.com/heyvaldemar/siyuan-docker-compose
cd siyuan-docker-compose

# 2. Create the Docker network the stack expects
docker network create siyuan-network

# 3. Copy the environment template and set the access code
cp .env.example .env
$EDITOR .env

# 4. Deploy
docker compose -f siyuan-docker-compose.yml -p siyuan up -d
```

Open `http://YOUR_SERVER:6806` and enter the access code from `.env`. Notes live in `./workspace` next to the compose file.

### What success looks like

```bash
docker compose -f siyuan-docker-compose.yml -p siyuan ps
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:6806/   # 401 until you enter the code — that's the kernel up and auth enforced
```

### Common first-deploy issues

- **Container restarts and the log prints the kernel help text.** You are on a pre-v1.0.0 revision of this compose: SiYuan v3.8 changed its CLI, `git pull` fixes it.
- **`docker compose up` fails with `set in .env`.** `SIYUAN_AUTH_CODE` is empty; the error names it.
- **Network not found.** Step 2 was skipped.

## Supply chain trust

The image [`b3log/siyuan`](https://hub.docker.com/r/b3log/siyuan) is pinned to `tag@sha256:<digest>` as an interpolation default in the compose `x-images` block. `git pull` alone delivers the tested version; `SIYUAN_IMAGE_TAG` in `.env` overrides deliberately.

The daily `check-pin-freshness` CI job re-resolves the pin and compares it against the latest SiYuan release. GitHub Actions are pinned by commit SHA; Dependabot keeps those fresh.

## Production checklist

- [ ] **Strong access code** — it is the only thing between the internet and your notes if you expose the port; prefer keeping 6806 on a private network or behind a VPN.
- [ ] **Back up `./workspace`** — that directory is your entire knowledge base.
- [ ] **Update deliberately** — bump the pin after reading SiYuan release notes; the kernel migrates workspace data on version jumps.

## Unattended updates

Releases are the update channel: a tag is cut only after CI has built the pinned images, booted the full stack, and passed the smoke tests. `update.sh` moves a deployment to the newest tag and nothing else:

```bash
./update.sh --dry-run   # show what would be applied
./update.sh             # update within the current major and redeploy
```

Put it on a timer for hands-off minor/patch updates:

```bash
# crontab -e
17 5 * * *  /opt/siyuan-docker-compose/update.sh >> /var/log/siyuan-update.log 2>&1
```

The script refuses to cross a MAJOR template version on its own — majors are breaking by definition and their release notes exist to be read. After reading them, `./update.sh --allow-major` performs the jump. It also refuses to touch a checkout with local modifications: your customization belongs in `.env`, which updates never overwrite.

This is deliberately a host-side script and not a container in the stack: an in-stack updater needs the Docker socket (root on the host) and turns "someone pushed to a repo" into "someone deployed to your machine" with no operator in the loop. A cron job under your own user updates only to tagged, CI-verified states and leaves the trust boundary where it was.

## Resource limits

Every service carries memory and CPU limits plus reservations as compose-level defaults — the same values CI boots the stack under. Override any of them in `.env` (the knobs and their defaults are listed in `.env.example`, e.g. `TRAEFIK_MEMORY_LIMIT=512m`) and the override survives every `git pull`. If a service is OOM-killed under real load, `docker inspect <container> --format '{{.State.OOMKilled}}'` says so; raise its `_MEMORY_LIMIT` and recreate.

## Backups

The `backups` container runs on a loop: an initial delay (`SIYUAN_BACKUP_INIT_SLEEP`, default 30m), then every `SIYUAN_BACKUP_INTERVAL` (default 24h) it takes a `tar.gz` of the rest of the data directory (live database files excluded), into the `siyuan-backups` volume; files older than `SIYUAN_BACKUP_PRUNE_DAYS` (default 7) are pruned. Each artefact logs `... backup OK: <file> (<bytes> bytes)` or `FAILED` (kept as `<file>.failed`) — grep the log for `FAILED` from your monitoring.

**Verify backups are running:**

```bash
docker compose -p siyuan logs backups | tail -5
docker compose -p siyuan exec backups ls -la /srv/siyuan/backups/
```

**Restore** a backup set with the interactive script (`chmod +x siyuan-restore-data.sh` once): it stops siyuan, unpacks the data archive over the data directory, and starts siyuan again.

```bash
./siyuan-restore-data.sh
```

**Off-host replication.** Backups live in a named volume on the same host — bind-mount `SIYUAN_BACKUPS_PATH` to a directory covered by your off-host backup solution (restic, rclone, Borg, S3 sync).

## Testing

The [Deployment Verification](https://github.com/heyvaldemar/siyuan-docker-compose/actions/workflows/deployment-verification.yml?query=branch%3Amain) workflow runs on every push, pull request, and every day at 06:00 UTC: actionlint, a Trivy scan of the pinned image, the weekly freshness check, and a deploy-and-test job that boots the kernel with an ephemeral access code and requires it to answer with auth enforced.

### Backup and restore, proven

`tests/e2e-backup-restore.sh` runs against the live stack and is what CI executes after the smoke test. The scenario that matters most is the restore roundtrip: the application is stopped, the baseline archive is unpacked over the data directory, and a file created after the baseline is gone. The tests stop the application briefly and write into its data directory — run them on a staging copy with short intervals in `.env` (`SIYUAN_BACKUP_INIT_SLEEP=15s`, `SIYUAN_BACKUP_INTERVAL=60s`), never on production.

```bash
chmod +x tests/e2e-backup-restore.sh
./tests/e2e-backup-restore.sh
```

---

## About the maintainer

<div align="center">

**Maintained by [Vladimir Mikhalev](https://github.com/heyvaldemar)** — Docker Captain · IBM Champion · AWS Community Builder

[YouTube](https://www.youtube.com/channel/UCf85kQ0u1sYTTTyKVpxrlyQ?sub_confirmation=1) · [Blog](https://heyvaldemar.com) · [LinkedIn](https://www.linkedin.com/in/heyvaldemar/)

</div>
