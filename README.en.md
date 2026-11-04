# fnmedia-ext Turbo

**中文: [README.md](README.md)**

> **Optimized build of the fnOS media extension** — based on upstream
> [**yujian0815/fnmedia-ext**](https://github.com/yujian0815/fnmedia-ext) **v0.2.68** (full build),
> keeping all upstream features and only tuning the playback path.

[![base](https://img.shields.io/badge/based%20on-fnmedia--ext%20v0.2.68-blue)](#which-upstream-version-is-this-based-on)
[![license](https://img.shields.io/badge/license-MIT-green)](LICENSE)

## Thanks first

99% of the work belongs to **yujian0815** and the upstream project
[fnmedia-ext](https://github.com/yujian0815/fnmedia-ext) (MIT). This repo only tweaks a few
places in the playback path that slow down start-up, and measures the result.

## Which upstream version is this based on

| Item | Value |
|---|---|
| Upstream base | **v0.2.68** (latest, full build) |
| This release | **v2.0.0** |
| trim.media | 0.9.8-1 (tested) |
| fnOS | ≥ 1.2.0 (tested on 1.2.0800) |

### Why v2.0.0 instead of v1.0.0

v1.0.0 was based on v0.2.39. Upstream later rewrote the line-selection stack in v0.2.68
(new per-title `linepredict` prediction, **global `speedbench` ranking removed**, racing changed to
"wait for every candidate", first `main` waits for the real playlist). v2.0.0 re-applies **all**
optimizations of this repo on top of v0.2.68 instead of simply stacking them:

| Area | v1.0.0 (0.2.39) | v2.0.0 (0.2.68) |
|---|---|---|
| Cold-start line pick | global source speed ranking | **follow upstream**: per-title `linepredict` |
| In-play correction | measured throughput (EMA) | kept, complementary to linepredict |
| Race policy | first success wins + budget | kept as default, **switchable back to upstream "wait all"** |
| Posters | direct fetch each time | **disk cache + background warm-up** |
| First `main` wait | 2.4 s then placeholder | 4 s then placeholder (`main_cms_budget`, configurable) |

## Measured results (same NAS, same title, same moment)

| Metric | Upstream v0.2.68 | Turbo v1.0.0 | **Turbo v2.0.0** |
|---|---|---|---|
| Multi-line menu (preset) | 0.07 s | 0.01 s | **0.01 – 0.07 s** |
| Playlist build (`main`, healthy line) | 2.1 s | 1.1 – 2.4 s | **0.17 – 1.9 s** (0.01 s when prewarmed) |
| Worst case `main` (bad source) | **8.0 s** | 2.4 s | **4.0 s** (configurable) |
| **First segment TTFB** | **1.9 – 2.8 s (= whole segment)** | 0.18 – 0.62 s | **0.33 – 0.85 s** |
| **Seek TTFB (6th segment)** | **3.9 – 4.7 s** | 0.20 – 0.37 s | **0.21 – 0.65 s** |

Compared with upstream v0.2.68: first byte 3–5× faster, seeking 5–10× faster (upstream still buffers
the whole segment; this build streams it), worst-case wait 4 s instead of 8 s.
Compared with v1.0.0: roughly on par — faster when playlists are prewarmed, slightly later
placeholder on bad sources (tunable via `main_cms_budget=2.4`).

## What is changed (9 items)

1. **Streaming segment relay** — upstream buffers the whole segment before replying; this build
   streams from the first upstream byte (falls back to buffering for 4xx/5xx, PNG-wrapped
   segments and `content-encoding` responses).
2. **Race: first success wins + shared 2 s budget** — cancels the rest as soon as one line is
   probed OK; `line_fastest_firstwin=false` restores the upstream "wait for all" policy.
3. **Known-good line single-shot** — lines with measured throughput ≥ 200 KB/s are probed alone.
4. **Measured-throughput ordering (EMA)** — real download speed is recorded while streaming
   (zero extra traffic) and used as a secondary sort key next to upstream `linepredict`.
5. **Poster disk cache + warm-up** — 7-day TTL, first pass 20 s after start, then every 30 min.
6. **`main_cms_budget`** (default 4 s) — bound the first `main` wait for normal CMS lines.
7. **In-process DNS cache** — `socket.getaddrinfo` with 600 s TTL / 2048 entries.
8. **Takeover wait + systemd hardening** — `FNMEDIA_STAGE_WAIT` (default 180 s) and
   `ExecStartPre` waiting for the venv.
9. **Chart/hot card acceleration** — two-phase resolve, detail wait 0.12 s → 2.5 s, play wait cap
   12 s, background pre-resolve of the first 6 unbound cards.

## Install

App Center → **Manual install** → pick `fnmedia-ext-turbo-2.0.0-public.fpk` → accept the disclaimer.

- Requires network during install (creates a venv).
- Same app id as upstream (`fnmedia-ext`), so installing over it is treated as an upgrade and
  `state/` is backed up and restored.
- Back up `/vol1/@appcenter/fnmedia-ext/repo/state/` before uninstalling.

## Troubleshooting knobs (App → Playback lines)

| Key | Default | Meaning |
|---|---|---|
| `line_fastest_firstwin` | `true` | first success wins; set `false` for upstream behavior |
| `line_fastest_budget` | `2.0` | total race budget (seconds) |
| `main_cms_budget` | `4.0` | first `main` wait bound (2–6 s) |
| `linepredict_enabled` | `true` | per-title poster-time prediction (upstream 0.2.68) |

## Disclaimer

- This repository does **not** ship, bundle or distribute any media content, source-list or play link.
- Community build — **not** published by the upstream author. Upstream issues go to the
  [upstream repo](https://github.com/yujian0815/fnmedia-ext).
- Releases here are the **public build** (upstream private/proprietary modules stripped).
- MIT License, upstream copyright retained.
