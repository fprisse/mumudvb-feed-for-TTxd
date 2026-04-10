# DVB-T Teletext Extraction wtih mumudvb

## Context and Goal

This manual continues an existing project: **ttxd** — a C program that receives
an MPEG-TS stream over HTTP, extracts DVB teletext pages, and sends them as JSON
datagrams over UDP to a Node-RED flow for display in a FlowFuse Dashboard 2.0.

Currently ttxd receives its stream from an **HDHomeRun CONNECT HDHR4-2DT**
(DVB-T terrestrial tuner) at `http://192.168.1.154:5004/auto/v1`.

The goal of this build is to add a parallel path using an **RTL-SDR USB stick**
as a DVB-T tuner, served by **mumudvb** as a local HTTP MPEG-TS stream server.
This allows the same ttxd binary and Node-RED flow to work unchanged, and also
enables live video viewing of the same stream via VLC or any MPEG-TS client.

---

## Existing Infrastructure

| Component | Details |
|---|---|
| Machine | `echo` running Ubuntu 24 |
| Node-RED | Running on `echo`, port 1880 |
| FlowFuse Dashboard | `@flowfuse/node-red-dashboard` 1.30.2 |
| ttxd binary | `/usr/local/bin/ttxd` |
| ttxd service | `/etc/systemd/system/ttxd.service` |
| Current stream source | HDHomeRun at `http://192.168.1.154:5004/auto/v1` |
| Teletext output | UDP JSON to `127.0.0.1:5555` |

### ttxd usage
```
ttxd <ip>:<port> <channel> <pid> <udp-port>
```
Example:
```
ttxd 192.168.1.154:5004 1 7013 5555
```

### Current ttxd.service
```ini
[Unit]
Description=DVB Teletext service (HDHomeRun → UDP → Node-RED)
After=network.target

[Service]
ExecStart=/usr/local/bin/ttxd 192.168.1.154:5004 1 7013 5555
Restart=on-failure
User=user

[Install]
WantedBy=multi-user.target
```

Service is **not enabled** (no autostart). Started/stopped by Node-RED exec nodes
on a 600-second scan cycle.

---

## RTL-SDR Driver Policy

The machine has the following blacklists in place — **keep these**:

```
blacklist dvb_usb
blacklist dvb_usb_v2
blacklist rtl2832
blacklist dvb_usb_rtl28xxu
```

These prevent the kernel from automatically loading DVB drivers when an RTL-SDR
stick is plugged in, keeping the sticks free for SDR use (spectrum monitoring,
ADS-B, AIS etc.).

### Driver load/unload strategy

Drivers are loaded explicitly via `ExecStartPre` in the systemd service and
unloaded via `ExecStopPost`. On clean stop this works perfectly. On crash or
hard kill, the drivers stay loaded until next reboot — acceptable trade-off.

On every boot the blacklist guarantees a clean state. An additional cleanup
service runs on boot to handle the rare crash case.

The key insight: a blacklist prevents **automatic** kernel loading. An explicit
`modprobe` call in a service **overrides** the blacklist. These two mechanisms
do not conflict.

---

## Target Architecture

```
[RTL-SDR stick]
      |
      | DVB-T RF signal
      v
[dvb_usb_rtl28xxu driver] (loaded by mumudvb.service ExecStartPre)
      |
      | /dev/dvb/adapter0
      v
[mumudvb daemon]  ← /etc/mumudvb/mumudvb-dvbt.conf
      |
      | HTTP MPEG-TS unicast on 127.0.0.1:8080
      v
   ┌──┴──────────────────────┐
   │                         │
   v                         v
[ttxd]                   [VLC / any player]
   |
   | UDP JSON to 127.0.0.1:5555
   v
[Node-RED → FlowFuse Dashboard]
```

mumudvb runs as a **persistent service** serving the stream continuously.
ttxd continues to start/stop on the Node-RED scan cycle as before, but now
points at mumudvb instead of the HDHomeRun.

---

## Phase 1 — Scan and Identify

First establish the DVB-T parameters for NPO 1 in your area.

### Install tools
```bash
sudo apt install dvb-tools w-scan2
```

### Temporarily load drivers
```bash
sudo modprobe dvb_usb_rtl28xxu
sudo modprobe rtl2832
sleep 2
ls /dev/dvb/
```

### Scan for channels
```bash
w_scan2 -ft -c NL -C UTF-8 > /tmp/nl-dvbt.conf
```
or:
```bash
dvbv5-scan /usr/share/dvb/dvb-t/nl-Amsterdam > /tmp/nl-scan.conf
```

