# Zendesk Audit Logs → Coralogix

API integration that polls Zendesk **account audit logs** and ships them to Coralogix.

**Cadence:** every **5 minutes**, collect the **last 5 minutes** of activity, then POST to Coralogix.

```
Zendesk Support API                  Coralogix Logs API
GET /api/v2/audit_logs   →  poller  →  POST /logs/v1/singles
(last 5 minutes)                         application: zendesk
                                         subsystem: audit-logs
```

Production mode is **cron every 5 minutes** with `--once`. Overlaps are safe because the poller checkpoints event IDs in `state.json`.

## APIs used

| Direction | API | Auth |
| --- | --- | --- |
| Pull | Zendesk `GET /api/v2/audit_logs` | API token (`{email}/token:{api_token}`) |
| Push | Coralogix `POST https://ingress.<domain>/logs/v1/singles` | Send-Your-Data API key (Bearer) |

This is **account** audit activity (users, roles, rules, API tokens, logins, exports). It is not ticket comment history (`/api/v2/tickets/{id}/audits`).

Zendesk audit logs require an **Enterprise** plan (or above) and an **admin** API token.

`ZENDESK_EMAIL` is the Zendesk API username, not a mailbox login. Zendesk API tokens always authenticate as `{email}/token`.

## Prerequisites

- Python 3.10+
- Zendesk admin API token  
  Admin Center → **Apps and integrations → APIs → Zendesk API → Add API token**
- Coralogix Send-Your-Data API key  
  Coralogix → **Data Flow → API Keys**
- Zendesk subdomain (`acme` in `https://acme.zendesk.com`)
- Coralogix domain (`eu1.coralogix.com`, `coralogix.in`, …)

## Quick start

```bash
git clone <this-repo-url>
cd zendesk-coralogix

python3 -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env
chmod +x run.sh
```

Edit `.env`:

```bash
ZENDESK_SUBDOMAIN=your-subdomain
ZENDESK_EMAIL=admin@example.com
ZENDESK_API_TOKEN=...

CORALOGIX_SEND_YOUR_DATA_KEY=...
CORALOGIX_DOMAIN=eu1.coralogix.com
CORALOGIX_APPLICATION_NAME=zendesk
CORALOGIX_SUBSYSTEM_NAME=audit-logs

POLL_INTERVAL_SECONDS=300
LOOKBACK_MINUTES=5
```

EU / AU Zendesk hosts: set `ZENDESK_DOMAIN=zendesk.eu` (or `zendesk.com.au`), or `ZENDESK_BASE_URL=https://your-subdomain.zendesk.eu`.

### Test one 5-minute window (no ingest)

```bash
python ship_zendesk_to_coralogix.py --once --dry-run --lookback-minutes 5
```

### Test one 5-minute window (send to Coralogix)

```bash
python ship_zendesk_to_coralogix.py --once --lookback-minutes 5
```

### Optional in-process loop

```bash
python ship_zendesk_to_coralogix.py
```

This sleeps `POLL_INTERVAL_SECONDS` (default `300`) between cycles. Prefer cron in production so the job restarts after reboot.

## Cron every 5 minutes

### 1. Install on the host

```bash
sudo mkdir -p /opt/zendesk-coralogix
sudo git clone <this-repo-url> /opt/zendesk-coralogix
cd /opt/zendesk-coralogix

python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
chmod +x run.sh
# put real API keys in .env
```

### 2. Confirm a manual run

```bash
/opt/zendesk-coralogix/run.sh
```

You should see a log line such as `Done. Total events shipped: N` (N may be `0` if nothing changed in the last 5 minutes).

### 3. Install crontab

```bash
crontab -e
```

Add:

```cron
# Zendesk audit logs → Coralogix (last 5 minutes, every 5 minutes)
*/5 * * * * /opt/zendesk-coralogix/run.sh >> /var/log/zendesk-coralogix.log 2>&1
```

`run.sh` already passes `--once --lookback-minutes 5`.

Without the wrapper:

