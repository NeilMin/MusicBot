# Bilibili Audio Source — Design

**Date:** 2026-08-17
**Status:** Approved
**Branch:** `feature/bilibili-audio-source`

## Goal

Let JMusicBot play audio from Bilibili (哔哩哔哩) videos, alongside the sources it
already supports. A user pastes a Bilibili link into `/play` and the bot queues the
video's audio track.

## Constraints

These come from how the bot is deployed and are not negotiable:

1. **Everything ships inside the shaded jar.** No yt-dlp, no ffmpeg, no external
   binary on the host. The bot is deployed as `JMusicBot-*-All.jar`.
2. **Cross-platform.** Windows, macOS, and Linux ARM64 (Raspberry Pi under Docker)
   must all work. This rules out anything platform-specific.
3. **No new native libraries**, so the Docker jlink module list stays as it is.
4. **Anonymous only.** No login, no SESSDATA cookie, no credential storage.

## Verified Findings

Probed against the live API on 2026-08-17. These measurements drive the design:

| Question | Finding |
|---|---|
| `/x/player/playurl` unsigned | Works. Returns audio 30216 / 30232 / **30280 (192K)** without login. |
| `/x/player/wbi/playurl` + WBI | Works. The endpoint Bilibili's own web player uses. |
| Audio container | `ftyp iso5…dash` fragmented MP4, **AAC 44.1 kHz stereo**. |
| Range requests | HTTP **206** supported → seeking works. |
| CDN auth on `*.bilivideo.com` | **403** without `Referer`. **403** without a browser `User-Agent`. Both required. |
| Search, anonymous | `/x/web-interface/wbi/search/type` works with WBI signing, no login. |
| `baseUrl` lifetime | ~120 minutes. |

The CDN auth finding is why this needs a real source manager: the existing
`HttpAudioSourceManager` sends neither header and gets a 403.

The container finding is why no transcoding is needed: fMP4/AAC is the same shape as
YouTube's adaptive formats, which lavaplayer's `MpegAudioTrack` already decodes.

## Architecture

New package `com.jagrosh.jmusicbot.audio.bilibili`. Each class has one job and can be
tested without touching the others.

| Class | Responsibility | Depends on |
|---|---|---|
| `BilibiliLink` | Parse user input into a request descriptor. Pure function, no I/O. | — |
| `WbiSigner` | Derive the mixin key, compute `w_rid`. Caches keys ~12h. | `MessageDigest` |
| `BilibiliApiClient` | Typed wrapper over the Bilibili endpoints. | `HttpInterface`, Jackson |
| `BilibiliAudioSourceManager` | Implements `AudioSourceManager`. Owns the header-configured `HttpInterfaceManager`, dispatches `loadItem`. | the three above |
| `BilibiliAudioTrack` | Extends `DelegatedAudioTrack`. Resolves the stream at playback time. | lavaplayer |

`BilibiliLink` and `WbiSigner` are pure and deterministic, so they are unit-testable
with no network. `BilibiliApiClient` is tested against captured real responses.

### Dependencies

Zero new Maven dependencies:

- **Jackson** (`jackson-databind`) — already a runtime dependency, used for JSON.
- **Apache HttpClient** — arrives with lavaplayer, used via `HttpInterface`.
- **MD5** — `java.security.MessageDigest`, in `java.base`. No jlink module change.

## Input Handling

| Input | Behavior |
|---|---|
| `bilibili.com/video/BV...` | Single track, page 1 |
| `bilibili.com/video/av...` | Single track, page 1 |
| `...?p=N` | Single track, page N |
| `b23.tv/...` short link | Follow redirect, then as above |
| Bare `BV` id | Single track, page 1 |
| `bilisearch:<query>` | Search results as a selectable list |

**Multi-page (分P) videos always resolve to exactly one track.** Without `?p`, that is
P1. With `?p=N`, it is page N. A multi-page video is never expanded into a queue of
every page — that would flood the queue with content the user did not ask for.

Page N maps to `pages[N-1].cid` from the `view` response. An out-of-range `p` falls
back to P1 rather than failing.

## Playback Path

The central decision: **stream URLs are resolved lazily, at playback time.**

**At load time** (`loadItem`), only `/x/web-interface/view` is called, for title,
uploader, duration, thumbnail, and the page list. This is cheap and returns nothing
that expires.

**At playback time** (`BilibiliAudioTrack.process`), `/x/player/wbi/playurl` is called,
the highest-bandwidth entry in `data.dash.audio` is selected, and a
`PersistentHttpStream` (carrying `Referer` and browser `User-Agent`) is handed to
`MpegAudioTrack`.

The split exists because `baseUrl` expires in about 120 minutes. Resolving at load time
would mean a track sitting in a long queue has a dead URL by the time it reaches the
front. Resolving late also lets seek and replay fetch a fresh URL.

**Fallback:** older videos return no `dash` object, only `durl` (progressive MP4). In
that case use `durl[0].url` through the same `MpegAudioTrack` path.

## Registration and Configuration

- `AudioSource.BILIBILI("bilibili", "Bilibili videos and search", 25, ...)` — priority
  25 places it among the platform-specific sources, well before the `HTTP` catch-all at
  100, so it claims Bilibili URLs first.
- `reference.conf` gains `bilibili = true` under `playback.audioSources`.
- `BiliSearchCmd extends SearchCmd` with `searchPrefix = "bilisearch:"`, mirroring
  `SCSearchCmd`, plus the v2 slash-command equivalent.

Because the source is toggled through the existing `audioSources` block, an operator who
does not want it sets `bilibili = false` and nothing else changes.

## Error Handling

| Condition | Behavior |
|---|---|
| API `code != 0` | `FriendlyException` carrying Bilibili's own message (e.g. 稿件不可见) |
| CDN 403 | Retry each `backupUrl`, then re-fetch `playurl` once for a fresh URL |
| Member-only / Hi-Res content | Explicit "requires 大会员" message, not a stack trace |
| WBI key fetch fails | Fall back to the unsigned `/x/player/playurl`, which is verified working |
| Region-locked video | Report the restriction plainly |

The WBI fallback matters: it means a change to the WBI scheme degrades the feature to a
still-working path instead of breaking playback outright.

## Testing

All unit tests run offline.

- `BilibiliLink` — a table of URL forms mapped to expected descriptors, including
  malformed input and out-of-range `?p`.
- `WbiSigner` — deterministic assertions using fixed `img_key`/`sub_key` values, so the
  mixin permutation and MD5 are pinned.
- `BilibiliApiClient` — parses captured real API responses stored as fixtures, covering
  the DASH case, the `durl`-only case, and error codes.
- `AudioSourceUnitTest` — extend the platform-priority group to include `BILIBILI`.
- Config tests — the `bilibili` toggle enables and disables the source.
- One integration test hitting the live API, disabled by default so CI stays offline
  and deterministic.

## Out of Scope

Deliberately excluded from this version:

- Bangumi (`ep`/`ss`) links — different endpoints, and most content needs 大会员.
- Login / SESSDATA — anonymous access already yields 192K audio.
- Favorites (收藏夹) and collections (合集) as playlists.
- Hi-Res (30251) and Dolby audio — both require membership.
