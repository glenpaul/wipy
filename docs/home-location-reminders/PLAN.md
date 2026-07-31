# Home Location-Aware Reminder System — Design Plan

A fully local system that knows which room (or yard zone) you are in, lets you
create reminders by voice from your iPhone, and speaks up at the right moment —
e.g. *"Don't forget the celery in the refrigerator"* as you walk out the front
door on your way to work.

Design constraints:

- **100% local** — no cloud services; works with the internet down.
- **Off-the-shelf, inexpensive hardware** — commodity ESP32 boards, Zigbee
  sensors, and a small single-board computer or used mini PC.
- **iPhone 17 as the human interface** — voice in, notifications and speech out.

---

## 1. How it works — the celery scenario

1. **The night before**, you say into your phone (or a kitchen voice satellite):
   *"Remind me to take the celery out of the refrigerator when I leave for work."*
2. The speech is transcribed **locally** (Whisper on the home server), parsed
   into a reminder record: `{task: "take celery out of refrigerator",
   trigger: leaving-home, window: weekday morning}`.
3. **Next morning**, the system tracks you at room level: bedroom → bathroom →
   kitchen → front hallway.
4. You open the front door. The door contact sensor fires **instantly**; the
   presence system confirms it was *you* in the exit zone (not the dog, not
   another household member) and that the context matches (weekday, 7–9 am,
   you were just in motion toward the exit).
5. The system speaks the reminder through the speaker nearest the door **and**
   sends a critical push notification to your iPhone: *"Before you go — celery,
   refrigerator."*
6. You reply "done" (voice or tap) and the reminder clears; if you don't
   acknowledge, it repeats once and re-arms.

---

## 2. Architecture overview

```
                         ┌─────────────────────────────────────────┐
                         │        LOCAL SERVER (~$130–200)         │
                         │  used mini PC (N100) or Raspberry Pi 5  │
                         │                                         │
  iPhone 17 ────────────▶│  Home Assistant  ── automation engine   │
  (HA Companion app,     │  MQTT broker (Mosquitto)                │
   Assist voice UI,      │  ESPresense companion / Bermuda         │
   Siri Shortcuts)       │  Wyoming voice stack:                   │
                         │    faster-whisper (STT, local)          │
        ▲                │    Piper (TTS, local)                   │
        │ WiFi (LAN only)│  Ollama + small LLM (optional NL parse) │
        │                └───────┬──────────────┬──────────────────┘
        │                        │ MQTT/WiFi    │ Zigbee (USB dongle)
        │                        ▼              ▼
   ┌────┴──────────┐   ┌──────────────────┐  ┌──────────────────────┐
   │ BLE presence  │   │ ESP32 nodes      │  │ Zigbee sensors       │
   │ source: phone │──▶│ 1 per room/zone  │  │  door/window contact │
   │ IRK, watch,   │BLE│ (ESPresense)     │  │  PIR motion          │
   │ or BLE tag    │   │ ~$5–8 each       │  │  ~$8–15 each         │
   └───────────────┘   └──────────────────┘  └──────────────────────┘
                                │
                       optional voice satellites
                       (ESP32-S3-BOX / HA Voice PE)
```

Everything above the sensor row runs on one small box on your LAN. Nothing
leaves the house.

---

## 3. Location sensing (house + yard)

Room-level presence is the hard part; the trick is to **fuse three cheap
signal types** rather than chase one perfect sensor.

### 3.1 Who & which room — BLE RSSI trilateration (ESPresense)

- Place one **ESP32 dev board (~$5–8)** running [ESPresense](https://espresense.com/)
  in each room/zone. Each node measures the Bluetooth signal strength of your
  device and reports over MQTT; the ESPresense companion software resolves
  "nearest node = current room."
- **What to track (pick one):**
  - **iPhone 17 directly** — iPhones randomize their BLE MAC address, but
    ESPresense supports tracking via the phone's **IRK** (Identity Resolving
    Key), which you extract once. This gives identity + room with nothing
    extra to carry.
  - **Apple Watch** — same IRK approach; better because it's on your wrist
    even when the phone is on the counter.
  - **A $10 BLE beacon fob** on your keychain — dead simple and reliable;
    keys are usually with you when leaving for work, which fits the use case.
- Typical performance: room-level accuracy with 1 node per room, update
  latency ~5–20 s. Good for "which room is Glen in," **not** good enough
  alone for split-second triggers — that's what §3.2 is for.
- Alternative/complement: **Bermuda** (a Home Assistant integration) does the
  same BLE-area logic using ESPHome bluetooth-proxy nodes — same $5 ESP32
  hardware, one firmware doing double duty (presence + BLE proxy).

