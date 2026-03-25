# Duparr

**Self-hosted music library deduplication for Navidrome, Symfonium, and any local music collection.**

Duparr scans your library, finds duplicate tracks, scores them by quality, and lets you review and move them through a web UI — nothing is ever deleted.

![Dashboard](https://img.shields.io/badge/status-v1.0.0-green) ![Docker](https://img.shields.io/badge/docker-self--hosted-blue) ![License](https://img.shields.io/badge/license-MIT-lightgrey)

---

## Features

- **Never deletes** — duplicates are moved to a `/duplicates` folder, fully reversible from the UI
- **Dual detection** — metadata matching (fast) with optional Chromaprint audio fingerprinting (accurate)
- **Incremental scanning** — only processes new or changed files on subsequent scans
- **Smart scoring** — keeps the highest quality copy: FLAC > AAC/OGG > MP3, bitrate, tags, artwork
- **Variant detection** — preserves remixes, live versions, acoustic versions, and radio edits
- **Same-album guard** — won't mark CD1/CD2 or vinyl sides as duplicates
- **Cover art fetcher** — finds and embeds missing album art from MusicBrainz and iTunes
- **Full undo** — restore any moved file to its original location from the UI
- **Progress bar** — live scan and dedup progress with log streaming
- **Optional auth** — basic auth via env var, off by default for LAN use

---

## Screenshots

> Dashboard showing duplicate groups with confidence scores, keeper/dupe labelling, and space savings.

---

## Quick Start

### 1. Clone

```bash
git clone https://github.com/mortaljinx/duparr.git
cd duparr
```

### 2. Configure `docker-compose.yml`

Edit the volume paths to match your setup:

```yaml
volumes:
  - /your/music/library:/music-rw
  - /your/duplicates/folder:/duplicates
  - /your/docker/duparr/ui:/ui
  - duparr-data:/data
```

### 3. Build and run

```bash
docker compose build
docker compose up -d
```

### 4. Open the UI

```
http://your-server-ip:5045
```

Click **SCAN + FIND** to index your library and detect duplicates.

---

## How It Works

```
Scan → Index → Group → Score → Review → Move → Undo
```

1. **Scan** — walks your music directory, reads tags and mtime, stores in SQLite. Incremental — only changed files are re-read.
2. **Find Dupes** — groups tracks by `(primary_artist, clean_title)`. Variants (remixes, live, acoustic) are detected and preserved. Same-album tracks require fingerprint confirmation.
3. **Score** — each track is scored, highest score is the keeper.
4. **Review** — UI shows every group with keeper highlighted, confidence badge, space saved, and reason.
5. **Apply** — moves duplicate files to `/duplicates` preserving relative paths. DB updated only after successful move.
6. **Undo** — restore any file from the Undo page.

---

## Scoring System

| Condition | Points |
|---|---|
| FLAC format | +100 |
| AAC / OGG / Opus | +50 |
| MP3 | +30 |
| Bitrate (proportional, capped) | +0 to +50 |
| Complete tags | +10 |
| Has embedded artwork | +5 |
| Filesize tiebreaker | +0 to +10 |
| `_collections` folder | −100 |
| Compilation album tag | −40 |

---

## Configuration

All configuration is via environment variables in `docker-compose.yml`.

| Variable | Default | Description |
|---|---|---|
| `MUSIC_DIR` | `/music-rw` | Music root inside container |
| `DUP_DIR` | `/duplicates` | Where duplicates are moved |
| `DB_PATH` | `/data/music.db` | SQLite database path |
| `USE_FINGERPRINT` | `false` | Enable Chromaprint audio fingerprinting |
| `FPCALC_PATH` | `/usr/bin/fpcalc` | Path to fpcalc binary |
| `EXCLUDE_DIRS` | _(empty)_ | Comma-separated dirs to skip (e.g. `/music-rw/mnt`) |
| `MIN_SCORE_GAP` | `0` | Minimum score difference before a track is flagged |
| `COVER_SOURCES` | `musicbrainz,itunes` | Cover art sources, in priority order |
| `COVER_MIN_SIZE` | `300` | Minimum cover dimension in pixels |
| `COVER_EMBED` | `true` | Embed cover art into audio files |
| `COVER_SAVE_FILE` | `true` | Save `cover.jpg` alongside audio files |
| `DUPARR_PASSWORD` | _(empty)_ | Enable basic auth — leave empty to disable |
| `NAVIDROME_URL` | _(empty)_ | Auto-trigger Navidrome rescan after Apply |
| `NAVIDROME_USER` | _(empty)_ | Navidrome username |
| `NAVIDROME_PASSWORD` | _(empty)_ | Navidrome password |

---

## Authentication

Auth is **off by default** — suitable for trusted LAN use.

To enable, add to your `docker-compose.yml`:

```yaml
environment:
  DUPARR_PASSWORD: "your-password-here"
```

The UI will prompt for a password in the browser. The `/health` endpoint is always public for Docker healthchecks.

---

## Variant Detection

Duparr preserves tracks that are genuine variants — they appear in the UI as **VAR.** and are never moved.

Detected as variants:
- Bracket keywords in title: `(Remix)`, `[Live]`, `(Acoustic)`, `(Demo)`, `(Instrumental)` etc.
- Multi-word bracket phrases: `(Radio Edit)`, `(Extended Mix)`, `(Club Version)` etc.
- Dash-suffix patterns: `Song - Live`, `Song - Acoustic Mix`, `Song - Remaster`
- Multi-word phrases in file path: `live version`, `radio edit`, `extended mix` etc.

---

## Same-Album Guard

Tracks in the same folder, or sibling disc subfolders (`CD1`/`CD2`, `Disc 1`/`Disc 2`, `Side A`/`Side B`), are **not** flagged as duplicates based on metadata alone. Fingerprint match is required. This prevents false positives on multi-disc albums and box sets.

---

## Cover Art

Click **# COVERS** in the dashboard to fetch missing album art.

- Groups tracks by folder (one fetch per album)
- Queries MusicBrainz Cover Art Archive first, iTunes as fallback
- Embeds into MP3 (ID3), FLAC, and M4A/AAC files
- Saves `cover.jpg` alongside tracks for Navidrome/Symfonium

---

## Undo

Every move is logged. From the **UNDO** page you can:
- See all moved files with their original paths
- Restore individual files or all files at once
- See total space currently in `/duplicates`

---

## Docker Compose Example

```yaml
services:
  duparr:
    image: duparr-local
    build: .
    container_name: duparr
    restart: unless-stopped
    ports:
      - "5045:5045"
    volumes:
      - /mnt/nas/media/music:/music-rw
      - /mnt/nas/media/duplicates:/duplicates
      - /opt/docker/duparr/ui:/ui
      - duparr-data:/data
    environment:
      MUSIC_DIR: /music-rw
      DUP_DIR: /duplicates
      DB_PATH: /data/music.db
      USE_FINGERPRINT: "false"
      EXCLUDE_DIRS: /music-rw/mnt
      COVER_SOURCES: musicbrainz,itunes
      COVER_MIN_SIZE: "300"
      COVER_EMBED: "true"
      COVER_SAVE_FILE: "true"
      MIN_SCORE_GAP: "0"
      # DUPARR_PASSWORD: "changeme"
      # NAVIDROME_URL: "http://navidrome:4533"
      # NAVIDROME_USER: "admin"
      # NAVIDROME_PASSWORD: "changeme"

volumes:
  duparr-data:
```

---

## Supported Formats

MP3, FLAC, AAC (M4A), OGG, Opus, WMA, WAV, AIFF, APE, WavPack

---

## Requirements

- Docker + Docker Compose
- Music library mounted as a volume
- A separate `/duplicates` volume with write access

---

## Roadmap

- [ ] Smart keep rules — user-configurable format/folder preferences
- [ ] Auto-clean mode — cron-triggered apply for HIGH confidence groups only
- [ ] CLI mode — headless operation without the web UI
- [ ] Webhook support — notify external services after Apply
- [ ] Non-root container user

---

## License

MIT — do whatever you want with it.

---

## Notes

- `USE_FINGERPRINT=false` is recommended for large libraries (51k+ tracks). Metadata matching is fast and accurate for most collections. Enable fingerprinting for collections with lots of untagged or mis-tagged files.
- The `/duplicates` folder mirrors your library structure. If you decide to delete after reviewing, `rm -rf /duplicates` is safe.
- Duparr does not modify any file in your music library except to embed cover art (when `COVER_EMBED=true`).
