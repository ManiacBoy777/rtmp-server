# rtmp-server

**Record an Osee GoStream straight to your PC — wirelessly, no OBS in the loop.** This repository is a two-file, Dockerized [MediaMTX](https://mediamtx.org) server that accepts an RTMP stream from an Osee GoStream switcher over your local network and records it directly to your PC's `Videos` folder as fragmented MP4. Clone it, run one `docker compose` command, point the GoStream at the PC — every stream it publishes is recorded automatically.

![MediaMTX](https://img.shields.io/badge/server-MediaMTX-111111)
![Docker](https://img.shields.io/badge/deploy-Docker%20Compose-2496ed)
![Ingest](https://img.shields.io/badge/ingest-RTMP%20%3A1935-2f6db3)
![Recording](https://img.shields.io/badge/recording-fMP4-brightgreen)
![Branch](https://img.shields.io/badge/branch-main-555555)
![Visibility](https://img.shields.io/badge/repo-public-brightgreen)

> **Just want it recording?** Install Docker Desktop, clone this repo, run `docker compose up -d`, then set the GoStream's stream URL to `rtmp://<your-pc-ip>:1935/live` and pick any stream key — the key becomes the subfolder your recording lands in. Full walkthrough in [Quick Start](#quick-start) and [Publishing from the GoStream](#publishing-from-the-gostream).

---

## Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [Repository Layout](#repository-layout)
- [Quick Start](#quick-start)
- [Publishing from the GoStream](#publishing-from-the-gostream)
- [Watching and Playing Back](#watching-and-playing-back)
- [Configuration Reference](#configuration-reference)
- [Operations](#operations)
- [Security Model](#security-model)
- [Troubleshooting](#troubleshooting)
- [Maintenance](#maintenance)
- [Known Limitations](#known-limitations)
- [Credits](#credits)

---

## Overview

| | |
| --- | --- |
| **Purpose** | Record a livestream from an Osee GoStream switcher directly to a PC, over the LAN, without OBS or capture hardware |
| **Server** | [MediaMTX](https://mediamtx.org) (`bluenviron/mediamtx` image) — a single Docker container |
| **Ingest** | RTMP on port **1935**, path `live` + a stream key you choose per event |
| **Recording** | Automatic for every published path (`record: yes`), fragmented MP4 (`fmp4`) |
| **Recording destination** | `<your Windows user>\Videos\Livestream\<stream key>\` on the host PC |
| **Also published** | RTSP `:8554`, HLS `:8888`, WebRTC `:8889`, SRT `:8890`, MediaMTX API `:9997` |
| **Deploy** | `docker compose up -d` — one service, `restart: unless-stopped` |
| **Files you edit** | `mediamtx.yml` (server/recording settings) and the recordings volume path in `compose.yml` |

### Design principles

1. **The switcher is the recorder's remote control.** There is nothing to start or stop on the PC: MediaMTX records any path the moment it is published, so pressing *live* on the GoStream is the entire workflow.
2. **Two files, no code.** The whole server is `compose.yml` + `mediamtx.yml`. There is no application to build, patch, or maintain — MediaMTX does the work.
3. **The stream key is the filing system.** Each event gets its own key, and the key becomes the subfolder under `Videos\Livestream\` — recordings sort themselves.
4. **Standard protocols out.** The same stream is simultaneously available over RTSP, HLS, WebRTC, and SRT for monitoring or other consumers, because MediaMTX republishes every path it receives.

---

## How It Works

```text
   Osee GoStream switcher
   (program feed, Stream slot
    "PC Recording" enabled)
            │  RTMP publish
            │  rtmp://<pc-ip>:1935/live   key: <event name>
            ▼
┌──────────────────────────────────────────────────────────┐
│  MediaMTX container  (compose.yml, mediamtx.yml)         │
│                                                          │
│   paths.all → record: yes, format fmp4                   │
│   recordPath /recordings/%path/livestream-<date>_<time>  │
│                                                          │
│   republish:  RTSP :8554 · HLS :8888 · WebRTC :8889      │
│               SRT  :8890 · API :9997                     │
└───────────────────────┬──────────────────────────────────┘
                        │  volume mount
                        ▼
   Host PC:  %USERPROFILE%\Videos\Livestream  →  /recordings/live
                        │
                        ▼
   Videos\Livestream\<event name>\livestream-<date>_<time>.mp4
```

**Step by step:**

1. The GoStream publishes its program feed over the LAN to `rtmp://<pc-ip>:1935/live`, using the stream key you set (for example `my_event`). In MediaMTX terms the published path is `live/my_event`.
2. `mediamtx.yml` sets `record: yes` on `paths: all`, so MediaMTX records *every* published path, in `fmp4` format, using the `recordPath` template.
3. The container's `/recordings` tree is backed by the host's `Videos\Livestream` folder (mounted at `/recordings/live` in `compose.yml`), so the finished file appears on the PC at `Videos\Livestream\my_event\livestream-<date>_<time>.mp4` — a fragmented MP4 that plays in standard players and editors.
4. While the stream is live, the same feed can be monitored over HLS, RTSP, or WebRTC at the paths in [Watching and Playing Back](#watching-and-playing-back).

> **A note on the filename template.** `recordPath` is `"/recordings/%path/livestream-%d-%m-%Y_%H-%M-%s"` — `%path` expands to the published path (`live/<stream key>`) and the remaining tokens stamp the recording's start date and time into the filename, so repeated events in the same folder never overwrite each other.

---

## Repository Layout

| Path | Purpose |
| --- | --- |
| `compose.yml` | The deployment: one `mediamtx` service (image, published ports, volumes, restart policy). |
| `mediamtx.yml` | The server configuration mounted into the container at `/mediamtx.yml`: HLS and API listeners, and the record-everything path rule with its path template and format. |
| `README.md` | This document. |

That's the entire repository — no source code, no build step, no CI, and no `LICENSE` file. Everything else is MediaMTX's own behavior, configured by the two files above.

---

## Quick Start

### Prerequisites

- **Docker Desktop** (Docker Engine + Docker Compose) on the PC that will hold the recordings.
- **Git** to clone the repository.
- The GoStream and the PC on the **same local network**, and the PC's LAN IP address (`ipconfig` in a terminal on Windows).

On Windows, Git and Docker Desktop install in one line:

```powershell
winget install --id Git.Git -h && winget install --id Docker.DockerDesktop -h
```

### Start the server

```powershell
git clone https://github.com/ManiacBoy777/rtmp-server.git
cd rtmp-server
docker compose up -d
```

### Verify it's up

```powershell
docker compose ps        # mediamtx: running
docker logs mediamtx     # listeners starting, including RTMP on :1935
```

Then configure the GoStream (next section) and start streaming — a new folder and an `.mp4` file will appear under `Videos\Livestream\` within moments of going live.

### Stop the server

```powershell
docker compose down
```

Recordings already written are ordinary files in `Videos\Livestream\` and are unaffected by stopping or removing the container.

---

## Publishing from the GoStream

Done once per GoStream, from the **GoStream Control** software on the PC:

1. **Create the stream platform.** Go to `Stream > Create Stream XML File`:
   - **Platform Name:** something descriptive, e.g. `PC Recording`.
   - **URL:** `rtmp://<your-pc-ip>:1935/live` — replace `<your-pc-ip>` with the PC's LAN address (run `ipconfig` to find it).
   - Click **EXPORT FILE**.
2. **Arm a stream slot.** Go to `Stream > Open Streaming Dialogue`:
   - Select one of the three available stream slots.
   - Under **Platform**, choose the platform you just created (`PC Recording`).
   - In the **Stream Key** field, enter the name for this event (e.g. `my_event`). **The key is the subfolder name** — the recording lands in `Videos\Livestream\my_event\`.
   - Ensure **Enable Live** is checked.
3. **Go live.** Start streaming from the GoStream. MediaMTX accepts the publish and records automatically; stop the stream to finalize the file.

Per event, only step 2's stream key changes — a new key, a new folder.

---

## Watching and Playing Back

Anything published to MediaMTX is republished on every protocol the container exposes. With a stream live at path `live/<stream key>`:

| Consumer | Address |
| --- | --- |
| **HLS** (browsers, players) | `http://<pc-ip>:8888/live/<stream key>/` |
| **RTSP** (VLC, OBS media/VLC source, NVRs) | `rtsp://<pc-ip>:8554/live/<stream key>` |
| **WebRTC** (browser page served by MediaMTX) | `http://<pc-ip>:8889/live/<stream key>` |
| **Finished recordings** | Files under `Videos\Livestream\<stream key>\` — fragmented MP4, playable directly |

HLS is explicitly enabled in `mediamtx.yml` (`hls: yes` on `:8888`); the RTSP, WebRTC, and SRT listeners are MediaMTX's standard listeners, published by `compose.yml`.

---

## Configuration Reference

### `compose.yml` — deployment

| Setting | Value | Meaning |
| --- | --- | --- |
| `services.mediamtx.image` | `bluenviron/mediamtx` | The official MediaMTX image. **No tag is pinned** — `docker compose pull` moves to whatever is current; see [Maintenance](#maintenance). |
| `container_name` | `mediamtx` | Fixed name, so `docker logs mediamtx` always works |
| `restart` | `unless-stopped` | The server comes back after reboots and crashes until you explicitly stop it |
| `volumes` (recordings) | `${USERPROFILE}\Videos\Livestream:/recordings/live` | Host folder that receives recordings. Windows-specific: it relies on the `USERPROFILE` environment variable. Change the left side to record elsewhere. |
| `volumes` (config) | `./mediamtx.yml:/mediamtx.yml` | The repo's config file is the container's config file; edit it locally, restart the container to apply |

### Published ports

All ports are published on **all host interfaces** (no host-IP binding in `compose.yml`).

| Port | Protocol | Purpose |
| --- | --- | --- |
| `1935` | RTMP | **Ingest.** The GoStream publishes here (`rtmp://<pc-ip>:1935/live`) |
| `8554` | RTSP | Stream playback / redistribution |
| `8888` | HLS (TCP) | Stream playback in browsers and players; enabled in `mediamtx.yml` |
| `8889` | WebRTC | Browser-based playback page |
| `8890` | SRT | Additional ingest/playback protocol (published without a protocol suffix, i.e. as TCP — see [Troubleshooting](#troubleshooting) if an SRT client can't connect, since MediaMTX serves SRT over UDP) |
| `9997` | MediaMTX API (TCP) | Server API for status/automation; listener address set in `mediamtx.yml` |

### `mediamtx.yml` — server

The file in full, annotated:

```yaml
hls: yes                 # enable the HLS listener
hlsAddress: :8888        # …on port 8888

api:                     # API section present; no explicit value is set in the file
apiAddress: :9997        # API listener on port 9997

paths:
  all:                            # the rule below applies to every path
    record: yes                   # record everything that is published
    recordPath: "/recordings/%path/livestream-%d-%m-%Y_%H-%M-%s"
    recordFormat: fmp4            # fragmented MP4
```

| Key | Value in this repo | Meaning |
| --- | --- | --- |
| `hls` | `yes` | HLS serving is on |
| `hlsAddress` | `:8888` | HLS listen address/port |
| `api` | *(no value set)* | The key is present without an explicit `yes`; if you depend on the API, set `api: yes` explicitly so its state doesn't rest on a default |
| `apiAddress` | `:9997` | API listen address/port |
| `paths.all.record` | `yes` | Every published path is recorded — no per-path setup |
| `paths.all.recordPath` | `/recordings/%path/livestream-%d-%m-%Y_%H-%M-%s` | Where recordings go inside the container, and how they're named (`%path` + start date/time stamp) |
| `paths.all.recordFormat` | `fmp4` | Fragmented MP4 — resilient if a stream dies mid-record, and playable in standard tooling |

Everything not listed here (RTMP/RTSP/WebRTC/SRT listener settings, timeouts, logging, authentication, and the rest of MediaMTX's configuration) runs at MediaMTX's built-in defaults. The upstream [MediaMTX documentation](https://mediamtx.org) is the reference for any key you add.

### Where recordings land — path anatomy

```text
%USERPROFILE%\Videos\Livestream\  <stream key>  \  livestream-<date>_<time>.mp4
└──────────────┬──────────────┘  └──────┬─────┘  └─────────────┬─────────────┘
   compose.yml recordings volume   GoStream        recordPath template
   (mounted at /recordings/live)   stream key      (%path + date/time stamp)
```

Worked example: PC user `media`, stream key `my_event` → the file is written to `C:\Users\media\Videos\Livestream\my_event\livestream-….mp4`.

---

## Operations

| Task | Command |
| --- | --- |
| Start (detached) | `docker compose up -d` |
| Stop and remove the container | `docker compose down` |
| Restart after a config edit | `docker compose restart` (config is read at container start) |
| Status | `docker compose ps` |
| Follow logs | `docker logs -f mediamtx` — publish connects/disconnects and recording activity show here |
| Update the server image | `docker compose pull && docker compose up -d` |

---

## Security Model

Be clear-eyed about what this configuration is and isn't:

- **There is no authentication.** `mediamtx.yml` configures no credentials, so **anyone on the network who can reach port 1935 can publish** (and be recorded), and anyone who can reach the playback ports can watch any live path. That is a deliberate trade-off for a trusted LAN — a church media PC and its switcher — and it is the reason this server must stay on a trusted network.
- **All six ports bind on all interfaces.** Docker publishes them on every host address, LAN included. Windows Firewall prompts/behavior on first run decide who else can reach them.
- **The API port (9997) is published too.** MediaMTX's API can inspect and control the server; on a shared network, treat its exposure like the publish port's.
- **Never port-forward this server to the internet** as configured. If remote ingest is ever needed, put it behind a VPN or add MediaMTX authentication (`authMethods`, per-path users) in `mediamtx.yml` first — the upstream documentation covers both.
- **No secrets live in this repository** — there are no stream keys, passwords, or tokens in either file, and none are needed for this setup to work.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| GoStream won't connect / stream never starts | Wrong PC IP in the stream XML, PC and switcher on different networks, or the container isn't running | Re-check the IP with `ipconfig` (it changes if the PC gets a new DHCP lease — a static/reserved IP prevents this), confirm both devices are on the same LAN, and check `docker compose ps` + `docker logs mediamtx` |
| Stream connects but **no file appears** | Recording writes to the mounted volume; a bad host path or missing `USERPROFILE` in the shell that started Compose breaks the mount | Check `docker logs mediamtx` for recording errors; confirm `Videos\Livestream` exists on the host and the volume line in `compose.yml` resolves to it |
| Recording is in an unexpected folder | The stream key *is* the folder | The file is under `Videos\Livestream\<the exact stream key used>` — check for typos in the key configured on the GoStream |
| A player chokes on the recording | The format is **fragmented** MP4 (`fmp4`), which a few older tools dislike | Play with a current VLC/ffmpeg-based tool, or remux losslessly: `ffmpeg -i in.mp4 -c copy out.mp4` |
| `docker compose up` fails with "port is already allocated" | Something else on the PC already uses one of 1935 / 8554 / 8888 / 8889 / 8890 / 9997 | Find and stop the conflicting program, or remap that port's host side in `compose.yml` (and update the GoStream URL if you move 1935) |
| Config change has no effect | MediaMTX reads `mediamtx.yml` at container start | `docker compose restart` after every edit |
| SRT client can't connect on 8890 | `compose.yml` publishes `8890` without a protocol suffix, so Docker maps TCP; MediaMTX serves SRT over UDP | Change the mapping to `"8890:8890/udp"` and `docker compose up -d` |
| Windows Firewall blocks the GoStream | Docker Desktop / the published ports were not allowed through the firewall on the LAN profile | Allow Docker Desktop through Windows Firewall for your network profile, keeping the network itself trusted/private |
| API calls to `:9997` fail | The `api:` key in `mediamtx.yml` carries no explicit value | Set `api: yes` in `mediamtx.yml` and restart the container |

---

## Maintenance

- **Updating MediaMTX:** the image is unpinned (`bluenviron/mediamtx` with no tag), so `docker compose pull && docker compose up -d` upgrades to the current release. MediaMTX occasionally renames configuration keys between releases — after an upgrade, skim `docker logs mediamtx` for config warnings and compare against the upstream release notes. If you need repeatability over novelty, pin a tag in `compose.yml` (e.g. `bluenviron/mediamtx:<version>`) and bump it deliberately.
- **Disk usage:** recordings accumulate under `Videos\Livestream\` and nothing in this repo rotates or deletes them. Archive or prune old event folders as part of your routine; fMP4 files are as large as the stream's bitrate × duration.
- **Config changes:** edit `mediamtx.yml`, restart, and watch the logs once — a malformed YAML key will surface there immediately.
- **Backups:** the repo *is* the backup — the entire server rebuilds from these two files plus the recordings folder, which you back up like any other media.

---

## Known Limitations

- **Windows-oriented.** The recordings volume uses `${USERPROFILE}`, so out of the box this targets a Windows PC running Docker Desktop. On macOS/Linux, replace the left side of that volume with an absolute host path.
- **No authentication or encryption** (see [Security Model](#security-model)) — trusted-LAN use only, as configured.
- **One PC is the recorder.** If the PC or Docker Desktop is down, nothing records; there is no failover and no buffering on the switcher side.
- **Recording follows the publisher.** MediaMTX records while a path is being published; if the stream drops and reconnects, expect separate files per publishing session.
- **No `LICENSE` file** ships in this repository.

---

## Credits

- [MediaMTX](https://mediamtx.org) by bluenviron — the media server doing all of the work; this repository is configuration for it, published as the `bluenviron/mediamtx` Docker image.
- [Osee GoStream](https://osee-tech.com) — the switcher whose native RTMP streaming this setup records.
- [Docker Compose](https://docs.docker.com/compose/) — the entire deployment mechanism.
- **Author:** ManiacBoy777 — built to record Crossings Church livestreams from the media PC without adding OBS to the recording path.

---

*Public configuration repository — personal project. Last verified end to end: 2026-11-06 (America/Phoenix) — repository audited in full at `main` commit `04068d5` (`compose.yml`, `mediamtx.yml`, prior README, and commit history read in full); every port, path, and setting documented above is taken from those files.*