### 3.2 Instant events — door contacts + motion

BLE presence is smooth but slow; door sensors are dumb but instant. Reminders
that depend on a *transition* ("walking out the door") should be **triggered by
the contact sensor and conditioned on presence**:

- **Zigbee door/window contact sensors (~$10–15)** on: front door, back door,
  garage door, refrigerator (yes — a contact sensor on the fridge closes the
  loop on "did I actually open the fridge before leaving").
- **Zigbee PIR motion sensors (~$10)** or **mmWave presence sensors
  (LD2410-based, ~$15–20)** in transition zones (front hallway, mudroom).
  mmWave detects *still* people, PIR detects *moving* people; hallways only
  need PIR.
- These attach via a **$20–30 Zigbee USB dongle** (SONOFF ZBDongle-E or
  similar) on the server. Zigbee sensors are battery-powered, last 1–2 years,
  and need no wiring.

### 3.3 Yard zones

- Put 1–2 ESPresense ESP32 nodes in weatherproof boxes at outdoor outlets
  (porch, garage, shed). BLE range of ~10–20 m outdoors gives you coarse yard
  zones: "front yard," "back yard," "garage."
- A contact sensor on the gate and a PIR under the eaves fill in transitions.
- If the yard exceeds WiFi range, a $20 outdoor WiFi extender or a powerline
  adapter to the shed solves it.

### 3.4 Fusion logic

Home Assistant holds a single `person.glen` room state, updated by:

1. ESPresense/Bermuda BLE area (slow, identity-bearing, ~85–95% right),
2. corrected by motion/mmWave events (fast, identity-free),
3. sanity-checked by door events (you can't be in the yard if no exterior
   door opened).

A small automation (or the stock "presence simulation" pattern) is enough; no
ML required.

---

## 4. Compute — one inexpensive box

| Option | Price | Notes |
|---|---|---|
| **Used/new N100 mini PC** (Beelink, etc.) | ~$130–170 | **Recommended.** Runs Whisper `small` in ~1 s, headroom for a 3B LLM. |
| Raspberry Pi 5 (8 GB) + PSU + SSD | ~$120–150 | Fine; Whisper `base` only, slower LLM. |
| Any old laptop/desktop you own | $0 | Perfectly adequate. |

Software stack (all free, all local, all standard):

- **Home Assistant OS** (or Container) — hub, automation engine, iPhone app.
- **Mosquitto** — MQTT broker for the ESP32 fleet.
- **ESPresense companion** or **Bermuda** — BLE room resolution.
- **Wyoming voice pipeline**: `faster-whisper` (speech→text) + **Piper**
  (text→speech). Runs comfortably on the N100; zero cloud.
- **Ollama + a 3B-class model** *(optional)* — parses free-form reminder
  phrasing ("uh, remind me about the celery thing tomorrow before work") into
  structured reminders more robustly than intent templates. Skippable at
  first; HA's built-in Assist intents handle simple phrasing.

---

## 5. Voice interface — iPhone 17 as the MMI

- Install the **Home Assistant Companion app**. Its **Assist** screen is a
  push-to-talk voice UI wired to *your* local Whisper/Piper pipeline — the
  audio goes to your server, not Apple/Google.
- Add a **Siri Shortcut / Action-button binding** ("Hey Siri, home assistant")
  that opens Assist directly, so creating a reminder is: press Action button →
  speak → done.
- Reminder delivery to the phone uses **actionable notifications** (buttons:
  *Done* / *Snooze 10 min*) and **critical alerts** for leaving-home reminders
  so they sound even in silent mode. (Note: HA push notifications transit
  Apple's push service — that is the one non-LAN hop, used only for
  notification delivery. On-LAN, the app can also receive them locally, and
  speech announcements are entirely local.)
- **Hands-free in key rooms** *(optional but recommended)*: one or two voice
  satellites — **Home Assistant Voice Preview Edition (~$59)** or an
  **ESP32-S3-BOX-3 (~$50)** — in the kitchen and near the front door. Wake
  word ("Okay Nabu"), mic, and speaker; the same satellite near the door is
  the speaker that announces the celery reminder.

---

## 6. Reminder engine

Data model (stored in HA as `todo`/helper entities or a small SQLite table
managed by an AppDaemon/Pyscript script):

```yaml
reminder:
  id: 2026-07-30-celery
  task: "Take the celery out of the refrigerator"
  who: person.glen
  trigger:
    type: zone_transition          # zone_enter | zone_exit | zone_transition | time
    from: house                    # any interior zone
    via: binary_sensor.front_door  # instant trigger source
    direction: leaving
  window: {days: [mon-fri], after: "06:30", before: "09:30"}
  delivery: [nearest_speaker, phone_critical]
  state: armed                     # armed | announced | acked | expired
```

