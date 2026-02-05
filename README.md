# Network Monitoring Stack with Alerting

Complete monitoring stack using Prometheus, Blackbox Exporter, Grafana, Alertmanager, and Mattermost for infrastructure health monitoring and alerting.

## Architecture

```
monitoring/
├── docker-compose.yml          # Main orchestration file
├── prometheus/
│   ├── prometheus.yml          # Prometheus configuration
│   └── alerts.yml              # Alert rules
├── blackbox/
│   └── blackbox.yml            # Blackbox Exporter probes configuration
├── alertmanager/
│   ├── alertmanager.yml        # Alertmanager configuration (not in git)
│   ├── alertmanager.yml.template  # Template for alertmanager.yml
│   └── mattermost.tmpl         # Alert message template
└── grafana/
    ├── datasources.yml         # Prometheus datasource configuration
    └── blackbox_dashboard.json # Pre-configured monitoring dashboard
```

## Services

### Prometheus
- **Port**: 9091 (mapped from internal 9090)
- **URL**: http://localhost:9091
- **Purpose**: Metrics collection and alert rule evaluation
- **Configuration**: 
  - Scrapes Blackbox Exporter every 15 seconds
  - Evaluates alert rules every 15 seconds
  - Sends alerts to Alertmanager
- **Monitored Targets**:
  - Google.com, Cloudflare.com (external benchmarks)
  - Grafana (internal service)
  - VyOS Router: 10.102.13.215 (DNS, ICMP, TCP checks)

### Blackbox Exporter
- **Port**: 9115
- **URL**: http://localhost:9115
- **Purpose**: Active probing of endpoints (HTTP, DNS, TCP, ICMP)
- **Modules**:
  - `http_2xx`: HTTP/HTTPS with 2xx response validation
  - `tcp_connect`: TCP port connectivity check
  - `icmp`: ICMP ping check
  - `dns_check`: DNS resolution check

### Grafana
- **Port**: 3000
- **URL**: http://localhost:3000
- **Default Credentials**: admin / admin
- **Purpose**: Metrics visualization and dashboards
- **Features**:
  - Pre-configured Prometheus datasource
  - Blackbox monitoring dashboard with 9 panels
  - 5-second auto-refresh
  - Probe success/failure tracking
  - Response time graphs
  - SSL certificate status

### Alertmanager
- **Port**: 9094 (mapped from internal 9093)
- **URL**: http://localhost:9094
- **Purpose**: Alert routing, grouping, and notification delivery
- **Integration**: Sends alerts to Mattermost via webhook
- **Features**:
  - 5-minute grouping interval
  - 10-second repeat interval for testing
  - Resolved alert notifications
  - Custom message templates

### Mattermost
- **Port**: 8065
- **URL**: http://localhost:8065
- **Purpose**: Team collaboration and alert notification display
- **Default Admin**: Create on first login
- **Features**:
  - Incoming webhook integration
  - Custom bot usernames per alert source
  - Alerts channel for monitoring notifications

### PostgreSQL (Mattermost Backend)
- **Port**: 5432 (internal only)
- **Purpose**: Mattermost database backend
- **Database**: mattermost
- **User**: mmuser

## Alert Rules

### 1. ProbeDown
- **Trigger**: probe_success == 0 for 2 minutes
- **Severity**: critical
- **Description**: Target is unreachable

### 2. HttpStatusNotOK
- **Trigger**: HTTP status code not 2xx for 1 minute
- **Severity**: warning
- **Description**: HTTP endpoint returning errors

### 3. ProbeSlow
- **Trigger**: Probe duration > 5 seconds for 5 minutes
- **Severity**: warning
- **Description**: Target responding slowly

### 4. SSLCertExpiringSoon
- **Trigger**: SSL certificate expires in < 30 days
- **Severity**: warning
- **Description**: SSL certificate needs renewal

### 5. MultipleProbesDown
- **Trigger**: More than 3 probes down simultaneously for 5 minutes
- **Severity**: critical
- **Description**: Widespread connectivity issues

## Monitored Endpoints

### HTTP Checks (blackbox-http)
- https://www.google.com
- https://www.cloudflare.com
- http://grafana:3000 (internal)

