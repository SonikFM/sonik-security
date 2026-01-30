# Sonik Security (SigNoz) Deployment Runbook

## Overview

Sonik Security is our self-hosted observability stack powered by SigNoz. It runs on the same stateful EC2 host as PostHog (`monitor.sonik.fm`), sharing the `/data` volume and Caddy ingress.

## Architecture

- **Ingress**: Cloudflare Tunnel → Caddy (`:80`) → SigNoz (`:8080`) / OTLP (`:4318`)
- **Storage**: ClickHouse + Zookeeper on `/data` (EBS)
- **Images**: Pinned to `ghcr.io/sonikfm/*` (airgap-safe)
- **AI**: Claude Opus 4.5 via OpenRouter (RCA enabled)

## Deployment Steps

### 1. Preflight Check

Ensure the host is healthy and `/data` has space.

```bash
# SSH into host
ssh ubuntu@100.94.181.123

# Check disk space (should have >20% free on /data)
df -h /data

# Check if Docker root is on /data
docker info | grep "Docker Root Dir"
# Expected: /data/docker
```

### 2. Deploy via GitHub Actions

Run the `Deploy: SigNoz (Echo6)` workflow in `sonik-security` repo.

Inputs:
- `run_deploy`: `true`
- `take_snapshot`: `true` (CRITICAL: Always take a snapshot before upgrading)

### 3. Validation

After deploy completes, verify services are healthy.

#### UI Access
Visit [https://security.sonik.fm](https://security.sonik.fm) and ensure you can log in.

#### Health Check
```bash
curl -I https://security.sonik.fm/api/v1/health
# Expected: HTTP/2 200
```

#### OTLP Ingest
Verify the collector is accepting data (public endpoint).

```bash
curl -X POST https://otel.sonik.fm/v1/traces \
  -H "Content-Type: application/json" \
  -d '{"resourceSpans":[]}'
# Expected: {} (200 OK)
```

#### ClickHouse Connectivity
```bash
# On host
docker compose -f ~/sonik-security/docker-compose.base.yml exec clickhouse clickhouse-client --query "SELECT 1"
# Expected: 1
```

## Rollback Procedure

If a deployment fails or corrupts data:

### 1. Revert Code/Images
- Revert the PR or commit in `sonik-security`.
- Re-run the Deploy workflow with the previous stable version.

### 2. Restore Data (if corrupted)
If ClickHouse data is FUBAR, restore the EBS snapshot taken during preflight.

1. **Stop Services**:
   ```bash
   cd ~/sonik-security && docker compose down
   cd ~/sonik-monitor && docker compose down
   ```

2. **Unmount /data**:
   ```bash
   sudo umount /data
   ```

3. **Restore Snapshot (AWS Console/CLI)**:
   - Create new volume from the snapshot ID (found in GitHub Actions logs).
   - Detach old volume (`/dev/sdf`).
   - Attach new volume to instance as `/dev/sdf`.

4. **Remount & Restart**:
   ```bash
   sudo mount -a
   cd ~/sonik-monitor && docker compose up -d
   cd ~/sonik-security && docker compose up -d
   ```

## Troubleshooting

### "No space left on device"
Check if Docker root is actually on `/data`. If not, run the bootstrap steps from `user-data.sh` manually.

### "Connection refused" on OTLP
Ensure Caddy is running and routing correctly.
```bash
docker logs echo-server  # (or whatever caddy container name is used)
```

### "ClickHouse init failed"
Check if the custom image `ghcr.io/sonikfm/signoz-clickhouse` was built correctly with `histogramQuantile`.
```bash
docker run --rm -it --entrypoint ls ghcr.io/sonikfm/signoz-clickhouse:25.5.6-histq1 -l /var/lib/clickhouse/user_scripts/histogramQuantile
```
