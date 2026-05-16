# Runbook: VPN / Download Pipeline Troubleshooting

## Symptoms
- Downloads stalled or not starting
- qBittorrent showing "connection closed" or no peers
- Arr apps can't find releases
- Gluetun container unhealthy or restarting

## Quick Check (60 seconds)

```bash
# Is Gluetun running and healthy?
docker ps | grep gluetun

# Check Gluetun logs for VPN connection status
docker logs --tail 50 gluetun

# Verify VPN IP (should NOT be your real IP)
docker exec gluetun wget -qO- https://ipinfo.io/ip
```

## Common Causes

### VPN Provider Outage
**Symptom:** Gluetun logs show repeated connection failures.
**Fix:** Check provider status page. Wait, or switch to a different server/region in `.env` and restart: `docker compose restart gluetun`

### WireGuard Key Expired
**Symptom:** Gluetun logs show authentication errors.
**Fix:** Generate new keys from VPN provider dashboard, update `WIREGUARD_PRIVATE_KEY` and `WIREGUARD_PRESHARED_KEY` in `.env`, restart Gluetun.

### Container Dependency Chain Broken
**Symptom:** qBittorrent or arr apps can't reach the internet, but Gluetun is healthy.
**Fix:** Restart the full download chain:
```bash
docker compose restart gluetun qbittorrent radarr sonarr lidarr jackett
```

### Port Conflict After Compose Change
**Symptom:** Container won't start, port already in use.
**Fix:** Check what's using the port:
```bash
sudo ss -tlnp | grep :PORT_NUMBER
```

## Prevention
- Snapshot ZFS before any compose changes
- Commit compose changes to git before and after
- Test VPN connectivity after every Gluetun restart
