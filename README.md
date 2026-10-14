# cnc-watch — CNC Probe shop monitor

**A self-hosted monitor for a CNC machine shop.** It reads live status from Brother Speedio and Fanuc-controlled machines over the network. It shows a shop-floor dashboard, turns raw status into cycle and alarm events, sends those events to other systems through webhooks, and produces cycle-time reports. Everything runs on one Ubuntu box.

## Why it exists

Modern CNC controllers already know what they're doing: which program is running, how many parts they've made, when they stopped and why. Getting that information off the machine usually means expensive vendor software or someone walking the floor with a clipboard. CNC Probe talks to each machine's controller directly, keeps a compact history and turns it into things people can use:

- a live wall-board for the shop floor,
- cycle-time reports that show what a job actually takes,
- events that other systems can react to (a Slack alert on an alarm, an andon light, an ERP update).

## Features

- **Multi-protocol polling.** Each machine type is handled by a plugin:
  - `http_brother`: Brother Speedio controllers (SX1, R450X2, R650X1). It reads the controller's built-in web pages: running log, work counter, alarm log and list, tool data, status log, maintenance info and measurement results.
  - `focas_fanuc`: Fanuc 30i/31i controllers over FOCAS (TCP 8193), including Matsuura machines. It keeps a persistent socket and reads system info, run state, the main and executing program, and admin-configured macro variables (tool, part count, program, pallet).
  - `opcua_brother`: a placeholder for future OPC UA support.
- **Live dashboard.** A public, full-screen shop-floor view and a per-machine detail page, updated over WebSockets.
- **Event detection.** Turns raw polls into `cycle.started`, `cycle.completed` and `alarm.<CODE>` events. A "sticky program" rule stops subprogram calls (O8xxx/O9xxx) from corrupting cycle records.
- **Signed webhooks.** Subscribe any URL to events or event patterns, optionally filtered by machine or machine model. Every payload is HMAC-SHA256 signed and every delivery is logged.
- **Cycle-time reports.** Filter by machine, program and date range; view in the browser or export to CSV.
- **Google Sheets logging (optional).** Appends completed cycles to a sheet through a Google service account.
- **Alarm catalog.** A shop-wide list of every alarm code seen, with which machines and models raised it.
- **Machine tester.** Point it at an IP and protocol to check connectivity and see raw data before adding the machine.
- **Storage that stays small.** Polling runs about every 2 seconds, but history is written only when state actually changes, plus a periodic heartbeat. A daily maintenance timer trims old poll history and compacts the database. Cycle and alarm events are kept indefinitely.
- **One-command lifecycle.** `install.sh`, `update.sh` and `uninstall.sh`, with systemd units and an mDNS alias, so the box shows up as `cnc-probe.local`.

## How it works

```
 Brother (HTTP) ─┐                       ┌─► SQLite (WAL): current state,
 Fanuc (FOCAS)  ─┼─► poller.py ─► db.py ─┤   change-only history, events
 (plugins)      ─┘     │                 └─► webhooks.py ─► your endpoints
                        │                 └─► sheets.py   ─► Google Sheets
                        ▼
                   app.py (FastAPI) ─► dashboard, admin, reports (static/)
                                    └► WebSocket live streams
```

- `poller.py` runs one async polling loop per machine and passes each result to its protocol plugin.
- `db.py` updates a one-row-per-machine snapshot on every poll. It writes a history row only when the state signature changes (status, program, counter, tool) and detects cycle and alarm transitions.
- `webhooks.py` and `sheets.py` deliver events in the background, so a slow endpoint never stalls polling.
- `maintenance.py` runs daily from a systemd timer. It enforces the retention window (30 days by default) and runs `VACUUM`.

## Tech stack

Python 3 · FastAPI · Uvicorn · aiohttp / httpx · BeautifulSoup · pyfocas · SQLite (WAL) · vanilla HTML/CSS/JS · systemd · Avahi (mDNS)

## Requirements

- Ubuntu 22.04 LTS or newer (24.04 is the main test target)
- Root or sudo access, and internet access during install
- Network reach from the server to each machine controller
- Inbound `8765/tcp` (web UI) and `5353/udp` (mDNS) on the shop network

## Install

```bash
sudo apt update && sudo apt install -y git
sudo git clone https://github.com/bcrum81/cnc-watch.git /opt/cnc-probe
cd /opt/cnc-probe
sudo bash install.sh
```

The installer adds system packages, creates a virtualenv, installs dependencies and creates `config/` and `data/`. It also generates a random session key, seeds an admin login, installs and starts the systemd units, and opens the ports in `ufw` if it's active.

> The app expects to live at `/opt/cnc-probe` and run as the `cnc-probe` service. That path is hard-coded.

When install finishes, open:

