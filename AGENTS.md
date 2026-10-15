# AGENTS.md

Context for AI coding agents working on this repository. Read this before changing anything. `README.md` is the operator documentation; this file explains **why** the project is shaped the way it is, so that later changes don't undo deliberate decisions.

## What this project is

A multistream relay for a ZowieBox hardware encoder. The ZowieBox can push at most 2 simultaneous streams, so it pushes **one** RTMP stream to MediaMTX on a Raspberry Pi. MediaMTX then:

1. forwards it, unmodified, to YouTube and Facebook (native `forward`);
2. pushes it, unmodified, to Telegram (`runOnOnline` hook running ffmpeg with `-c copy`);
3. archives the audio track to a USB stick (`runOnAvailable` hook running ffmpeg with `-c copy`);

and a second container uploads finished archive files to Google Drive every 15 minutes (rclone run by supercronic).

## Target environment

- **Host:** Raspberry Pi 4 with **1 GB RAM**, Ubuntu (arm64), Docker Engine and the Compose plugin. Images must be published for `linux/arm64`.
- **Shared host:** the Pi is also a NetBird routing peer for its network. Don't add anything that changes host networking, firewalling or routing.
- **Network:** the LAN is `192.168.10.0/24`, and the ZowieBox publishes from it. Home upload bandwidth carries 3× the stream bitrate, so anything that uploads must stay rate-limited.
- **Storage:** the SD card holds the OS and must not receive archive writes. The USB stick is mounted at `/mnt/stream-archive` via `fstab` (by UUID, `nofail`). The `stream-audio/` directory exists **only on the stick**.
- **Repo location on the Pi:** `/opt/mediamtx-relay`.

## Files

| File | Role |
|---|---|
| `compose.yaml` | `mediamtx` (relay + archive recording) and `archive-sync` (scheduled upload) |
| `mediamtx.yml` | Ingest, auth, path `live/main`, Telegram and archive hooks. **No secrets** |
| `archive-sync/Dockerfile` | `rclone/rclone` + supercronic (version- and SHA-256-pinned, arm64 and amd64) |
| `archive-sync/crontab` | The one scheduled `rclone copy` |
| `.env.example` | Every variable `compose.yaml` needs, with placeholders |
| `.gitignore` | Ignores `.env` and `rclone/` (credentials) |
| `README.md` | Operator documentation: setup, operations, troubleshooting |

## Invariants: don't break these

1. **No secrets in tracked files.** Stream keys and the publish password live only in `.env`. They reach MediaMTX as `MTX_*` environment variables (`MTX_PATHDEFAULTS_FORWARD_<n>_DEST`, `MTX_AUTHINTERNALUSERS_0_PASS`). rclone credentials live in `rclone/rclone.conf`. Both are git-ignored.
2. **`live/main` is the only path in `mediamtx.yml`.** The forward destinations are set on `pathDefaults` and inherited by every defined path, so adding another path (or `all_others`) would forward that path to YouTube and Facebook too.
3. **The `zowie` user stays first in `authInternalUsers`.** Its password is injected by index (`..._0_PASS`).
4. **Telegram stays on its own ffmpeg hook (`runOnOnline`), separate from the archive hook (`runOnAvailable`).** Don't move Telegram back to `forward` without testing against the real Telegram ingest first (see Verified facts), and don't merge the two hooks into one ffmpeg process: a Telegram failure would then interrupt the archive.
5. **Nothing is re-encoded.** Forwarding is native, the Telegram hook uses `-c copy`, and the archive uses `-map 0:a -c copy`. Transcoding on a Pi 4 would be expensive and would change quality. Encoder settings belong on the ZowieBox.
6. **Never write archive data to the SD card.** Both containers rely on `stream-audio/` existing only on the stick: if it's missing, the hook and the upload fail and retry instead of writing elsewhere. Never create `stream-audio/` on the host path or in an entrypoint, and never use `create_host_path: true` for it. Both bind mounts use `propagation: rslave` so a stick mounted later is picked up.
7. **`rclone copy`, never `rclone sync` or `move`.** Losing or wiping local files must never delete the remote archive.
8. **Keep `RCLONE_MIN_AGE`** (currently `10m`). The file being recorded keeps changing its mtime; this is what stops partial files from being uploaded.
9. **Keep `RCLONE_BWLIMIT`** (currently `256k`, about 2 Mbit/s). Uploads must never compete with a live stream for upload bandwidth.
10. **Pin exact versions everywhere:** the MediaMTX image tag, the rclone image tag, and supercronic's version plus its checksums. No `latest`, no ranges.
11. **Explicit names.** Environment variables are spelled out (`YOUTUBE_STREAM_KEY`, not `YT_KEY`). The repo owner explicitly asked for this.