### DNS Checks (blackbox-dns)
- 8.8.8.8 (Google DNS)
- 1.1.1.1 (Cloudflare DNS)
- 10.102.13.215 (VyOS)

### ICMP Checks (blackbox-icmp)
- 8.8.8.8 (Google DNS)
- 1.1.1.1 (Cloudflare DNS)
- 10.102.13.215 (VyOS)

### TCP Checks (blackbox-tcp)
- grafana:3000 (Grafana UI)
- prometheus:9090 (Prometheus UI)
- google.com:443 (HTTPS)
- 10.102.13.215:22 (VyOS SSH)
- 10.102.13.215:80 (VyOS HTTP)
- 10.102.13.215:443 (VyOS HTTPS)

## Quick Start

```bash
# 1. Clone repository
git clone <repository-url>
cd monitoring

# 2. Create Alertmanager config from template
cp alertmanager/alertmanager.yml.template alertmanager/alertmanager.yml

# 3. Start all services
docker compose up -d

# 4. Wait for services to start (~30 seconds)
docker compose ps

# 5. Access Mattermost and create webhook
# - Open http://localhost:8065
# - Create account and team
# - Go to Integrations → Incoming Webhooks
# - Create webhook, copy the URL

# 6. Update Alertmanager configuration
# Edit alertmanager/alertmanager.yml:
#   - Replace webhook URL with your actual Mattermost webhook
#   - Save file

# 7. Restart Alertmanager
docker restart alertmanager

# 8. Enable Mattermost webhook overrides
# System Console → Integrations:
#   - Enable integrations to override usernames: True
#   - Enable integrations to override profile picture icons: True

# 9. Access services
# Grafana: http://localhost:3000 (admin/admin)
# Prometheus: http://localhost:9091
# Alertmanager: http://localhost:9094
# Mattermost: http://localhost:8065
```

## Verification

```bash
# Check container status
docker compose ps

# View logs
docker compose logs -f

# Check Prometheus targets
curl http://localhost:9091/api/v1/targets | jq

# Check active alerts
curl http://localhost:9094/api/v2/alerts | jq

# Send test alert
curl -H "Content-Type: application/json" -d '[{
  "labels": {"alertname":"TestAlert","severity":"warning"},
  "annotations": {"summary":"Test alert","description":"Testing"}
}]' http://localhost:9094/api/v2/alerts
```

## Demo - Trigger Alert

```bash
# Stop Grafana to trigger ProbeDown alert (fires after 2 minutes)
docker stop grafana

# Wait 2-3 minutes, check Mattermost for alert

# Restart Grafana (triggers resolved notification)
docker start grafana
```

## Custom Python Integration

To send alerts from custom Python scripts to the same Mattermost channel:

```python
import requests

webhook_url = "http://mattermost:8065/hooks/YOUR_WEBHOOK_ID"

payload = {
    "username": "Python Script",     # Custom bot name
    "icon_emoji": ":snake:",          # Custom icon
    "text": "🐍 Custom alert from Python!"
}

requests.post(webhook_url, json=payload)
```

## Useful Prometheus Queries

```promql
# Check if endpoint is up
probe_success{job="blackbox-http"}

# HTTP response time
probe_duration_seconds{job="blackbox-http"}

# SSL certificate expiry (days)
(probe_ssl_earliest_cert_expiry - time()) / 86400

# DNS lookup time
probe_dns_lookup_time_seconds{job="blackbox-dns"}

# ICMP round-trip time
probe_icmp_duration_seconds{job="blackbox-icmp"}

# HTTP status codes
probe_http_status_code{job="blackbox-http"}
```

## Maintenance

```bash
# Stop all services
docker compose down

# Stop and remove volumes (deletes all data)
docker compose down -v

# Restart single service
docker restart <service-name>

# View service logs
docker logs <service-name>

# Update configuration and reload
docker compose restart prometheus
```

## Security Notes

- **Secrets Management**: `alertmanager/alertmanager.yml` is gitignored (contains webhook URL)
- **Template System**: Use `alertmanager.yml.template` for version control
- **Volumes**: Persistent data stored in Docker volumes (excluded from git)
- **Environment Files**: `.env` files are gitignored

## Troubleshooting

