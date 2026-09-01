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

The weekly `check-pin-freshness` CI job re-resolves the pin and compares it against the latest SiYuan release. GitHub Actions are pinned by commit SHA; Dependabot keeps those fresh.

## Production checklist

- [ ] **Strong access code** — it is the only thing between the internet and your notes if you expose the port; prefer keeping 6806 on a private network or behind a VPN.
- [ ] **Back up `./workspace`** — that directory is your entire knowledge base.
- [ ] **Update deliberately** — bump the pin after reading SiYuan release notes; the kernel migrates workspace data on version jumps.

## Testing

The [Deployment Verification](https://github.com/heyvaldemar/siyuan-docker-compose/actions/workflows/deployment-verification.yml?query=branch%3Amain) workflow runs on every push, pull request, and every Monday at 06:00 UTC: actionlint, a Trivy scan of the pinned image, the weekly freshness check, and a deploy-and-test job that boots the kernel with an ephemeral access code and requires it to answer with auth enforced.

---

## About the maintainer

<div align="center">

**Maintained by [Vladimir Mikhalev](https://github.com/heyvaldemar)** — Docker Captain · IBM Champion · AWS Community Builder

[YouTube](https://www.youtube.com/channel/UCf85kQ0u1sYTTTyKVpxrlyQ?sub_confirmation=1) · [Blog](https://heyvaldemar.com) · [LinkedIn](https://www.linkedin.com/in/heyvaldemar/)

</div>