### Find NPO 1 parameters
From the scan output, note down for NPO 1:
- Frequency (Hz, e.g. `474000000`)
- Bandwidth (MHz, usually `8`)
- Service ID
- **Teletext PID** (may differ from HDHomeRun's `7013`)

To confirm teletext PID:
```bash
dvbsnoop -s ts -pd 3 0 | grep -i teletext
```

### Unload drivers
```bash
sudo rmmod dvb_usb_rtl28xxu rtl2832
```

---

## Phase 2 — Install mumudvb

```bash
sudo apt install mumudvb
```

If not in apt, build from source:
```bash
sudo apt install git build-essential libgcrypt-dev
git clone https://github.com/uvaliente/mumudvb
cd mumudvb
./configure && make && sudo make install
```

---

## Phase 3 — mumudvb Configuration

Create config file (fill frequency and bandwidth from Phase 1 scan):

```ini
# /etc/mumudvb/mumudvb-dvbt.conf

# DVB device
card=0
tuner=0

# DVB-T tuning parameters — fill from scan
freq=474000000
bandwidth=8
modulation=QAM_AUTO
transmission_mode=AUTO
guard=AUTO
delivery_system=DVBT

# Autoconfiguration reads PAT/SDT and maps all services on the multiplex
autoconfiguration=full

# HTTP unicast server — localhost only
unicast=1
unicast_port=8080
bind_address=127.0.0.1

# Logging
log_type=syslog
```

With `autoconfiguration=full` all services on the multiplex are discovered
automatically. Channels are served at:
```
http://127.0.0.1:8080/channel/1        (by logical channel number)
http://127.0.0.1:8080/bysid/1800       (by service ID — use this for reliability)
```

---

## Phase 4 — Boot Cleanup Service

Handles the case where mumudvb crashed on previous run and left drivers loaded:

```ini
# /etc/systemd/system/dvbt-driver-cleanup.service

[Unit]
Description=Unload DVB-T drivers on boot if left loaded from previous crash
DefaultDependencies=no
Before=mumudvb-dvbt.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c '/sbin/rmmod dvb_usb_rtl28xxu 2>/dev/null; /sbin/rmmod rtl2832 2>/dev/null; true'

[Install]
WantedBy=multi-user.target
```

Enable it:
```bash
sudo systemctl enable dvbt-driver-cleanup
```

---

## Phase 5 — mumudvb systemd Service

```ini
# /etc/systemd/system/mumudvb-dvbt.service

[Unit]
Description=mumudvb DVB-T stream server (RTL-SDR)
After=network.target dvbt-driver-cleanup.service

[Service]
Type=simple

# Load DVB-T drivers (overrides blacklist explicitly)
ExecStartPre=/sbin/modprobe dvb_usb_rtl28xxu
ExecStartPre=/sbin/modprobe rtl2832
# Wait for device to enumerate
ExecStartPre=/bin/sleep 2

ExecStart=/usr/bin/mumudvb -d -c /etc/mumudvb/mumudvb-dvbt.conf

# Unload drivers on clean stop
ExecStopPost=/bin/sh -c '/sbin/rmmod dvb_usb_rtl28xxu 2>/dev/null; /sbin/rmmod rtl2832 2>/dev/null; true'

Restart=on-failure
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
```

Deploy and test:
```bash
sudo systemctl daemon-reload
sudo systemctl start mumudvb-dvbt
journalctl -u mumudvb-dvbt -f
```

---

## Phase 6 — Verify Stream

```bash
# Check mumudvb channel list
curl http://127.0.0.1:8080/

# Test stream in VLC
vlc http://127.0.0.1:8080/bysid/1800

# Confirm MPEG-TS contains teletext PID
ffprobe http://127.0.0.1:8080/bysid/1800 2>&1 | grep -i teletext
```

---

## Phase 7 — Update ttxd Service

Once the teletext PID is confirmed from the mumudvb stream, update the service.

Note: the channel argument (`1`) maps to the logical channel in mumudvb's URL
scheme — confirm exact URL format from Phase 6.

```bash
sudo nano /etc/systemd/system/ttxd.service
```

Change ExecStart to point at mumudvb:
```ini
ExecStart=/usr/local/bin/ttxd 127.0.0.1:8080 1 <teletext-pid> 5555
```

The `<teletext-pid>` value is confirmed from the Phase 1 scan — it may differ
from the HDHomeRun's `7013`.

```bash
sudo systemctl daemon-reload
sudo systemctl start ttxd
journalctl -u ttxd -n 10
```

Verify teletext JSON arriving:
```bash
nc -ulk 5555
```

---

## Phase 8 — Node-RED Integration

No changes to the Node-RED flow are required. The exec nodes still call:
```
sudo /usr/bin/systemctl start ttxd
sudo /usr/bin/systemctl stop ttxd
```

The sudo permissions file already covers this:
```
nodered ALL=(ALL) NOPASSWD: /usr/bin/systemctl start ttxd, /usr/bin/systemctl stop ttxd
```

The mumudvb service runs independently and continuously — Node-RED does not
need to manage it.

---

## Notes and Caveats

- mumudvb with `autoconfiguration=full` streams the entire multiplex. NPO 1,
  NPO 2, NPO 3 and regional channels are typically on the same multiplex and
  will all be available via separate URLs.

- If the RTL-SDR stick index changes (multiple sticks), adjust `card=` in the
  mumudvb config. Use `card=0` for the first stick, `card=1` for the second.

- The HDHomeRun remains functional and unchanged. Both paths (HDHomeRun via
  ttxd directly, and RTL-SDR via mumudvb) can coexist — just update the
  ttxd.service ExecStart URL to switch between them.

- DVB-T teletext PIDs may differ from DVB-S/cable PIDs for the same channel.
  Always confirm from the actual scan rather than assuming PID 7013.

- mumudvb log output via `journalctl -u mumudvb-dvbt` will show which channels
  were discovered and their service IDs after successful tuning.