## Decisions and rejected alternatives

These were each considered and rejected for stated reasons. Don't reintroduce them without a new reason.

- **ffmpeg pusher containers or `ffmpeg -f tee` for forwarding → rejected.** MediaMTX has forwarded natively since **v1.20.0**, including to `rtmps://`, with one independent handler per destination that retries every 5 s. Shell loops around ffmpeg also had a broken PID 1 (no signal forwarding). Use ffmpeg only for what `forward` can't do.
- **A single ffmpeg `tee` → rejected.** With `onfail=ignore`, a failed destination is dropped until restart; with `abort`, one bad destination takes down all three.
- **Native `record: yes` for the archive → rejected.** It records every track, so video comes along at about 2.7 GB/h, and the default `recordDeleteAfter: 1d` deletes recordings. The archive is audio only.
- **ADTS `.aac` hourly segments** are deliberate: no index or trailer, so a power cut loses only the last few seconds. MP4 would need finalization. FLAC or WAV of a lossy source add size without quality.
- **datarhei Restreamer → rejected:** stale releases. **Community MediaMTX UIs → rejected:** small one-person projects.
- **A systemd timer on the host for uploads → replaced** by supercronic in a container, so the whole project deploys with `docker compose up -d`. Busybox `crond` (needs crontab and stdout glue) and Ofelia (needs `/var/run/docker.sock`, which is root-equivalent) were rejected.
- **Host networking for `mediamtx`** is recommended by MediaMTX. It preserves the ZowieBox's source IP for the `ips` check and lets the API bind to `127.0.0.1:9997`. `ips` accepts only IPs and CIDRs, not hostnames.
- **A Pi instead of a VPS:** the Pi is free and keeps ingest LAN-only. The trade-off is that the home connection uploads 3×.

## Verified facts (don't re-derive them from older docs)

- **MediaMTX v1.21.x:**
  - The hook is `runOnAvailable`/`runOnAvailableRestart`; `runOnReady` no longer exists.
  - Hook commands are executed **without a shell** (shell-quote split, `$VAR` expansion from the hook environment, then the process environment).
  - Restarted hooks pause 5 s. On stop, the process group receives SIGINT.
- **Forward destination format:** `rtmps://host/app#streamKey`. Everything before `#` becomes the RTMP app, and keys (the fragment) are stripped from MediaMTX's logs.
- **Telegram vs native forward (tested 2026-10-14, v1.21.2):** a `forward` destination to Telegram failed with `EOF` within a second on every attempt, with the correct URL and key. ffmpeg from inside the container, with the identical URL and key, streamed fine. The likely cause is a client-compatibility difference: MediaMTX's RTMP client (gortmplib) uses a plain handshake and minimal connect parameters, while ffmpeg uses a digest handshake. This hasn't been reported upstream. ffmpeg error lines print the full Telegram URL, key included.
- **MediaMTX env overrides:**
  - They work for lists by index (`..._FORWARD_0_DEST`).
  - Map keys in env var names (path names) are lowercased and can't contain `/` or `_`. So `MTX_PATHS_LIVE/MAIN_...` is impossible, which is why `pathDefaults` is used.
  - Env overrides are applied **before** path defaults are merged.