### Alerts not appearing in Mattermost
1. Check Alertmanager logs: `docker logs alertmanager`
2. Verify webhook URL in `alertmanager/alertmanager.yml`
3. Ensure Mattermost username override is enabled
4. Test webhook directly: `curl -X POST <webhook-url> -H 'Content-Type: application/json' -d '{"text":"test"}'`

### Grafana not showing data
1. Verify Prometheus is running: `curl http://localhost:9091/-/healthy`
2. Check datasource UID matches: Should be `prometheus`
3. Reload provisioning: Restart Grafana

### Targets down in Prometheus
1. Check Blackbox Exporter: `curl http://localhost:9115/probe?target=google.com&module=http_2xx`
2. Verify network connectivity from containers
3. Check Prometheus config: `docker exec prometheus cat /etc/prometheus/prometheus.yml`

## Port Reference

| Service | Internal Port | External Port | Purpose |
|---------|--------------|---------------|---------|
| Prometheus | 9090 | 9091 | Metrics & Alerts |
| Blackbox Exporter | 9115 | 9115 | Probing |
| Grafana | 3000 | 3000 | Dashboards |
| Alertmanager | 9093 | 9094 | Alert Routing |
| Mattermost | 8065 | 8065 | Notifications |
| PostgreSQL | 5432 | - | Database (internal) |

## VM Deployment Steps

After pulling this repository on your VM, follow these steps:

### 1. Initial Setup
```bash
# Pull repository
git pull

# Navigate to monitoring directory
cd monitoring

# Copy Alertmanager template
cp alertmanager/alertmanager.yml.template alertmanager/alertmanager.yml
```

### 2. Start Services
```bash
# Start all containers
docker compose up -d

# Wait 30 seconds for all services to initialize
sleep 30

# Verify all containers are running
docker compose ps
```

### 3. Configure Mattermost
```bash
# Open Mattermost in browser
# http://YOUR_VM_IP:8065

# 1. Create first admin account
# 2. Create a team (e.g., "Monitoring")
# 3. Create a channel called "alerts"
# 4. Go to: Main Menu → Integrations → Incoming Webhooks
# 5. Add Incoming Webhook
#    - Title: "Prometheus Alerts"
#    - Channel: alerts
#    - Save
# 6. Copy the webhook URL (looks like: http://mattermost:8065/hooks/XXXXX)
```

### 4. Configure Alertmanager
```bash
# Edit alertmanager configuration
nano alertmanager/alertmanager.yml

# Replace the webhook URL on line with api_url:
# Change: http://mattermost:8065/hooks/CHANGE_ME
# To: http://mattermost:8065/hooks/YOUR_ACTUAL_WEBHOOK_ID

# Save and exit (Ctrl+X, Y, Enter)

# Restart Alertmanager
docker restart alertmanager
```

### 5. Enable Mattermost Overrides
```bash
# In Mattermost web interface:
# 1. Click on your profile picture → System Console
# 2. Navigate to: Integrations → Integration Management
# 3. Enable the following:
#    ✓ Enable integrations to override usernames
#    ✓ Enable integrations to override profile picture icons
# 4. Save
```

### 6. Test the Setup
```bash
# Send a test alert
curl -H "Content-Type: application/json" -d '[{
  "labels": {"alertname":"TestAlert","severity":"warning"},
  "annotations": {"summary":"VM deployment test","description":"Testing alert system"}
}]' http://localhost:9094/api/v2/alerts

# Check Mattermost #alerts channel - you should see the alert from "Prometheus Monitor"
```

### 7. Access Services
- **Grafana**: http://YOUR_VM_IP:3000 (admin/admin)
- **Prometheus**: http://YOUR_VM_IP:9091
- **Alertmanager**: http://YOUR_VM_IP:9094
- **Mattermost**: http://YOUR_VM_IP:8065

### 8. Final Verification
```bash
# Check all targets are UP
curl -s http://localhost:9091/api/v1/targets | jq -r '.data.activeTargets[] | "\(.labels.job): \(.health)"'

# All should show "up"
```

### Quick Troubleshooting on VM
```bash
# If alerts don't appear:
docker logs alertmanager --tail 50

# If Grafana has no data:
docker logs prometheus --tail 50

# If containers keep restarting:
docker compose logs --tail 100

# Restart everything:
docker compose down && docker compose up -d
```