Trigger evaluation (HA automation):

```
WHEN  binary_sensor.front_door opens
AND   person.glen's area was hallway/kitchen within last 90 s
AND   now() inside reminder.window
THEN  announce on nearest media_player + notify iPhone
      mark reminder 'announced'; re-arm once if not acked in 2 min
```

Creation pipeline:

```
voice → Whisper (STT) → Assist intent match
                         └─ fallback: local LLM → structured JSON → validate → store
confirmation spoken back via Piper ("Okay — I'll remind you when you leave for work.")
```

Other trigger types the same engine covers for free: *"when I go into the
garage, remind me the recycling goes out"* (zone_enter), *"when I'm in the
yard, remind me to check the drip line"* (zone_enter yard), plain timed
reminders.

---

## 7. Bill of materials (typical 8-room house + yard)

| Item | Qty | Unit | Subtotal |
|---|---|---|---|
| ESP32 dev boards (ESPresense nodes) | 8 | $6 | $48 |
| USB power adapters/cables for nodes | 8 | $3 | $24 |
| Weatherproof boxes (yard nodes) | 2 | $8 | $16 |
| Zigbee USB dongle | 1 | $25 | $25 |
| Zigbee door contact sensors | 4 | $12 | $48 |
| Zigbee PIR motion sensors | 3 | $10 | $30 |
| mmWave presence sensor (hallway) | 1 | $18 | $18 |
| BLE keychain beacon (optional) | 1 | $10 | $10 |
| Voice satellite (HA Voice PE) | 1–2 | $59 | $59–118 |
| **Mini PC (N100) server** | 1 | $150 | $150 |
| **Total** | | | **≈ $430–490** |

Minimum viable version (phone-only voice, 4 rooms, no satellites, Pi you
already own): **under $150**.

---

## 8. Build phases

1. **Phase 0 — Server & voice (weekend 1).** Install Home Assistant +
   Mosquitto; set up Wyoming Whisper/Piper; pair the iPhone app; verify you
   can talk to Assist locally.
2. **Phase 1 — Instant sensors (weekend 1).** Zigbee dongle + front-door
   contact + hallway PIR. Build the first automation: door opens on a weekday
   morning → speak a hard-coded test reminder. *This alone already delivers
   the celery scenario, minus identity.*
3. **Phase 2 — Room presence (weekend 2).** Flash 3–4 ESP32s with ESPresense,
   extract the iPhone/Watch IRK (or use a fob), tune per-room RSSI thresholds,
   add the `person in exit zone` condition to the automation.
4. **Phase 3 — Reminder engine (weekend 3).** Reminder store + intents
   ("remind me to X when I leave / when I'm in the Y"), acknowledgment flow,
   snooze/re-arm.
5. **Phase 4 — Coverage & polish.** Remaining rooms, yard nodes, voice
   satellites, fridge-door sensor, optional local LLM for free-form phrasing.

---

## 9. Risks & mitigations

| Risk | Mitigation |
|---|---|
| iPhone BLE MAC randomization breaks tracking | Use IRK-based tracking (supported by ESPresense/Bermuda); or track Apple Watch; or carry a $10 fob on your keys |
| BLE room latency (5–20 s) misses fast transitions | Never trigger on BLE alone — trigger on contact/PIR events, use BLE as the identity *condition* with a 60–90 s lookback |
| Multiple people / pets cause false announcements | Identity comes only from BLE (per-person device/fob); PIR alone never fires a personal reminder |
| Whisper misheard the reminder | Always speak the parsed reminder back for confirmation before storing |
| WiFi dead spots in yard | Outdoor extender or powerline adapter; yard zones can be coarse |
| RSSI drift / node placement | ESPresense calibration per room; keep nodes away from metal and at ~1.5 m height |

---

## 10. Where WiPy fits (this repository)

The WiPy 1.x in this repo is built on the TI CC3200 — WiFi only, **no BLE
radio** — so it cannot serve as an ESPresense/BLE-proxy presence node. It can
still earn a place in the system as a **WiFi/MQTT sensor node**: wire a PIR, a
reed switch (door contact), or a relay to its GPIOs and publish events to the
Mosquitto broker with a short MicroPython script (see `examples/` and the MQTT
libraries under `lib/`). Practically, though, new deployments should buy $6
ESP32 boards for the presence layer — they are cheaper, have BLE, and carry
the mature ESPresense/ESPHome firmware ecosystem. Later Pycom successors to
the WiPy (WiPy 2/3, ESP32-based) can run ESPHome/ESPresense directly.
