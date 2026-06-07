+++
title = 'How demoinfocs-golang Parses CS2 Demos'
date = 2026-06-07T12:30:00+08:00
draft = false
tags = ["counter-strike", "demo", "demoinfocs", "golang", "protobuf", "cs-tech"]
series = ["CS Demo Tech Notes"]
+++

Sources summarized:

- `docs/how-dem-parsing-works.html` (my research note, not an upstream demoinfocs-golang doc)
- `docs/game-events.md` (from the upstream repo)

## 1) File Shape

A CS2 demo starts with `PBDEMS2` then continues as a frame stream.

Each frame contains:

1. command varint
2. tick varint
3. payload-size varint
4. payload bytes

Payloads may be Snappy-compressed and protobuf-encoded.

## 2) Main Parse Loop

Core pattern:

1. read command/tick/size
2. read payload bytes
3. decompress if needed
4. `proto.Unmarshal` into message struct
5. enqueue/dispatch

Conceptual flow:

```text
ParseToEnd -> parseHeader -> loop(parseFrame) -> msgQueue -> dispatcher -> user handlers
```

{{< mermaid >}}
flowchart TD
	A[ParseToEnd] --> B[parseHeader]
	B --> C[parseFrame loop]
	C --> D[Read command tick size]
	D --> E[Read payload bytes]
	E --> F{Compressed?}
	F -- Yes --> G[Snappy decode]
	F -- No --> H[Keep payload]
	G --> I[proto.Unmarshal]
	H --> I
	I --> J[msgQueue]
	J --> K[Dispatcher]
	K --> L[User handlers]
{{< /mermaid >}}

## 3) Message Layers

Three layers are important:

| Layer | Examples | Purpose |
| --- | --- | --- |
| DEM commands | `DEM_Packet`, `DEM_SendTables`, `DEM_ClassInfo` | Top-level frame semantics |
| NET/SVC messages | `svc_PacketEntities`, `svc_CreateStringTable` | Server->client state updates |
| Game events | `player_death`, `bomb_planted`, `round_end` | High-level gameplay events |

A `DEM_Packet` contains nested NET/SVC message stream.

## 4) SendTables and Entities

Entity system summary:

1. `DEM_SendTables` defines class schemas.
2. `DEM_ClassInfo` maps class IDs to names.
3. `svc_PacketEntities` applies deltas repeatedly.

This is the most complex part due to compact bit-level and field-path encoding.

## 5) String Tables

Key string tables:

| Table | Purpose |
| --- | --- |
| `userinfo` | SteamID/name/bot/HLTV info |
| `instancebaseline` | Default values for new entities |
| `modelprecache` | Model paths used for differentiation |

## 6) Game Events

Game events use a two-step model:

1. `GameEventList`: id -> descriptor (name + field types)
2. `GameEvent`: event id + values for that event

The parser resolves descriptor, decodes fields, emits typed events, and also emits a generic event.

## 7) GOTV vs POV Event Availability

`docs/game-events.md` provides a matrix per event name.

Examples:

| Event | GOTV | POV |
| --- | --- | --- |
| `player_death` | yes | yes |
| `round_end` | yes | yes |
| `bomb_planted` | yes | yes |
| `buytime_ended` | yes | no |
| `ammo_pickup` | no | yes |

Note from upstream docs: some demos can miss expected events. In those cases, parsing property updates can be a fallback strategy.

## 8) Helpful File Map (from docs)

- `pkg/demoinfocs/parsing.go`
- `pkg/demoinfocs/parser.go`
- `pkg/demoinfocs/s2_commands.go`
- `pkg/demoinfocs/game_events.go`
- `pkg/demoinfocs/stringtables.go`
- `pkg/demoinfocs/net_messages.go`
- `pkg/demoinfocs/sendtables/sendtablescs2/`

## Related

- Source repo: [markus-wa/demoinfocs-golang](https://github.com/markus-wa/demoinfocs-golang)
