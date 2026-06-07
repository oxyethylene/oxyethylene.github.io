+++
title = 'The .dem.info Binary Format'
date = 2026-06-07T12:20:00+08:00
draft = false
tags = ["counter-strike", "demo", "protobuf", "binary-format", "cs-tech"]
series = ["CS Demo Tech Notes"]
+++

This post documents the `.dem.info` companion file stored next to Valve matchmaking demos.

## What is `.dem.info`

For a downloaded Valve demo, CS Demo Manager stores two files side by side:

- `match730_...dem`
- `match730_...dem.info`

The `.dem.info` file is a raw protobuf binary blob:

- no file header
- no JSON wrapper
- first byte is the first protobuf field tag

Top-level message: `CDataGCCStrike15_v2_MatchInfo`.

## Top-Level Message

`CDataGCCStrike15_v2_MatchInfo` fields used by the app:

| Field | Wire type | Name | Meaning |
| --- | --- | --- | --- |
| 1 | varint | `matchid` | Unique match id (`uint64`) |
| 2 | varint | `matchtime` | Unix start timestamp (`uint32`) |
| 3 | len-delim | `watchablematchinfo` | Nested message with `serverIp`, `tvPort` |
| 4 | len-delim | `roundstatsLegacy` | Old CS:GO single-message format |
| 5 | len-delim (repeated) | `roundstatsall` | Modern format, one entry per round |

Compatibility logic is effectively:

```text
roundstatsLegacy ?? last(roundstatsall)
```

## Nested Round Stats Message

`CMsgGCCStrike15_v2_MatchmakingServerRoundStats` commonly used fields:

| Field | Name | Notes |
| --- | --- | --- |
| 1 | `reservation` | Nested player/game-type container |
| 2 | `map` | Contains demo URL string in this context |
| 5-10 | `kills/deaths/assists/headshots/mvps/scores` | Packed cumulative arrays for 10 player slots |
| - | `teamScores` | Packed array with team score snapshot |
| - | `matchDuration` | Duration in seconds |
| - | `matchResult` | `0=tie, 1=CT-start won, 2=T-start won` |
| - | `reservationid` | Used in naming/share-code flow |

Inside `reservation`, important fields include:

- `accountIds` (10 slots)
- `gameType`
- optional tournament team names

## Old vs Modern Layout

### Old CS:GO

- field 4 (`roundstatsLegacy`) present
- field 5 (`roundstatsall`) absent
- usually tiny files

### Modern CS2 / late CS:GO

- field 4 absent
- field 5 repeated per round
- much larger files

## Protobuf Tag Formula

```text
tag = (field_number << 3) | wire_type
```

Examples:

- `0x08` -> field 1, varint (`matchid`)
- `0x10` -> field 2, varint (`matchtime`)
- `0x1a` -> field 3, length-delimited (`watchablematchinfo`)
- `0x22` -> field 4, length-delimited (`roundstatsLegacy`)
- `0x2a` -> field 5, length-delimited (`roundstatsall`)

## What CS Demo Manager Extracts

| Value | Source | Used for |
| --- | --- | --- |
| Match date | `matchtime` | Date display/sorting |
| Duration | `matchDuration` | Match length |
| Share-code parts | `matchid + reservationid + tvPort` | Share-code generation |
| Map | `gameType` decode | Map display |
| Player IDs | `reservation.accountIds` | Player lookup/linking |
| Round stats | `roundstatsall` deltas | Per-round metrics |
| Result | `teamScores + matchResult` | Win/loss/tie display |
| Demo URL | `map` string | Download target |

## Related

- Source repo: [akiver/cs-demo-manager](https://github.com/akiver/cs-demo-manager)
