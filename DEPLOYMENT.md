# VM Deployment Guide

Step-by-step guide for deploying the monitoring stack on a fresh VM.

## Prerequisites

- Docker and Docker Compose installed
- Git installed
- Network access to monitored targets
- Ports 3000, 8065, 9091, 9094, 9115 available

## Deployment Steps

### Step 1: Clone Repository
```bash
git clone <your-repo-url>
cd monitoring
```

### Step 2: Create Alertmanager Configuration
```bash
cp alertmanager/alertmanager.yml.template alertmanager/alertmanager.yml
```

### Step 3: Start All Services
```bash
docker compose up -d
```

Wait ~30 seconds for all services to initialize.

### Step 4: Verify Services Are Running
```bash
docker compose ps
```

All services should show status "Up".

### Step 5: Create Mattermost Webhook

1. Open browser: `http://YOUR_VM_IP:8065`
2. Create first admin account (remember credentials!)
3. Create team (e.g., "Monitoring")
4. Create channel called `alerts`
5. Main Menu → Integrations → Incoming Webhooks
6. Click "Add Incoming Webhook"
   - **Title**: Prometheus Alerts
   - **Channel**: alerts
   - Click **Save**
7. **Copy the webhook URL** (format: `http://mattermost:8065/hooks/XXXXXXXXXXXXX`)

### Step 6: Configure Alertmanager with Webhook

```bash
# Edit Alertmanager config
nano alertmanager/alertmanager.yml
```

Find this line:
```yaml
api_url: 'http://mattermost:8065/hooks/CHANGE_ME'
```

Replace `CHANGE_ME` with your actual webhook ID from Step 5.

Save and exit (Ctrl+X, Y, Enter).

### Step 7: Restart Alertmanager
```bash
docker restart alertmanager
```

### Step 8: Enable Mattermost Username Overrides

1. In Mattermost: Profile Picture → **System Console**
2. Navigate: **Integrations** → **Integration Management**
3. Enable these settings:
   - ✅ Enable integrations to override usernames
   - ✅ Enable integrations to override profile picture icons
4. Click **Save**

### Step 9: Test Alert System
```bash
curl -H "Content-Type: application/json" -d '[{
  "labels": {"alertname":"DeploymentTest","severity":"info"},
  "annotations": {"summary":"VM Deployment Complete","description":"Alert system is working!"}
}]' http://localhost:9094/api/v2/alerts
```

Check Mattermost `#alerts` channel - you should see message from "Prometheus Monitor".

### Step 10: Access Services

Open in browser:
- **Grafana**: http://YOUR_VM_IP:3000
  - Username: `admin`
  - Password: `admin` (change on first login)
- **Prometheus**: http://YOUR_VM_IP:9091
- **Alertmanager**: http://YOUR_VM_IP:9094
- **Mattermost**: http://YOUR_VM_IP:8065

### Step 11: Verify Monitoring Targets

```bash
curl -s http://localhost:9091/api/v1/targets | jq -r '.data.activeTargets[] | "\(.labels.job): \(.labels.instance) - \(.health)"'
```

All targets should show status "up".

## Testing Alert Functionality

To demonstrate alerting during a presentation:

```bash
# Stop Grafana (triggers ProbeDown alert after 2 minutes)
docker stop grafana

# Wait 2-3 minutes
# Alert will appear in Mattermost

# Restart Grafana (triggers "Resolved" notification)
docker start grafana
```

## Updating Configuration

### Update Monitored Targets
```bash
# Edit Prometheus config
nano prometheus/prometheus.yml

# Reload Prometheus
docker restart prometheus
```

### Update Alert Rules
```bash
# Edit alert rules
nano prometheus/alerts.yml

# Reload Prometheus
docker restart prometheus
```

### Change Alert Thresholds
Edit values in `prometheus/alerts.yml`:
- `for: 2m` - Time before alert fires
- `probe_success == 0` - Condition to trigger

## Common Issues

### Alerts Not Appearing in Mattermost

**Check 1**: Verify webhook URL
```bash
grep api_url alertmanager/alertmanager.yml
# Should show your actual webhook ID, not CHANGE_ME
```

**Check 2**: Check Alertmanager logs
```bash
docker logs alertmanager --tail 50
# Look for "status code 200" (success) or errors
```

**Check 3**: Verify Mattermost overrides enabled
- System Console → Integrations
- Both override settings must be ON

**Fix**: If still not working, recreate webhook:
1. Delete old webhook in Mattermost
2. Create new webhook
3. Update `alertmanager/alertmanager.yml` with new URL
4. `docker restart alertmanager`

### Grafana Shows No Data

**Check 1**: Verify Prometheus is running
```bash
curl http://localhost:9091/-/healthy
# Should return "Prometheus Server is Healthy."
```

**Check 2**: Check datasource
- Grafana → Configuration → Data Sources
- Should have "prometheus" datasource
- URL should be `http://prometheus:9090`

**Fix**: Restart Grafana
```bash
docker restart grafana
```

### Targets Showing as Down

**Check 1**: Verify Blackbox Exporter
```bash
curl 'http://localhost:9115/probe?target=google.com&module=http_2xx'
# Should return metrics
```

**Check 2**: Check network connectivity
```bash
docker exec prometheus ping -c 2 blackbox
docker exec prometheus ping -c 2 google.com
```

**Fix**: Restart affected services
```bash
docker restart blackbox prometheus
```

### Container Keeps Restarting

**Check logs**:
```bash
docker logs <container-name> --tail 100
```

**Common causes**:
- Port already in use (check with `netstat -tulpn`)
- Configuration file syntax error
- Insufficient memory

**Fix**: Check Docker resources
```bash
docker stats
# Ensure sufficient CPU and memory available
```

## Maintenance Commands

```bash
# View all container logs
docker compose logs -f

# View specific service logs
docker logs prometheus -f

# Restart all services
docker compose restart

# Stop all services
docker compose down

# Stop and remove all data
docker compose down -v

# Update to latest images
docker compose pull
docker compose up -d
```

## Security Hardening (Production)

1. **Change default passwords**:
   - Grafana admin password
   - Mattermost admin password

2. **Configure firewall**:
   ```bash
   # Only allow specific IPs to access services
   ufw allow from YOUR_IP to any port 3000
   ufw allow from YOUR_IP to any port 8065
   ```

3. **Enable HTTPS**:
   - Use reverse proxy (nginx/traefik)
   - Add SSL certificates

4. **Backup configuration**:
   ```bash
   # Regular backups of:
   tar -czf backup-$(date +%F).tar.gz \
     prometheus/prometheus.yml \
     prometheus/alerts.yml \
     alertmanager/alertmanager.yml \
     grafana/
   ```

## Uninstall

```bash
# Stop and remove everything
cd monitoring
docker compose down -v

# Remove repository
cd ..
rm -rf monitoring
```

## Getting Help

Check README.md for:
- Architecture overview
- Service descriptions
- Prometheus query examples
- Troubleshooting guide
