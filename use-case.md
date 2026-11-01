# Rover Relay: Use Case and Build Plan

## Product

Rover Relay, remote hands for the 3 a.m. outage. An on-call AI agent pages a physical rover over Agent Relay. The rover drives to a server rack, reads the status light and asset tag with OpenAI vision, says what it found aloud, replies in the Relay thread, and a ticket with photo evidence is created. Durable Actors own the mission queue, the control lock, and crash recovery.

## Hardware

- The Framework laptop rides on the rover with its screen facing forward: webcam = eyes, display = ghost face, speaker = voice.
- An Arduino Uno connected over USB serial drives an L298N motor driver and 2 DC motors (tank drive, no encoders).
- A phone mounted on a cardboard "rack" shows the status lights.
- Everything runs on the rover laptop. All devices share one phone hotspot, and other devices use the rover laptop's LAN IP.

## Read first (15 min max; trust the docs over this summary)

- Durable Actors: https://docs.useterse.ai/quickstart, https://github.com/TerseAI/durable-actors (README, examples/ai-chat).
  - CLI: `npx durable-actors init|dev|generate`.
  - Actors: `class X extends Actor<Member, Incoming, Outgoing>`, `@Persisted`, `onConnect`, `onMessage`, `this.broadcast`, `@Reentrant`.
  - Client: `actors.X.get(id).method()`, `prepareWebsocket`.
  - Requires Node 22.19+, pnpm, Bun 1.3.9+.
- Agent Relay: https://agentrelay.com/docs, https://github.com/agentworkforce/relay. Find the simplest TypeScript way to register named agents and send and receive threaded messages.

## Repo (TypeScript pnpm monorepo)

- `firmware/rover/rover.ino`
- `actors/`: Durable Actors (Rover, Rack)
- `apps/rover`: rover brain = serial bridge + mission executor + Relay agent "rover"
- `apps/agents`: Relay agents "oncall", "db-agent", "network-agent"
- `apps/server`: Express; actor websocket grants, camera-frame and speech relay to the face page, Supabase writes
- `web/`: Vite + React pages `/rack` (phone), `/face` (rover screen), `/dashboard`
- `.env.example` with: `OPENAI_API_KEY`, `VISION_MODEL`, `TEXT_MODEL`, `SUPABASE_URL`, `SUPABASE_SERVICE_KEY`, `SERIAL_PORT`, and the Relay variables from its docs. Never commit secrets.

## Rules

- Build only the milestone sent. After each one, run it, show proof it works, and git commit.
- Simple code, no auth, no over-engineering.
- Never fake Durable Actors or Agent Relay. Put Relay behind an adapter file (`relay.ts`). If a sponsor API blocks you for more than 20 minutes, stop and ask.
- Safety first: the motors stop on any error.

Reply with a short plan and the version checks to run. Wait for Milestone 1 before writing code.

---

## Prompt 1: motors (11:00–11:45)

**MILESTONE 1: firmware, serial bridge, calibration.**

1. `firmware/rover/rover.ino`:
   - 115200 baud, newline-terminated commands.
   - `M <left> <right> <ms>`: speeds -255..255, ms ≤ 2000; then stop and reply `DONE`.
   - `S`: stop, reply `OK`. `H`: heartbeat.
   - Watchdog: stop the motors if no line arrives for 500 ms.
   - Pins: ENA D5, ENB D6 (PWM; ENA/ENB jumpers removed), IN1 D7, IN2 D8, IN3 D9, IN4 D10.
   - Constants to invert each motor's direction.
2. `apps/rover/src/serial.ts` using `serialport`:
   - Auto-detect the Arduino port (`SERIAL_PORT` overrides).
   - Send a heartbeat every 200 ms.
   - `move(l, r, ms)` waits for `DONE`; `stop()` stops.
3. `pnpm --filter rover calibrate`: an interactive CLI that runs forward 1 s and left/right turns of 0.5 s so I can measure. Save `msPerMeter` and `msPer90deg` to `calibration.json`, and add helpers `forward(meters)` and `turn(deg)` that use them.
4. `pnpm --filter rover waypoints`: record the route HOME→RACK_B as a list of moves in `waypoints.json`, plus the reverse route.

Also explain how to upload the sketch with arduino-cli, and give a wiring checklist (battery → L298N 12V/GND; Arduino GND joined to the same GND).

## Prompt 2: Durable Actors (11:45–12:30)

**MILESTONE 2: Durable Actors.**

Scaffold `actors/` with `npx durable-actors init`.

First, prove persistence: show that a value survives restarting `apps/rover`. Then test restarting the actors dev server itself and report whether state survives that too.

Rover actor (id `rover-1`):

