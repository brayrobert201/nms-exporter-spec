---
id: SPEC-0011
title: 'Install, systemd units and the dev-deploy loop'
description: 'How the exporter and wall screens are installed to /opt, run as the Steam user under a hardened template unit, and tested on a scratch port before each deploy.'
type: snapshot
area: save-exporter
author: bob
created: 2026-10-23
last_updated: '2026-10-23'
status: current
depends_on: [SPEC-0002, SPEC-0009, SPEC-0010]
supersedes: []
superseded_by: []
related_specs: [SPEC-0001, SPEC-0012, SPEC-0082, SPEC-0091]
screenshots: []
---

# SPEC-0011: Install, systemd units and the dev-deploy loop

## Summary

The exporter runs unattended on the gaming PC for weeks. It is installed by copying the
repository's runtime files to `/opt/nms-exporter` and enabling a systemd **template** unit
instantiated with the Steam user's name, because only that user can read the Proton save
directory. The wall screens run as user units inside the desktop session. A dev-deploy script
tests the working tree on a scratch port, optionally renders a screen, then installs and reloads.
The database and secret-sync units of the later map work belong to `storage-deploy`.

## Inputs and outputs

Installed layout:

```
/opt/nms-exporter/
  nms_exporter.py  nmsexporter/  data/  conky/       (copied from the repo)
/etc/systemd/system/nms-exporter@.service
/etc/systemd/user/nms-conky@.service
/etc/nms-exporter/            (mode 700; optional s3.env, mode 600, root)
/var/lib/nms-exporter/        (StateDirectory: nms_state.json)
/var/cache/nms-exporter/      (CacheDirectory: mapping.json)
/usr/local/share/fonts/nms/   (portal glyph font, then fc-cache)
```

## Requirements

### SPEC-0011-REQ-001: install.sh

`sudo ./install.sh` shall: refuse unless root; take the Steam user from `NMS_USER`, else
`SUDO_USER`, else fail; copy the runtime files to `/opt/nms-exporter`; install both unit files and
the glyph font; create `/etc/nms-exporter` mode 700; `daemon-reload`; enable and **restart**
`nms-exporter@<user>` (restart so a re-install always runs the new code); print the URL and the
optional wall-screen steps. It is idempotent.

### SPEC-0011-REQ-002: Exporter unit

`nms-exporter@.service`:

```ini
[Service]
Type=simple
User=%i
WorkingDirectory=/opt/nms-exporter
ExecStart=/usr/bin/python3 /opt/nms-exporter/nms_exporter.py
Environment=NMS_EXPORTER_HOST=0.0.0.0
Environment=NMS_EXPORTER_PORT=9877
Environment=PYTHONUNBUFFERED=1
EnvironmentFile=-/etc/nms-exporter/s3.env
StateDirectory=nms-exporter
CacheDirectory=nms-exporter
Restart=on-failure
RestartSec=5s
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
ProtectKernelTunables=yes
ProtectControlGroups=yes
RestrictAddressFamilies=AF_INET AF_INET6
MemoryMax=512M
[Install]
WantedBy=multi-user.target
```

`ProtectHome=read-only` is the point: the service can read the save but never write it. The
leading `-` on `EnvironmentFile` makes the backup optional. (The reference later moved
`ExecStart` to a project virtualenv for the map's database driver; the save side needs only
system Python, SPEC-0012.)

### SPEC-0011-REQ-003: Wall-screen unit

`nms-conky@.service` (user unit), instance = `left` or `right`:
`ExecStart=/usr/bin/conky -c /opt/nms-exporter/conky/%i.conf`, `Restart=always`,
`RestartSec=10s`, `PartOf=` and `After=graphical-session.target`,
`WantedBy=graphical-session.target`. Enable with
`systemctl --user enable --now nms-conky@left nms-conky@right` after applying the window rules
(SPEC-0009-REQ-010).

### SPEC-0011-REQ-004: Reload wall screens without restart

After installing new renderer code, the deploy shall send `SIGUSR1` to the running conky
processes (conky reloads its config on USR1); the renderer is re-executed every `execpi`
interval anyway. The process pattern must match the unit's real command line (`conky -c
<install dir>/conky/`): the first version matched `^conky -d -c …`, which the unit never runs,
so "conky reloaded" was printed with nothing reloaded. When no conky is running the deploy says
so rather than claiming success.

### SPEC-0011-REQ-005: Dev-deploy

`scripts/dev-deploy.sh [--render] [--test]` shall:

1. byte-compile every Python file; stop on error;
2. start the working tree on port **9878** (`NMS_TEST_PORT`) with a scratch state file, wait up
   to 20 s for `/healthz`;
3. fetch `/json`, print `errors`, counts of farms/refiners/items and planner counts, and **fail if
   `errors` is non-empty**; count `nms_` lines in `/metrics`;
4. with `--render`, render `right.conf` in Xvfb (`-screen 0 2160x3840x24 -dpi 96`) against the
   test port into a PNG, with a private `XDG_RUNTIME_DIR` so it does not share the live screens'
   cache;
5. stop the test exporter **by the PID listening on its port**, never by process pattern (a
   pattern also matches the live service);
6. unless `--test`, run `install.sh` (non-interactive sudo), check the live `:9877/healthz`, and
   reload conky.

### SPEC-0011-REQ-006: Scraping

The scraper shall scrape `<gaming PC>:9877` every 60 s. The exporter decodes at most once per
new save regardless of scrape rate.

## Degraded behaviour

- **Steam user cannot be determined**: install fails before changing anything.
- **Exporter crashes**: systemd restarts it after 5 s; state survives in `StateDirectory`.
- **Memory runaway**: capped at 512 MB; the kernel kills it and systemd restarts it.
- **Desktop not running**: wall-screen units are tied to the graphical session and simply do not
  start; the exporter is unaffected.
- **Test exporter fails**: the deploy stops before installing, the live service keeps running.

## Verification

```bash
sudo ./install.sh
systemctl status nms-exporter@$USER          # active (running)
curl -fs localhost:9877/healthz              # ok
systemd-analyze security nms-exporter@$USER  # exposure well under the default
scripts/dev-deploy.sh --test --render        # errors: none; a PNG to look at
```

## Lessons

See [SPEC-0019](../lessons/SPEC-0019-save-exporter-lessons.md): stop test processes by port, not
by pattern; render at 96 dpi.
