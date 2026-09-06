# UltraFeeder + FR24 — ADS-B Ground Station Stack

A three-service docker-compose stack for [docker-adsb-ultrafeeder](https://github.com/sdr-enthusiasts/docker-adsb-ultrafeeder), [docker-flightradar24](https://github.com/sdr-enthusiasts/docker-flightradar24), and [docker-airnavradar](https://github.com/sdr-enthusiasts/docker-airnavradar), deployed as a **Portainer Stack from a public Git repo**.

Feeds: airplanes.live, adsb.fi, adsb.lol, ADS-B Exchange, Flightradar24, AirNav Radar.
Web UI: tar1090 (port 8080), FR24 status (port 8754).

## Prerequisites

- Raspberry Pi 3 (aarch64 / arm64) with Debian 13 or later
- RTL-SDR dongle (verified: `0bda:2838`, serial `1090`)
- Docker 29+ with Compose v5+
- Portainer CE running locally (Stacks / Compose mode)
- Antenna connected, USB dongle plugged in
- Your FR24 sharing key, ADS-B Exchange UUID, and AirNav Radar sharing key

## Deploy via Portainer

1. In Portainer, go to **Stacks → Add stack**.
2. Under **Build method**, select **Repository**.
3. Fill in:
   - **Repository URL:** `https://github.com/marcelo/ultrafeeder.git`
   - **Compose path:** `docker-compose.yml`
   - **Reference:** `main` (or the branch you want)
4. Click **Next**.
5. In the **Environment variables** panel, set the values below (copy from `.env.example`).
6. Click **Deploy the stack**.

> **Do NOT paste the compose file manually.** Git-based stacks allow you to redeploy by pulling the latest commit from Portainer.

## Environment Variables

Set these in the Portainer stack Environment panel. All have safe defaults — the stack boots even before you fill them in — but **the values marked REQUIRED must be set before feeds will work**.

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `FEEDER_NAME` | `ultrafeeder-pi3` | Yes | Callsign shown to aggregators; also the tar1090 page title |
| `FEEDER_TZ` | `Europe/Amsterdam` | No | Container timezone |
| `FEEDER_LAT` | `51.505` | **YES** | Latitude — placeholder, must override with real value |
| `FEEDER_LONG` | `5.391` | **YES** | Longitude — placeholder, must override with real value |
| `FEEDER_ALT_M` | `10` | Yes | Altitude in metres above ground |
| `ADSB_SDR_SERIAL` | `1090` | Yes | RTL-SDR iSerial — verify with `lsusb -v -d 0bda:2838 \| grep iSerial` |
| `ADSB_SDR_PPM` | `0` | No | Frequency correction — set to 0, tune later if needed |
| `READSB_GAIN` | `auto` | No | RF gain: `auto` or a numeric value (e.g. `49.6`) |
| `MULTIFEEDER_UUID` | *(empty)* | **YES** | Multi-feeder UUID — generate with `uuidgen` and set before deployment |
| `ADSBX_UUID` | *(empty)* | **YES** | Your ADS-B Exchange sharing UUID |
| `FR24_SHARING_KEY` | *(empty)* | **YES** | Your Flightradar24 sharing key |
| `AIRNAVRADAR_SHARING_KEY` | *(empty)* | **YES** | Your AirNav Radar (rbfeeder) sharing key |

### Generating your UUID

```bash
# On the Pi
uuidgen
# Example output: a1b2c3d4-e5f6-7890-abcd-ef1234567890
# Copy this into the MULTIFEEDER_UUID field in Portainer
```

## Ports

| Port | Service | Purpose |
|------|---------|---------|
| 8080 | ultrafeeder | tar1090 aircraft map (web UI) |
| 8754 | fr24 | Flightradar24 status page |

> **Warning:** The FR24 status page (port 8754) can stop the feeder. Never expose it to the internet — LAN only.

## Verification Checklist

After deploying, wait **2–3 minutes** for readsb to lock the SDR and connect to feeders, then run through:

### 1. Logs — no errors

```bash
# In Portainer: click the ultrafeeder container → Logs
# You should see:
#   readsb: device opened successfully
#   ULTRAFEEDER_CONFIG: adsb,feed.airplanes.live,30004 ...

# Click the fr24 container → Logs
# You should see:
#   [fr24] connection established to ultrafeeder:30005
#   [fr24] sharing data to FlightRadar24

# Click the airnavradar container → Logs
# You should see:
#   Connection established to ultrafeeder:30005
#   Connection with AirNav Radar server OK! Key accepted
```

### 2. Web UI — aircraft visible

Open `http://<PI_IP>:8080` in a browser. You should see aircraft on the map within a few minutes.

### 3. FR24 status — connected

Open `http://<PI_IP>:8754`. The page should show "Feeding FlightRadar24" and a positive aircraft count.

### 4. Aggregator dashboards

Check your feeder is visible on each aggregator's dashboard:

- **airplanes.live:** https://www.airplanes.live/stats.php (enter your feeder name)
- **adsb.fi:** https://adsb.fi/stats (look for your callsign)
- **adsb.lol:** https://adsb.lol/stats
- **ADS-B Exchange:** https://www.adsbexchange.com/stats/ (enter your UUID)
- **Flightradar24:** https://www.flightradar24.com/data/feeds (your FR24 dashboard)
- **AirNav Radar:** https://www.airnavradar.com (Account → Stations — claim receiver at https://www.airnavradar.com/raspberry-pi/claim)

## Troubleshooting

### readsb: "No supported devices found"

```bash
# Check dongle is detected
lsusb -d 0bda:2838

# Check device is accessible
docker exec ultrafeeder rtl_test -t
```

If the device isn't found, verify the `device_cgroup_rules` line in `docker-compose.yml` matches char major `189`:

```bash
ls -l /dev/bus/usb/001/004   # adjust bus/dev numbers to match your dongle
# Should show char 189,X
```

### fr24: "cannot connect to host"

This is normal for the first 1–2 minutes while ultrafeeder starts up. fr24 retries automatically. If it persists after 5 minutes, check that ultrafeeder is running and serving beast on port 30005:

```bash
docker logs ultrafeeder 2>&1 | grep -i "beast"
```

### airnavradar: "cannot connect to host"

Same as fr24 — normal for the first 1–2 minutes. If it persists after 5 minutes, verify ultrafeeder is serving beast:

```bash
docker logs airnavradar 2>&1 | tail -20
```

Also verify `AIRNAVRADAR_SHARING_KEY` is set in the Portainer environment panel.

### Wrong serial / multiple dongles

If you have more than one RTL-SDR, find the correct serial:

```bash
lsusb -v -d 0bda:2838 | grep iSerial
```

Set `ADSB_SDR_SERIAL` in the Portainer environment panel to match.

### Feeder not showing on aggregator dashboards

- Verify `FEEDER_LAT` / `FEEDER_LONG` / `FEEDER_ALT_M` are set to real values (not placeholders).
- Verify `ADSBX_UUID`, `FR24_SHARING_KEY`, and `AIRNAVRADAR_SHARING_KEY` are set in the panel.
- Wait 15 minutes — some dashboards update on a delay.

## Updating

### Stack image updates (manual)

In Portainer, open the stack → **Editor** → **Repull image and redeploy**. This pulls the latest image tag and recreates the containers.

### Compose file updates (from Git)

If you push a change to the compose file (e.g. adding a feeder), in Portainer go to the stack → **Editor** → **Save and redeploy**. Portainer pulls the latest commit from the repo.

## File Structure

```
ultrafeeder/
├── docker-compose.yml   # Stack definition (ultrafeeder + fr24 + airnavradar)
├── .env.example         # Annotated variable catalog (reference only)
├── .gitignore           # Ignores .env files with real secrets
└── README.md            # This file
```

## Architecture

```
RTL-SDR dongle
    │
    ▼
┌──────────────────────────────────────────┐
│  ultrafeeder                             │
│  ghcr.io/sdr-enthusiasts/docker-adsb-   │
│  ultrafeeder:latest                      │
│                                          │
│  readsb ──► feeder ──► airplanes.live    │
│           │          ► adsb.fi           │
│           │          ► adsb.lol          │
│           │          ► adsbexchange.com  │
│           │                              │
│  mlathub ◄── mlat results from above    │
│  tar1090 ──► web UI :8080               │
│                                          │
│  beast output :30005 ────────────────┐   │
└──────────────────────────────────────│───┘
                                       │
                        ┌──────────────┴──────────────┐
                        ▼                             ▼
               ┌────────────────┐           ┌────────────────┐
               │  fr24           │           │  airnavradar    │
               │  docker-fligh-  │           │  docker-airnav- │
               │  tradar24:latest│           │  radar:latest   │
               │                 │           │                 │
               │  reads beast    │           │  reads beast    │
               │  from ultra-    │           │  from ultra-    │
               │  feeder:30005   │           │  feeder:30005   │
               │                 │           │                 │
               │  feeds FR24     │           │  feeds AirNav   │
               │  status :8754   │           │  Radar          │
               └─────────────────┘           └─────────────────┘
```
