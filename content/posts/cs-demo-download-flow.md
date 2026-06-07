+++
title = 'How CS Demo Manager Downloads Steam CS Demos'
date = 2026-06-07T12:10:00+08:00
draft = false
tags = ["counter-strike", "demo", "steam", "protobuf", "electron", "cs-tech"]
series = ["CS Demo Tech Notes"]
+++

This post explains the end-to-end demo download pipeline used by CS Demo Manager, from Steam GC requests to local `.dem` files.

## Overview

CS Demo Manager does not fetch demos from a public REST endpoint. Instead, it uses Steam client + Game Coordinator flow to obtain temporary demo URLs.

High-level path:

1. UI triggers download.
2. Node server invokes `boiler-writter` (C++ helper).
3. Helper talks to Steam/GC and returns protobuf match data.
4. App decodes matches, validates URL expiry, and enqueues jobs.
5. Queue downloads archive from Valve CDN and decompresses to `.dem`.
6. App writes matching `.dem.info` sidecar and imports metadata.

{{< mermaid >}}
sequenceDiagram
   actor User
   participant UI
   participant Server as Node server
   participant BW as boiler-writter
   participant Steam as Steam client
   participant GC as Steam GC
   participant CDN as Valve CDN
   participant DB as Local DB

   User->>UI: Trigger demo download
   UI->>Server: Request recent demos
   Server->>BW: execFile()
   BW->>Steam: Use Steam SDK session
   Steam->>GC: Request recent match data
   GC-->>BW: Match protobuf response
   BW-->>Server: Return/write protobuf data
   Server->>Server: Decode and enrich match data
   Server->>CDN: Validate demo URLs (HEAD)
   CDN-->>Server: URL status
   Server->>Server: Enqueue valid downloads
   Server->>CDN: Download demo archive
   CDN-->>Server: Compressed demo bytes
   Server->>Server: Decompress and write .dem
   Server->>Server: Write .dem.info
   Server->>DB: Import metadata and analysis
   Server-->>UI: Report completion/progress
{{< /mermaid >}}

## Detailed Flow

1. Trigger
   - User clicks download, or startup auto-download runs.

2. Fetch matches from GC
   - Server calls `fetchLastValveMatches()`.
   - This executes `boiler-writter` via `execFile()`.

3. GC response
   - Steam GC returns protobuf payload (`CMsgGCCStrike15_v2_MatchList` family).
   - Payload includes match IDs and temporary demo URLs.

4. Decode and enrich
   - App decodes protobuf with `csgo-protobuf`.
   - Optional Steam Web API lookup adds player names/avatars.

5. Expiry check
   - App sends `HEAD` to each URL.
   - Non-`200` links are treated as expired.

6. Queue
   - Valid, not-yet-downloaded matches enter `DownloadDemoQueue`.
   - Processing is sequential.

7. Download + decompress
   - App performs HTTP `GET` from Valve CDN.
   - Supported archive formats:

| Extension | Decompression |
| --- | --- |
| `.gz` | `zlib.createGunzip()` |
| `.bz2` | `unbzip2-stream` |
| `.zip` | `unzipper` |

8. Finalize
   - Write `.dem` file.
   - Write `.dem.info` raw protobuf sidecar.
   - Parse/import into local DB.

## Components

| Component | Role |
| --- | --- |
| `boiler-writter` | Steam SDK bridge; requests data from GC |
| Steam Game Coordinator | Source of match metadata + demo URLs |
| `csgo-protobuf` | Protobuf decode/encode in Node |
| `csgo-sharecode` | Share-code decode/encode |
| `DownloadDemoQueue` | Sequential download manager |
| `.dem.info` | Metadata sidecar used by CS + app |

## Requirements and Limits

- Steam must be running and logged in.
- Counter-Strike should not be running when the fetch starts.
- Demo URLs expire after a limited time.
- Recent-match window is limited (commonly around the latest matches only).
- Steam API key is optional for profile enrichment, not required for core download.

## Related

- Source repo: [akiver/cs-demo-manager](https://github.com/akiver/cs-demo-manager)
