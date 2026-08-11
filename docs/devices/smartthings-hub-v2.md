# SmartThings Hub v2

> **Status**: Local discovery documented; local control and setup rejected
> **Protocol**: WiFi (mDNS + SmartThings/Edge/Matter local services)
> **Manufacturer**: Samsung SmartThings
> **Manufacturer Status**: Active

## Overview

SmartThings Hub v2 advertises local SmartThings, Edge driver, and Matter
controller services. The primary stable identity is TXT `id` from
`_smartthings._tcp`. The hub is a Matter **controller only, not a bridge** — its
attached Zigbee/Z-Wave devices are not exposed to other Matter fabrics.

Discovery is local and works today. **Local control and local onboarding do not**:
onboarding is a cloud-account claim, and there is no documented local device-control
API. Full verdict, and the (device-level, Matter-only) multi-admin escape hatch, in
the research note `research-notes/smartthings-hub-local.md`.

## Discovery

Browse `_smartthings._tcp.local.` first. `_smartthings-hedge._tcp.local.` on
port 8766 exposes Edge driver WebSocket features (unverified), and
`_matter._tcp.local.` advertises the hub's Matter operational (controller)
node endpoint (unverified; ephemeral port).

## Local Services

| Service | Port | TXT | Description |
|---|---:|---|---|
| `_smartthings._tcp.local.` | 8081 | `type=hubv2`, `id=...` | Primary hub service |
| `_smartthings-hedge._tcp.local.` | 8766 | `feat=ctrl` | Edge driver WebSocket |
| `_matter._tcp.local.` | 49722 | `T=6` | Matter controller endpoint (not a bridge) |

Observed hostname: `hubv2-0d052a8a662bc0001.local`

Machine-readable spec: `device-specs/devices/smartthings-hub-v2.yaml`

