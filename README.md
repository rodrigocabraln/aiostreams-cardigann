# AIOStreams as a Prowlarr indexer (Cardigann)

`aiostreams.yml` turns your **AIOStreams** instance into a **Torznab** indexer inside
Prowlarr — no extra service to host. Prowlarr hits AIOStreams' `/stream/...json`
endpoint, parses `streams[]`, and builds the magnets itself.

```
Radarr/Sonarr → Prowlarr → (aiostreams.yml) → AIOStreams → torrents/debrid
```

---

## Setup

### 1. Copy the file

Copy `aiostreams.yml` into Prowlarr's **`Definitions/Custom/`** folder (create it if it
doesn't exist):

| Install | Path |
|---|---|
| Docker (linuxserver / hotio) | `/config/Definitions/Custom/aiostreams.yml` |
| Native Linux | `/var/lib/prowlarr/Definitions/Custom/aiostreams.yml` |
| Windows | `C:\ProgramData\Prowlarr\Definitions\Custom\aiostreams.yml` |

Rule of thumb: it sits next to `prowlarr.db`, inside `Definitions/Custom/`.

```bash
# Docker example
docker cp aiostreams.yml prowlarr:/config/Definitions/Custom/aiostreams.yml
```

### 2. Edit the host

In `aiostreams.yml`, change **only the host** in `links` (scheme + host + port, without
the config path):

```yaml
links:
  - http://10.1.1.51:3000        # <-- your AIOStreams host
```

### 3. Restart Prowlarr

Cardigann **caches** definitions at startup; refreshing isn't enough:

```bash
docker restart prowlarr
```

### 4. Add the indexer

1. **Indexers → Add Indexer →** search for **"AIOStreams"** (Custom section).
2. Fill in **UUID** and **Base64 Encoded Configuration** (see below).
3. (optional) tweak *fallback seeders* or the validation IMDb ids.
4. **Test → Save**, then sync with Radarr/Sonarr (Settings → Apps).

### Where do UUID and Base64 config come from?

From your AIOStreams manifest URL:

```
http://10.1.1.51:3000/stremio/<UUID>/<BASE64_CONFIG>/manifest.json
                              └──┬──┘ └──────┬──────┘
                               UUID      Base64 config
```

Paste each part into its field (without `/manifest.json`, no stray slashes).

---

## Recommended: a torrent-only AIOStreams config

Point the indexer at an AIOStreams config with **only torrent/debrid addons** (Comet,
MediaFusion, StremThru…) and **no** direct-streaming addons, which don't provide
torrents. Configure **cached-only**, dedup, ranking and formatting **inside AIOStreams**
(it has those options); this indexer just transcribes what AIOStreams already filtered.
The AIO URL is just a config blob, so you can keep one config for Stremio (with
everything) and a torrent-only one for this indexer.

---

## What it covers and what it doesn't

**Covers:**
- **ID-based search** (imdbid; series with season+ep) — exactly how Radarr/Sonarr search
  automatically and interactively.
- Extracts infohash, title, size and seeders, and builds the magnet (Prowlarr generates
  it from the infohash).

**Doesn't cover / limitations:**
- **Torrents/magnets only.** Prowlarr and the *arrs only handle torrent or usenet;
  **direct HTTP streaming** streams have no infohash and are dropped.
- **No free-text search.** Stremio endpoints require an exact ID, and Cardigann can't
  convert text→ID. A raw text search (in the Prowlarr UI, with no linked ID) returns the
  validation title — this is expected, and that capability can't be removed without
  breaking the connection test. It doesn't affect the *arrs, which always search by ID.
- **Movies/TV classification is inferred (~98–99% accurate).** The indexer doesn't know
  the type "out of the box" like a dedicated service; it infers it from the content
  (`parsedFile.seasons`/`episodes` and title patterns). A ~1–2% of odd files (no season
  anywhere, or where the parser mistakes "1999"/"720"/"x264" for an episode) may be
  misclassified.
- **cached-only** is configured inside AIOStreams, not per-indexer.
- Relies on AIOStreams returning `streamData` (forced via the `User-Agent: AIOStreams/...`
  header). The base64 blob must be URL-safe (AIOStreams blobs are).

---

## Troubleshooting

- **"no results in the configured categories"** (Sonarr/Radarr): make sure you
  **restarted Prowlarr** after copying the file. If a manual search in Prowlarr returns
  results but the app's test doesn't, check that series come out tagged TV.
- **URLs with `%2F` / ~394-byte responses**: the host in `links` is wrong, or you put the
  config path with slashes into a field. The base goes in `links` (host), UUID/config go
  in their fields.
- **You see streaming results that don't work**: your AIOStreams config includes direct
  HTTP streaming addons. Use a torrent-only config (see above).
