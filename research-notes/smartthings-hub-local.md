# SmartThings Hub (v2 / v3 / Aeotec / Station) — Local Control & Setup Research Notes

## What it is
Samsung's smart-home hub: a Zigbee + Z-Wave + Matter + LAN controller that fronts
its attached devices through the SmartThings **cloud**. Unlike the Vera/MiOS hubs
(the reference local-first controller — see `vera-mios-hub`), SmartThings has **no
documented local device-control API and no local onboarding path**. This note
records whether "control and set up the hub locally" is achievable today. Short
answer: **discovery is; control and setup are not.** It is close to the mirror
image of Vera.

## Cloud status (checked 2026-08-11)
Samsung SmartThings is very much alive — and moving the *developer* surface behind
a paywall. Free SmartThings API access is being **phased out from 2026-10**;
non-commercial/personal use moves to a paid **Personal plan (~$4.99/month)**. The
consumer app stays free; the programmatic path does not. Precedent for the risk is
concrete: Samsung **shut down the Hub v1 cloud on 2021-06-30**, bricking those hubs.
The hub's own cloud dependency for claiming and driver distribution is therefore the
central liability, not a footnote.

## Local path A — Matter multi-admin (confirmed local, but device-level, not the hub)
The one genuinely local, official, no-account, no-fee path — and it does **not**
control the hub. A **Matter** device joined to SmartThings can be *co-commissioned*
into a second Matter controller: in the SmartThings app, "share" the device to
generate a new QR / 11-digit numeric pairing code, then commission it into your own
local Matter controller. From then on you drive that device directly over LAN/Thread,
independent of the hub and the cloud.
- **Only Matter (incl. Matter-over-Thread) devices** — never the hub's Zigbee/Z-Wave
  children (see path D).
- Per-device fabric limit (~5 ecosystems); the original setup code is single-use.
- This **bypasses the hub** — you become a co-controller of the *device*. It is the
  honest answer to "can I control it locally," but it is not "controlling the hub."
- Requires a Matter controller/commissioner on the client side. The liberatedbread
  app has none today (`rust/src/spec/types.rs` `Protocol = Ble | Wifi | Zigbee |
  Zwave | Other`), so this is a documentation/future target, not a shippable path.

## Local path B — Edge driver + LAN relay (local at runtime, cloud-gated to install)
SmartThings **Edge** drivers are Lua programs that run **on the hub** and can open
LuaSocket TCP/UDP/TLS sockets. The community "edgebridge" pattern (e.g.
`toddaustin07/lantrigger`) pairs such a driver with a small always-on **LAN relay**
so a local app can push triggers to, and read state from, hub-connected devices —
fully local once running, and resilient to internet outages.
- Installing a custom driver needs a **Samsung developer account + a cloud-enrolled
  driver channel** (a one-time cloud step; the hub downloads the driver from Samsung).
- Per-integration Lua work; undocumented primitives; Samsung can break it with a
  firmware bump. Community-hack-tier, not an API contract.
- This is the *only* route that reaches Zigbee/Z-Wave devices locally.

## Local path C — port 9495 CLI API (local, but diagnostic-only, needs a cloud token)
The hub exposes an undocumented local HTTPS API on **port 9495** used by the official
`smartthings-cli` (`edge:drivers:logcat`, `defaultLiveLogPort = 9495`): `GET /drivers`
and `GET /drivers/logs` (Server-Sent Events). It is a real LAN endpoint, but:
- It authenticates with a **cloud-minted Samsung OAuth bearer** and pins the hub's TLS
  certificate to its cloud identity — so it is "one-time cloud, then local," at best.
- It is **diagnostic** (list installed drivers, stream their logs). It does **not**
  control devices. Not a control surface.

## Local path D — the hub as a Matter bridge (does not exist)
Samsung has confirmed SmartThings hubs **do not act as a Matter bridge**: attached
Zigbee/Z-Wave/cloud devices are **not** exposed to other Matter controllers. The hub
is a Matter **controller only**. So there is no "point one integration at the hub and
get all its devices" — the single most-wanted local path is explicitly unavailable.

