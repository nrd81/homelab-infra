# Runbook: Container Recovery

## Symptoms
- Service unreachable (connection refused, timeout)
- Container in restart loop
- Container exited unexpectedly
- OOM (Out of Memory) kill

## Quick Check (60 seconds)

```bash
# What's running, what's not?
docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Check logs for the broken container
docker logs --tail 100 CONTAINER_NAME

# Check system resources
free -h
df -h
```

## Decision Tree

### Container is "Exited"
```bash
# Check exit code
docker inspect CONTAINER_NAME --format='{{.State.ExitCode}}'
```
- **Exit 0:** Stopped normally. Just restart: `docker compose up -d CONTAINER_NAME`
- **Exit 1:** Application error. Check logs.
- **Exit 137:** OOM killed. Container needs more memory or the host is under pressure.
- **Exit 139:** Segfault. Usually a bad image. Try pulling a fresh one: `docker compose pull CONTAINER_NAME && docker compose up -d CONTAINER_NAME`

### Container is "Restarting"
Something is crashing immediately on startup. Check logs:
```bash
docker logs --tail 50 CONTAINER_NAME
```
Common causes: missing config file, bad environment variable, port conflict, permission denied on volume mount.

### OOM Kill
```bash
# Confirm OOM
dmesg | grep -i "oom\|killed" | tail -20

# Check what's using memory
docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}"
```
If one container is eating memory, set a limit in compose:
```yaml
deploy:
  resources:
    limits:
      memory: 2G
```

### Permission Denied on Volume
```bash
# Check ownership
ls -la /path/to/volume

# Fix ownership to match PUID/PGID
sudo chown -R 1000:1000 /path/to/volume
```

## Nuclear Option: Full Stack Restart
If multiple containers are broken and you can't isolate the cause:
```bash
cd ~/docker-apps
docker compose down
docker compose up -d
```

## After Recovery
1. Verify all services are healthy: `docker ps`
2. Document what broke and why in a new runbook entry
3. If you changed anything, commit to git:
```bash
cd ~/homelab-infra
cp ~/docker-apps/docker-compose.yml .
git add -A
git commit -m "Describe what changed"
git push
```