| Page | URL |
|---|---|
| Dashboard | `http://cnc-probe.local:8765/` |
| Admin | `http://cnc-probe.local:8765/admin` |
| Integrations (webhooks, Sheets) | `http://cnc-probe.local:8765/admin/integrations` |
| Reports | `http://cnc-probe.local:8765/admin/reports` |
| Machine tester | `http://cnc-probe.local:8765/tester` |

If your network filters mDNS, use `http://<server-ip>:8765/` instead.

**Default login is `admin` / `admin`. Change it right after your first login** (Admin → Change Password). To use a different first-run password, set `CNC_DEFAULT_ADMIN_PASSWORD` before the first start.

### Update / uninstall

```bash
sudo bash /opt/cnc-probe/update.sh     # git pull, refresh deps and units, restart
sudo bash /opt/cnc-probe/uninstall.sh  # remove services; asks before deleting data
```

Neither script ever touches `config/` or `data/`.

## Configuration

Everything is configured in the web admin and stored as JSON under `config/`, which git ignores. Nothing machine-specific ships in the repo.

| File | Contents | Created by |
|---|---|---|
| `config/auth.json` | Admin username and password hash | installer |
| `config/secret_key` | Session signing key | installer |
| `config/machines.json` | Machines: name, IP, protocol, poll interval, macro numbers | Admin → Add Machine |
| `config/machine_order.json` | Dashboard display order | Admin |
| `config/webhooks.json` | Webhook subscriptions and their signing secrets | Integrations |
| `config/sheets.json` | Sheet ID, tab name, credentials path | Integrations |
| `config/google/credentials.json` | Google service-account key (optional) | you, by hand |
| `data/cnc_data.db` | SQLite history | app |

### Adding a machine

Admin → **Add Machine**. Pick a protocol, enter the controller's IP (for example `192.0.2.10`) and a poll interval. For FOCAS machines, also enter the macro numbers your shop uses for tool, part count, program and pallet. Use the **Tester** first to confirm the controller answers.

### Webhooks

Each subscription gets its own secret. Deliveries carry these headers:

```
X-CNC-Event: cycle.completed
X-CNC-Delivery-Id: <uuid>
X-CNC-Subscription-Id: <id>
X-CNC-Signature: sha256=<hex HMAC of the raw body>
```

To check a delivery, compute the HMAC-SHA256 of the raw request body with the subscription secret and compare it to the `X-CNC-Signature` value.

### Google Sheets (optional)

1. In Google Cloud, enable the Sheets API and create a service account with a JSON key.
2. Create a sheet with the headers `Date, Machine, Pallet, Program, Operation, Total Time (seconds)` and share it with the service account's email as an Editor.
3. Install the key:
   ```bash
   sudo install -m 600 your-key.json /opt/cnc-probe/config/google/credentials.json
   ```
4. In Integrations → Google Sheets, enter the Sheet ID (the part of the URL between `/d/` and `/edit`) and the tab name. Click **Test Connection**, enable it, and save.

## Operations

```bash
sudo systemctl status cnc-probe
sudo journalctl -u cnc-probe -f
systemctl list-timers cnc-probe-maintenance.timer

# Database maintenance by hand
sudo /opt/cnc-probe/venv/bin/python3 /opt/cnc-probe/maintenance.py --dry-run
sudo /opt/cnc-probe/venv/bin/python3 /opt/cnc-probe/maintenance.py --retention-days 60

# Audit / de-duplicate the events table
sudo /opt/cnc-probe/venv/bin/python3 /opt/cnc-probe/cleanup_events.py          # preview
sudo /opt/cnc-probe/venv/bin/python3 /opt/cnc-probe/cleanup_events.py --apply
```

## Project layout

```
app.py               FastAPI app: pages, REST API, WebSocket streams
poller.py            Async polling manager (one loop per machine)
db.py                SQLite schema, change-only storage, event detection
webhooks.py          Subscriptions + HMAC-signed delivery
sheets.py            Google Sheets background writer
maintenance.py       Retention prune + VACUUM (daily timer)
cleanup_events.py    Event-table audit tool
protocols/           Protocol plugins (http_brother, focas_fanuc, opcua_brother)
static/              Dashboard, admin, reports and tester UI
systemd/             Service, maintenance timer and mDNS alias units
bin/avahi-alias      mDNS CNAME publisher
scripts/             One-time migration helpers
install.sh / update.sh / uninstall.sh
```

## Status and origin

CNC Probe was built in-house at Henry Holsters, a manufacturer, to monitor a mixed fleet of Brother and Fanuc/Matsuura machining centers. It runs in production on our shop floor. The monitoring core is feature-complete: polling, dashboard, events, webhooks, reports, Sheets logging and the FOCAS plugin. OPC UA support is planned.

This repository is a public snapshot of the project.

## License

Copyright (C) 2026 Ben Crum.

Licensed under the [GNU Affero General Public License v3.0](LICENSE). You may use, modify and redistribute this software, but if you distribute it or run a modified version as a network service, you must make your source code available under the same license.