- **RTMP publish credentials** go in the query string: the stream key `main?user=zowie&pass=…`. Tested both as a single URL and as an app + key split, and confirmed working on the real ZowieBox.
- **Platform keys:**
  - YouTube: custom keys are reusable, and auto-start makes a broadcast **public** immediately.
  - Facebook: the persistent key must be used over RTMPS. Going live still requires "Go Live" in Live Producer.
  - Telegram: the key persists until someone revokes it. Going live still requires "Start Streaming" in the app. There's one livestream per channel.
- **Google Drive via rclone:**
  - Uses its own OAuth client ID; rclone's shared client ID is being retired in 2026.
  - The app is **published** to production, because grants in Testing mode expire after a week.
  - Scope is `drive.file`, so rclone only sees files it created. Changing the client ID hides existing uploads from rclone.
- **Audio:** ZowieBox encodes AAC-LC, 48 kHz, stereo, line-in, at 128 or 192 kbps. Facebook accepts up to 256 kbps. Telegram publishes no audio limits.

## How to validate changes

Use dummy values. Never put real keys in commands, commits or logs.

```bash
# compose.yaml interpolation (every required variable must resolve)
docker compose --env-file .env.example config --quiet && echo OK   # never overwrite a real .env

# mediamtx.yml (the env var stands in for the injected destinations)
docker run --rm -v ./mediamtx.yml:/mediamtx.yml:ro \
  -e MTX_PATHDEFAULTS_FORWARD_0_DEST='rtmps://a.rtmps.youtube.com/live2#dummy' \
  bluenviron/mediamtx:<MEDIAMTX_VERSION>-ffmpeg --validate-conf=/mediamtx.yml

# archive-sync image and crontab
docker compose build archive-sync
docker run --rm --entrypoint supercronic -v ./archive-sync/crontab:/etc/crontab:ro \
  mediamtx-relay/archive-sync:<RCLONE_VERSION> -test /etc/crontab
```

**Testing forwarding without real platforms:**
1. Run a second MediaMTX as a sink, e.g. RTMP on `:2935` with `paths: {all_others:}`.
2. Point the destinations at it, e.g. `rtmp://127.0.0.1:2935/live2#key`, with one destination deliberately dead.
3. Publish a test pattern with ffmpeg (`testsrc` + `sine`, `-g 30`).
4. Check `GET /v3/paths/forward-dests/list?path=live/main` for YouTube and Facebook: each destination's state is `forwarding`, `error` (with `lastError`) or `idle`. Telegram's hook appears on the sink as another publisher, e.g. `s/<key>`. Use a key without `:` for the sink, because MediaMTX rejects `:` in path names; real Telegram keys contain one.

Changes to platform settings, the ZowieBox, or Google Cloud can't be tested here. Describe them as manual steps for the operator.

## Working conventions

- **Smallest change:** prefer native features of the existing tools over adding components, scripts or glue.
- **Keep docs in sync:** when you change behaviour, update `README.md` (setup, operations, troubleshooting, design notes) and this file in the same change.
- **Variables:** when you add one, add it to `.env.example` with a comment, and reference it in `compose.yaml` with `${VAR:?message}` so a missing value fails loudly.
- **Upgrades:** before bumping a pinned version, read the release notes. MediaMTX has renamed config keys between releases, so run `--validate-conf` after every bump.

## Status

- **Confirmed on the Pi (2026-10-14):** `docker compose up -d --build` works; the ZowieBox publishes with the `main?user=…&pass=…` key; YouTube and Facebook receive the stream.
- **Not yet set up:** the rclone remote. `archive-sync` runs fail until `rclone/rclone.conf` exists.

## Open items

- **Telegram with the real stream:** the ffmpeg hook was tested against a local stand-in for Telegram, and plain ffmpeg was tested against the real Telegram ingest. The hook hadn't yet been confirmed against real Telegram with the ZowieBox stream, including 192 kbps audio.
- **Upstream report:** the Telegram incompatibility with MediaMTX's native forward could be reported to bluenviron/mediamtx. If it gets fixed, Telegram could move back to `forward`.