- `@Persisted` fields:
  - `missions`: array of `{id, from, priority (P1–P3), rack, slot, question, threadId, status, step (1–5), createdAt}`
  - `currentMissionId`
  - `possessedBy` (agent id of the current mission's owner) and `leaseExpiry`
  - `pose {x, y, heading}`
  - `log` (last 200 events)
  - `lastEvidenceUrl`
- Methods:
  - `submitMission(...)`: queue ordered by priority, then time; return the mission's queue position.
  - `startNext()`: set `currentMissionId`, `possessedBy = mission.from`, `step = 1`.
  - `advance(missionId, step, data)` and `complete(missionId, result)`.
  - `command(agentId, cmd)`: accept only from "rover" or the current `possessedBy`. Otherwise REJECT and log "rover is possessed by X".
  - `recover()`: return the current mission and step so the rover can resume after a crash.

Rack actor (id `rack-B`):

- Slots 1–4, each `{status: "ok" | "fail", assetTag}`.
- `setStatus(slot, status)`.

Both actors broadcast their full state on every change. Generate the client and write a script that exercises every method.

## Prompt 3: rover brain, eyes, voice (12:30–1:30)

**MILESTONE 3: rover brain, eyes, voice.**

1. `web/face`: a fullscreen black page with animated ghost eyes and a mouth that moves while it speaks, with a status line showing the mission and "step x/5".
   - It captures the laptop webcam with `getUserMedia`.
   - On a `capture` request over WebSocket it returns a 640px JPEG.
   - On a `say` request it speaks using `speechSynthesis` (fallback: OpenAI text-to-speech).
   - Add a big "Enable camera & audio" button to get browser permissions.
2. `apps/rover` mission executor: a 5-step state machine. Persist each step with `Rover.advance` BEFORE acting on it.
   1. ACCEPT: say "Boo-t sequence complete. Heading to rack B, slot 3."
   2. DRIVE: run the HOME→RACK_B waypoints.
   3. ALIGN: up to 3 loops of: capture a frame → vision call returning zod `{visible, horizontal: left|center|right, distance: far|ok|near}` → a small turn or move forward.
   4. INSPECT: capture → vision call returning zod `{slot, lightColor, assetTag, confidence}`. Retry once if confidence < 0.6.
   5. REPORT: upload the photo to the Supabase bucket "evidence", say "I see dead servers. Slot 3 is red, asset tag ...", and call `Rover.complete()`.
   - On boot: call `Rover.recover()`. If a mission is mid-flight, say "Resuming mission step N of 5" and continue from that step.
   - Any error stops the motors.
3. `scripts/run-rover.sh`: a restart loop with a 2-second delay, so a crashed brain comes back automatically.

Test everything with a hardcoded mission before adding Relay.

## Prompt 4: Agent Relay and the incident (1:30–2:30)

**MILESTONE 4: Agent Relay and the incident.**

1. `web/rack` (opened on the phone): a "RACK B" server panel with 4 slots. Each slot has a blinking green LED and a huge asset tag under it (e.g., DC-B3-0451). Tapping a slot calls `Rack.setStatus(slot, "fail")` and the LED turns solid red. Add a hidden reset button.
2. Relay agent "oncall":
   - Watches the Rack actor. On a new failure, it pages over Relay at priority P1: "@rover check rack B, slot 3: what color is the light and what's the asset tag?"
   - When the rover replies, it writes a row to the Supabase table `tickets` and posts "Ticket INC-xxxx created" in the thread.
3. Relay agent "rover":
   - Parses any message mentioning @rover with `TEXT_MODEL` into zod `{rack, slot, question, priority}`, so a human or a teammate's Claude Code can page it in plain English.
   - Calls `Rover.submitMission` and replies "On my way" or "Queued #N".
   - At step 5 (REPORT), replies in the original thread with the findings and the photo link.
4. `pnpm conflict`:
   - db-agent (P1) and network-agent (P2) page the rover at the same instant.
   - network-agent then tries a direct `Rover.command("network-agent", "drive")` and must be REJECTED.
   - Mission 2 starts automatically after mission 1 finishes.
5. Supabase: a SQL migration for the tables `tickets` and `mission_events`, plus a public storage bucket "evidence".

Prove it end to end: tap the phone → the rover drives, reads, speaks, and replies → a ticket appears.

## Prompt 5: dashboard (2:30–3:15)

**MILESTONE 5: dashboard.**

`web/dashboard` in a dark "haunted server room" style (near-black, green and red LED accents, ghost icon, monospace), readable from 3 meters, with no layout jumps.

Show:

- The live rover camera.
- A mirror of the rack panel.
- A mini-map of the rover's pose between HOME and RACK_B.
- The mission queue with priorities, the current step x/5, and a "👻 possessed by db-agent" badge.
- The Relay feed, showing every message between oncall, db-agent, network-agent, and rover, grouped by thread.
- Rejections in red, e.g. "REJECTED: rover is possessed by db-agent".
- The latest ticket card with its evidence photo.

Buttons: Reset demo, Return home, Run conflict test, Crash brain (makes the rover process exit so `run-rover.sh` restarts it), and a big Emergency stop that is always visible.

## Prompt 6: harden and ship (3:15–4:00)

**MILESTONE 6: harden and ship.**

- `pnpm demo:check`: checks serial, actors, Relay, Supabase, OpenAI, and the camera socket, printing green or red for each.
- Add a timeout to every network call. If vision fails, the rover says "inconclusive" instead of hanging.
- README: a one-paragraph pitch, a mermaid architecture diagram, setup steps, the 2-minute demo script, and how each sponsor's tech is used.
- Provide a 2-minute narration script synced to the dashboard, plus a reset checklist to run between rehearsals.
