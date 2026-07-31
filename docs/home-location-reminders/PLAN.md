# Home Location-Aware Reminder System — Design Plan

A fully local system that knows which room (or yard zone) you are in, lets you
create reminders by voice from your iPhone, and speaks up at the right moment —
e.g. *"Don't forget the celery in the refrigerator"* as you walk out the front
door on your way to work.

Design constraints:

- **100% local** — no cloud services; works with the internet down.
- **Off-the-shelf, inexpensive hardware** — commodity ESP32 boards and Zigbee
  sensors, built around hardware already on hand: an **Nvidia Jetson** for
  compute and **two Luxonis OAK cameras** for vision.
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
4. As you head down the hallway, the OAK-D camera at the door sees a person
   moving **toward the exit** and pre-arms the reminder. You open the front
   door — the contact sensor fires **instantly**; the presence system
   confirms it was *you* (not the dog, not another household member) and that
   the context matches (weekday, 7–9 am, motion toward the exit).
5. The system speaks the reminder through the speaker nearest the door **and**
   sends a critical push notification to your iPhone: *"Before you go — celery,
   refrigerator."*
6. You reply "done" (voice or tap) and the reminder clears; if you don't
   acknowledge, it repeats once and re-arms.

---

## 2. Architecture overview

```
                         ┌─────────────────────────────────────────┐
                         │      LOCAL SERVER — NVIDIA JETSON       │
                         │            (already owned)              │
                         │                                         │
  iPhone 17 ────────────▶│  Home Assistant  ── automation engine   │
  (HA Companion app,     │  MQTT broker (Mosquitto)                │
   Assist voice UI,      │  ESPresense companion / Bermuda         │
   Siri Shortcuts)       │  Wyoming voice stack (CUDA):            │
                         │    faster-whisper (STT, GPU)            │
        ▲                │    Piper (TTS)                          │
        │ WiFi (LAN only)│  Ollama + local LLM (GPU, NL parsing)   │
        │                │  DepthAI host service (camera events)   │
        │                └──────┬─────────┬─────────┬──────────────┘
        │                       │MQTT/WiFi│ Zigbee  │ USB3/PoE
        │                       ▼         ▼         ▼
   ┌────┴──────────┐  ┌──────────────┐ ┌─────────┐ ┌────────────────┐
   │ BLE presence  │  │ ESP32 nodes  │ │ Zigbee  │ │ 2× Luxonis OAK │
   │ source: phone │─▶│ 1 per room   │ │ door /  │ │ (already owned)│
   │ IRK, watch,   │BLE│ (ESPresense) │ │ PIR     │ │ on-camera      │
   │ or BLE tag    │  │ ~$5–8 each   │ │ sensors │ │ person detect  │
   └───────────────┘  └──────────────┘ └─────────┘ └────────────────┘
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

### 3.4 Vision — two Luxonis OAK-D cameras (already owned)

The OAK-D cameras are the system's precision layer. Their onboard VPU runs
the neural nets **on the camera itself** (DepthAI pipeline), so the Jetson
only receives lightweight detection metadata — not video — and stays free for
voice/LLM work. Being stereo-depth cameras, every person detection comes with
**spatial (X,Y,Z) coordinates** out of the box (DepthAI's
`spatialDetectionNetwork` + object tracker), i.e. real positions in meters,
not just bounding boxes.

- **Camera 1 — exit hallway / front door, facing into the house.** On-device
  person detection + tracking with spatial output gives distance-to-door and
  direction of travel, so it distinguishes "walking toward the door" from
  "walking past it" seconds before the door even opens. Define a virtual
  trip-zone (e.g. within 2 m of the door, velocity toward it) → publishes
  `person_toward_door`.
- **Camera 2 — kitchen or back yard**, whichever matters more day-to-day:
  - *Kitchen:* depth-defined zones ("at the fridge", "at the counter") for
    in-kitchen reminders — with stereo depth these zones are actual 3D boxes,
    immune to the perspective false-positives a 2D camera would give.
  - *Yard:* person detection with position over the whole yard from one
    vantage point — better coverage than several BLE nodes, and it works
    when you leave your phone inside.
- A small **DepthAI host service on the Jetson** (Python, ~100 lines)
  subscribes to each camera's detection stream and publishes zone events to
  MQTT: `vision/hallway/person_toward_door`, `vision/yard/person_present`.
  Home Assistant consumes these like any other binary sensor.
- **Privacy by construction:** inference happens on-camera, frames never
  leave the LAN, and the host service can be configured to consume metadata
  only — no recording at all.
- Cameras detect *a* person, not *which* person — identity still comes from
  the BLE layer (§3.1). Vision provides speed and geometry; BLE provides
  identity.

### 3.5 Fusion logic

Home Assistant holds a single `person.glen` room state, updated by:

1. ESPresense/Bermuda BLE area (slow, identity-bearing, ~85–95% right),
2. corrected by OAK vision zone events and motion/mmWave events (fast,
   precise, identity-free),
3. sanity-checked by door events (you can't be in the yard if no exterior
   door opened).

A small automation (or the stock "presence simulation" pattern) is enough; no
ML required on the fusion side — the ML already ran on the cameras.

---

## 4. Compute — the Jetson (already owned)

The Jetson is the single server for everything: hub, voice, LLM, and the
camera host service. The GPU is what elevates this from "budget build" to
genuinely good:

- **GPU-accelerated Whisper** — run `small`/`medium` with CUDA for fast,
  accurate transcription (vs. `base`-class models on a Pi). Voice commands
  feel instant.
- **A real local LLM** — Ollama with CUDA runs a 3–8B model (e.g. Llama 3.x
  or Qwen 3 class) at conversational speed, so free-form reminder phrasing
  ("uh, remind me about the celery thing tomorrow before work") parses
  reliably into structured reminders.
- **Headroom for vision** — the OAK-D cameras do their own inference, so the
  Jetson's GPU stays available for voice/LLM; the DepthAI host service is
  CPU-trivial.

Deployment notes by Jetson generation:

| Jetson | Fit |
|---|---|
| **Orin family** (Orin Nano / NX / AGX, JetPack 5/6) | Ideal. Everything below runs in Docker (arm64 + CUDA) without friction. |
| **Xavier NX / AGX Xavier** | Good. Same stack; use JetPack 5-compatible CUDA images. |
| **Original Nano (4 GB, JetPack 4.x)** | Workable but tight: run HA + Mosquitto + DepthAI on the Nano, prefer Whisper `base` and skip the LLM, or keep the Nano for cameras only and pair a Pi for HA. |

Software stack (all free, all local, all standard) — run as containers:

- **Home Assistant Container** — hub, automation engine, iPhone app endpoint.
- **Mosquitto** — MQTT broker for the ESP32 fleet and camera events.
- **ESPresense companion** or **Bermuda** — BLE room resolution.
- **Wyoming voice pipeline**: `faster-whisper` (STT, CUDA) + **Piper** (TTS).
- **DepthAI host service** — Python service driving both OAK-D pipelines and
  publishing zone events to MQTT (§3.4).
- **Ollama + 3–8B model** — natural-language reminder parsing (§6).

### 4.1 Repurposing the Raspberry Pis and Arduinos (already owned)

**Raspberry Pis — best used as voice satellites.** A Pi + a cheap USB
speakerphone (~$20, e.g. a used Jabra 410) running **Wyoming Satellite** with
local wake word gives a hands-free mic/speaker station identical in role to a
$59 HA Voice PE — audio streams to the Jetson's Whisper/Piper over the LAN.
Put one near the front door (it doubles as the announcement speaker for
leaving-home reminders) and one in the kitchen. Other good Pi roles, if
preferred:

- **BLE room scanner** — a Pi's onboard Bluetooth running
  `andrewjfreyer/monitor` (or room-assistant) covers a room's BLE presence
  over MQTT, saving an ESPresense node in up to two rooms.
- **Yard/outbuilding node** — a Pi in the garage or shed can host the BLE
  scanner and a wired PIR/reed switch in one weatherproof box.
- **Fallback HA host** — only relevant if the Jetson is an original 4 GB
  Nano: run HA + Mosquitto on a Pi and let the Nano do cameras + voice.

**Arduinos — depends on the board:**

- **WiFi-capable boards** (Uno R4 WiFi, Nano 33 IoT, MKR WiFi, or any
  ESP8266-based "Arduino") → standalone MQTT sensor nodes, same role as the
  WiPy in §10: reed switch on the fridge or gate, PIR in the mudroom,
  publishing to Mosquitto.
- **Classic AVR boards** (Uno R3, Nano, Pro Mini — no radio) → wired
  helpers rather than network nodes: hang reed switches/PIR off one and
  connect it over USB-serial to a nearby Pi or the Jetson, or use one as a
  door-side annunciator (piezo chirp + LED as a low-latency, zero-network
  reminder cue). Also ideal for bench-prototyping sensor placement before
  committing to Zigbee purchases.

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
- **Hands-free in key rooms**: use the two Raspberry Pis as **Wyoming
  Satellite** stations (§4.1) — each needs only a ~$20 USB speakerphone — in
  the kitchen and near the front door. Wake word, mic, and speaker; the
  satellite near the door is the speaker that announces the celery reminder.
  (Dedicated hardware like the $59 HA Voice PE or a $50 ESP32-S3-BOX-3
  remains an option for additional rooms later.)

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

Already owned (no cost): **Nvidia Jetson** (server), **2× Luxonis OAK-D**
(vision), **2× Raspberry Pi** (voice satellites / BLE scanners), **Arduino
boards** (wired sensor helpers), **iPhone 17** (voice interface).

| Item | Qty | Unit | Subtotal |
|---|---|---|---|
| ESP32 dev boards (ESPresense nodes; 2 rooms covered by Pi BLE scanners) | 6 | $6 | $36 |
| USB power adapters/cables for nodes | 6 | $3 | $18 |
| Weatherproof boxes (yard node / outdoor OAK-D) | 2 | $8 | $16 |
| Zigbee USB dongle | 1 | $25 | $25 |
| Zigbee door contact sensors | 4 | $12 | $48 |
| Zigbee PIR motion sensors | 2 | $10 | $20 |
| BLE keychain beacon (optional) | 1 | $10 | $10 |
| USB speakerphones for Pi voice satellites | 2 | $20 | $40 |
| Camera mounts / USB3 extension or PoE for OAK-D | 2 | $15 | $30 |
| **Total new spend** | | | **≈ $245** |

The OAK-D at the hallway replaces the mmWave sensor from the earlier draft,
the Jetson replaces the mini PC, and the Pis replace the dedicated voice
satellites — together cutting new spend roughly in half versus the
buy-everything build. Minimum viable version (phone-only voice, 4 rooms,
Pi BLE scanners instead of ESP32s): **under $100**.

---

## 8. Build phases

1. **Phase 0 — Jetson server & voice (weekend 1).** Docker on the Jetson:
   Home Assistant + Mosquitto + Wyoming Whisper (CUDA) / Piper; pair the
   iPhone app; verify you can talk to Assist locally.
2. **Phase 1 — Instant sensors (weekend 1).** Zigbee dongle + front-door
   contact + hallway PIR. Build the first automation: door opens on a weekday
   morning → speak a hard-coded test reminder. *This alone already delivers
   the celery scenario, minus identity.*
3. **Phase 2 — Vision (weekend 2).** Mount OAK-D #1 in the exit hallway;
   stand up the DepthAI host service with a spatial person-detection +
   tracking pipeline; define the door trip-zone; wire
   `vision/hallway/person_toward_door` into the automation as a pre-arm
   signal. Place OAK-D #2 (kitchen or yard) the same way.
4. **Phase 3 — Room presence (weekend 3).** Flash 3–4 ESP32s with ESPresense,
   extract the iPhone/Watch IRK (or use a fob), tune per-room RSSI thresholds,
   add the `person.glen in exit zone` identity condition to the automation.
5. **Phase 4 — Reminder engine (weekend 4).** Reminder store + intents
   ("remind me to X when I leave / when I'm in the Y"), Ollama-based
   free-form parsing, acknowledgment flow, snooze/re-arm.
6. **Phase 5 — Coverage & polish.** Remaining rooms, yard node, the two Pi
   voice satellites (Wyoming Satellite + USB speakerphones, front door +
   kitchen), fridge-door sensor (Zigbee, or an Arduino + reed switch per
   §4.1).

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
| Camera privacy concerns in the home | OAK-D inference is on-camera; host service consumes metadata only (no frames stored); cameras limited to transit zones, not private rooms |
| OAK-D USB3 cable length limits (~2 m) | Active USB3 extension, or PoE models/adapter if runs are long; the Jetson can also sit near the hallway camera |
| Jetson model constraints (older Nano) | See §4 table — shrink Whisper model and skip the LLM on a 4 GB Nano, or dedicate it to cameras and add a Pi for HA |

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