```cron
*/5 * * * * cd /opt/zendesk-coralogix && .venv/bin/python ship_zendesk_to_coralogix.py --once --lookback-minutes 5 >> /var/log/zendesk-coralogix.log 2>&1
```

Create the log file once so cron can write to it:

```bash
sudo touch /var/log/zendesk-coralogix.log
sudo chown "$(whoami)" /var/log/zendesk-coralogix.log
```

### 4. Verify cron

```bash
crontab -l
tail -f /var/log/zendesk-coralogix.log
```

Wait until the next 5-minute boundary (`:00`, `:05`, `:10`, …).

### macOS notes

- Grant **Full Disk Access** to `/usr/sbin/cron` if jobs do not run.
- Use the absolute venv Python path (as in the examples).
- For a laptop, a launchd plist or keeping the machine awake is more reliable than cron.

### systemd timer alternative (Linux)

`/etc/systemd/system/zendesk-coralogix.service`:

```ini
[Unit]
Description=Ship Zendesk audit logs to Coralogix

[Service]
Type=oneshot
WorkingDirectory=/opt/zendesk-coralogix
EnvironmentFile=/opt/zendesk-coralogix/.env
ExecStart=/opt/zendesk-coralogix/.venv/bin/python ship_zendesk_to_coralogix.py --once --lookback-minutes 5
```

`/etc/systemd/system/zendesk-coralogix.timer`:

```ini
[Unit]
Description=Run Zendesk → Coralogix every 5 minutes

[Timer]
OnCalendar=*:0/5
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now zendesk-coralogix.timer
sudo systemctl list-timers | grep zendesk
```

## How the 5-minute window works

Each run asks Zendesk for `created_at` between **now − 5 minutes** and **now**.

`state.json` (path via `STATE_FILE`) stores:

- recent audit log IDs (dedupe if cron overlaps)
- last shipped `created_at`

If a cron run is missed, the next cycle catches up from the last checkpoint instead of dropping events.

## Finding logs in Coralogix

In **Explore** / **Logs**:

- Application: `zendesk` (or `CORALOGIX_APPLICATION_NAME`)
- Subsystem: `audit-logs` (or `CORALOGIX_SUBSYSTEM_NAME`)
- Time picker must cover the Zendesk `created_at` timestamp

Useful fields inside `text`:

- `event_source`: `zendesk_audit`
- `action`: `create`, `update`, `destroy`, `login`, `exported`
- `actor_name`, `actor_id`, `ip_address`
- `source_type`, `source_label`, `change_description`

## Optional filters

```bash
ZENDESK_FILTER_ACTIONS=create,update,destroy,login,exported
ZENDESK_FILTER_SOURCE_TYPES=user,rule,apitoken
```

Leave empty to collect all audit events.

## CLI

```text
python ship_zendesk_to_coralogix.py --help

--once                  One cycle and exit (use with cron)
--dry-run               Fetch but do not send or save state
--lookback-minutes N    Override LOOKBACK_MINUTES (default 5)
--lookback-hours N      Convenience override
--log-level LEVEL       DEBUG, INFO, WARNING, ERROR
```

## Security

- Never commit `.env`. It is gitignored.
- Use an admin API token that can read audit logs.
- Rotate Zendesk and Coralogix keys if they are exposed.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| `Couldn't authenticate you` (401) | Wrong email/token pair, or token was not created by that admin. |
| `403 Forbidden` | Token user is not an admin, or the account is not on Enterprise. |
| `404` | Wrong `ZENDESK_SUBDOMAIN`. Use the `acme` part of `acme.zendesk.com`. |
| No logs in Coralogix | Wrong `CORALOGIX_DOMAIN`, or Explore time range misses `created_at`. |
| `Missing required environment variable` | `.env` missing in the cron working directory. |
| Cron runs with no output | Check absolute paths, venv Python, and log file permissions. |
| Rate limited (`429`) | Poller retries using `Retry-After`. Set `ZENDESK_PAGE_DELAY_SECONDS=1` if needed. |

## License

Use and modify freely for your organization.
