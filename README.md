# Lazarus

Recovery infrastructure for abandoned cloud-dependent devices. Lazarus analyzes what a
device says on the network and recreates the minimum local services needed to control it
without the vendor's cloud.

## The problem

A lot of "smart" hardware isn't self-contained. You think it works like this:

```
Phone ──→ Bulb
```

But it often actually works like this:

```
Phone ──→ Vendor cloud ──→ Bulb
```

When the vendor shuts down the service, changes the API, or abandons the product, the
hardware becomes useless even though the radio, CPU, sensors and LEDs all still work.

Lazarus replaces the vendor cloud with software you run yourself:

```
Laptop / Raspberry Pi ──→ Lazarus ──→ Device
```

## The hard part

Devices don't speak a nice universal API. One sends JSON:

```json
{"device": "238195", "dp": 1, "value": true}
```

another publishes MQTT, another sends raw bytes:

```
A9 03 71 00 01 B2 8F
```

So the first question for every new device is: **what language does this device speak?**

## How it works

Guided reverse engineering, not magic auto-discovery. A human runs controlled experiments
while Lazarus records and compares the evidence.

```
Experiment 1: turn ON      → {"property": 1, "value": true}
Experiment 2: turn OFF     → {"property": 1, "value": false}
Experiment 3: brightness 25% → {"property": 2, "value": 250}
Experiment 4: brightness 75% → {"property": 2, "value": 750}
```

Lazarus diffs those traces and proposes:

```
property 1 ≈ power
property 2 ≈ brightness (range ≈ 0–1000)
```

Once a command is understood well enough, Lazarus sends it itself. If the bulb turns on,
that capability has been recovered locally.

## Normalization

Three vendors encode "power on" three different ways:

```
Vendor A   {"dp": 1, "value": true}
Vendor B   {"action": "POWER", "state": 1}
Vendor C   A9 01 FF 31
```

Lazarus hides that behind one vocabulary — `turn_on()`, `turn_off()`,
`set_brightness()`, `lock()`, `get_battery()` — so it becomes a platform instead of a pile
of one-off hacks.

## Architecture

```
CLI / REST / Home Assistant
          │
   Common device API
          │
   Protocol adapters      ← most vendor complexity lives here
          │
   Device profiles        ← small per-model config
          │
    Physical devices
```

Like an OS driver stack: generic interface on top, protocol adapters absorbing the mess,
thin per-device profiles at the bottom. Many unrelated products share a chipset, SDK, or
cloud platform, so one adapter can unlock many devices.

## Two recovery modes

**Direct local control** — Lazarus speaks the device's protocol straight to the hardware.

**Cloud emulation** — some devices dial out and expect something server-shaped on the
other end. Lazarus runs a small local server handling the handshake, heartbeats, status
messages, sequence numbers and retries.

## Scope and limits

| Difficulty | Examples |
|---|---|
| Good targets | local HTTP, UDP, plain MQTT, readable binary protocols |
| Workable | encrypted traffic where local control or partial structure is recoverable |
| Hard | certificate pinning, per-device crypto keys, strong server auth |
| Out of scope | the cloud did the actual work (e.g. server-side vision/ML) |

Lazarus will never claim "works with every smart device." The honest claim is: a general
framework for analyzing and recovering local control of cloud-dependent IoT devices, with
reusable support for protocol families and device profiles.

## Roadmap

- **V0** — manually liberate one device. `python control.py on` turns on a real bulb with
  no vendor cloud involved.
- **V1** — turn those hacks into tools: `lazarus capture`, `compare`, `inspect`, `control`.
- **V2** — a second, unrelated device. This is the real test of which abstractions were
  actually generic.
- **V3** — extract what turned out to be reusable into protocol adapters.
- **V4** — automate the repetitive inference: diffing, correlation, clustering, sequence
  alignment, state-machine inference.

## The research question

Not "can I reverse-engineer this device?" but:

> **How much less work should the next device require because Lazarus already learned from
> the previous ones?**

The result worth reporting isn't "supports 30 devices." It's "the first device took three
days of manual reverse engineering; by the third, shared inference and tooling cut
onboarding to 25 minutes."

## Status

Early. Nothing here works yet.
