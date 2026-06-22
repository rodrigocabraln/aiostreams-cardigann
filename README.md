# AIOStreams as a Prowlarr indexer (Cardigann)

`aiostreams.yml` turns your **AIOStreams** instance into a **Torznab** indexer inside
Prowlarr — no extra service to host. Prowlarr calls AIOStreams' `/stream/...json`
endpoint, parses the `streams[]`, and builds the magnets itself.

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

In `aiostreams.yml`, change **only the host** in `links` (scheme + host + port — no path):

```yaml
links:
  - http://10.1.1.51:3000        # <-- your AIOStreams host
```

That is the only edit the file needs. Everything else (UUID, config, seeders) is set in
the Prowlarr UI.

### 3. Restart Prowlarr

Cardigann **caches** definitions at startup; refreshing the UI is not enough:

```bash
docker restart prowlarr
```

### 4. Add the indexer in Prowlarr

1. **Indexers → Add Indexer →** search for **"AIOStreams"** (Custom section).
2. Fill in **UUID** and **Base64 Encoded Configuration** (see below).
3. (optional) tweak *Fallback seeders* or the validation IMDb ids.
4. **Test → Save**, then sync with Radarr/Sonarr (Settings → Apps).

### Where do UUID and Base64 config come from?

From your AIOStreams manifest URL:

```
http://10.1.1.51:3000/stremio/<UUID>/<BASE64_CONFIG>/manifest.json
                              └──┬──┘ └──────┬──────┘
                               UUID      Base64 config
```

Paste each part into its field (without `/manifest.json`, no stray slashes). The base64
blob must be URL-safe (no `/`); AIOStreams blobs are.

---

## What it does

- **ID-based search**: movies by `imdbid`, series by `imdbid` + season + episode — exactly
  how Radarr/Sonarr search automatically and from the interactive search.
- For each torrent stream it extracts **infohash, title, size, seeders and category**, and
  Prowlarr builds the magnet from the infohash.
- Reads AIOStreams' structured `streamData`, with fallbacks, so it captures essentially
  **every torrent** in the response (only non-torrent and duplicate entries are dropped).

## What it does NOT do

- **Torrents/magnets only.** Prowlarr and the *arrs only handle torrent (or usenet);
  direct **HTTP streaming** results have no infohash and are skipped.
- **No free-text search.** Stremio endpoints need an exact ID; Cardigann can't turn text
  into an ID. A raw text search in the Prowlarr UI (with no linked ID) just returns the
  validation title. This does not affect the *arrs, which always search by ID.
- **Movies/TV is inferred (~98–99%).** The type is guessed from the parsed season/episode
  and the title. A tiny fraction of odd files may be miscategorised.
- **No cached-only filter here.** Configure cached-only / dedup / sorting **inside
  AIOStreams** (see below).

---

## Recommended AIOStreams config

Point the indexer at an AIOStreams configuration that has **only torrent/debrid scrapers**
enabled and **no direct-streaming addons** (those return no torrents and are dropped
anyway). Set **cached-only**, deduplication, ranking and formatting **inside AIOStreams** —
this indexer simply transcribes whatever AIOStreams returns. The AIOStreams URL is just a
config blob, so you can keep one config for normal Stremio use and a separate
torrent-only one for this indexer.

It does **not** depend on your AIOStreams output/format settings: it reads the structured
`streamData`, not the display text, so changing emojis, name templates or sorting won't
break parsing.

---

## Troubleshooting

- **"no results in the configured categories" (Sonarr/Radarr test):** make sure you
  **restarted Prowlarr** after copying the file. If a manual search in Prowlarr returns
  results but the app test doesn't, confirm series come out tagged **TV**.

- **URLs with `%2F` / tiny (~394-byte) responses:** the host in `links` is wrong, or you
  put the config path (with slashes) into a settings field. The host goes in `links`;
  UUID and config go in their own fields.

- **You see streaming results that aren't usable:** your AIOStreams config includes direct
  HTTP streaming addons. Use a torrent-only config (above).

- **"No files found are eligible for import" in Radarr/Sonarr:** this is a **download-client**
  issue, not the indexer. Some debrid clients mishandle **single-file** torrents: they put
  the file in a sub-folder named without the extension but report the path *with* the
  extension, so *arr scans the wrong folder. Folder-style torrents import fine. The magnet
  this indexer returns is valid (the same any indexer would produce for that infohash) — fix
  it on the download-client side (update it, or use a mount/symlink-based setup).