## Setup / onboarding — no local path
Hub onboarding is `cloud_account`: the welcome code printed on the hub is redeemed
against Samsung's service (BLE/Ethernet + Samsung account + cloud), which binds the
hub and provisions its drivers. The existing spec already records
`local_alternative: "None known."` There is no third-party local hub-setup path.

## What needs cloud
Claiming an unclaimed hub; Edge driver distribution/install; all device control via
the public API; even the local 9495 token. Local execution of *already-installed*
Edge automations continues without internet, but nothing a third party can stand up
locally does.

## Discovery — confirmed, and already shipped in the app
mDNS/DNS-SD, no cloud needed to find the hub:
- `_smartthings._tcp.local.` :8081 — primary identity (TXT `id`, `type`, `path`).
- `_smartthings-hedge._tcp.local.` :8766 — Edge driver WebSocket (`feat=ctrl`, **unverified**).
- `_matter._tcp.local.` :49722 — Matter operational node (`T=6`, ephemeral port, **unverified**).
The liberatedbread app already discovers and names the hub from `_smartthings._tcp`;
it then opens a read-only details sheet, which is the correct behaviour given the above.
Full spec: `device-specs/devices/smartthings-hub-v2.yaml`.

## Existing implementations
- Home Assistant `smartthings` integration — **cloud** (Cloud Push, OAuth); no local hub path.
- `SmartThingsCommunity/smartthings-cli` — source of the 9495 local diagnostic API.
- `toddaustin07/lantrigger` + edgebridge — the community local-relay pattern (path B).
- `QuiteYellow/SmartThings-Local` / "Localthings" — local CoAP-DTLS/OCF control of Samsung
  **appliances** (washers/fridges/ovens), **not** the hub or its children. Out of scope here.

## Open questions
- Is the `_smartthings-hedge._tcp` :8766 WebSocket (`feat=ctrl`) a usable local control
  channel, or auth-gated the same way as 9495? Rests on a single 2026-07-16 probe.
- Can the 9495 OAuth bearer be minted once and cached locally long-term, or does it
  expire and force periodic cloud round-trips?
- Does any firmware expose Matter-bridge capability (path D) in a later release?

## Rating
**Partial — mostly rejected for local.** Discovery/identification: **confirmed** and
shipped. Local hub **setup**: **rejected** (cloud-account only, no local alternative).
Local **device control through the hub**: **rejected** (hub is a Matter controller, not
a bridge; Edge distribution and onboarding are cloud-gated; 9495 is diagnostic-only).
The single local control path is **Matter multi-admin** — device-level, Matter-only,
bypasses the hub, and needs client-side Matter support the app lacks. Treat SmartThings
as a *discovery-and-documentation* target, not a local-control target.

## Sources (accessed 2026-08-11)
- developer.smartthings.com/docs/devices/hub-connected/ — Edge / hub-connected model, Lua, local execution
- developer.smartthings.com/docs/devices/hub-connected/edge-architecture — driver channels, LuaSocket
- developer.smartthings.com/blog … "Why We Chose Lua for SmartThings Edge Drivers" (2021-09-15)
- blog.smartthings.com "A New Enhanced SmartThings API Experience" — API paywall from 2026-10 (~$4.99/mo)
- home-assistant.io/integrations/smartthings — cloud (Cloud Push), OAuth; API-fee notice
- SmartThings statement via SmartThingsBeat (2022) + techhive.com — hub is NOT a Matter bridge
- support.smartthings.com … "SmartThings x Matter Integration" + docs.silabs.com multi-admin — share codes, ~5-fabric limit
- github.com/SmartThingsCommunity/smartthings-cli — 9495 local API (edge/drivers/logcat.ts)
- github.com/toddaustin07/lantrigger — Edge driver + edgebridge LAN relay pattern
- github.com/QuiteYellow/SmartThings-Local + mostlychris.com — local CoAP-DTLS for Samsung *appliances* (out of scope)
- community.smartthings.com (2026-06) — "no local API for the hub" confirmation
