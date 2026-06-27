---
layout: home
title: Bird Away&#58; Building an AI-Powered Bird Deterrent with Raspberry Pi and Claude
---

*June 2026 · [Source on GitHub](https://github.com/mattsahn/bird-away)*

![Bird Away](https://github.com/user-attachments/assets/51d76a95-e8b6-47e1-b92c-d38c6e815195)

## The Problem

This spring my pool started attracting birds — cormorants, herons, the occasional duck. The problem with birds and pools isn't just aesthetics: waste raises pathogen levels, throws off the chemistry, and makes the water genuinely unpleasant. Shooing them by hand works for about thirty seconds.

Motion-activated sprinklers exist at hardware stores, but they trigger on anything — leaves blown past a sensor, a kid running by, a neighbor's cat. I wanted something smarter: a system that could look at the pool and specifically recognize birds before firing the sprinkler.

Vision models have gotten good enough that "does this image contain a bird" is a one-API-call problem. So I built it.

## The Idea

Three parts: a camera watching the pool, an AI model deciding whether what it sees is a bird, and a sprinkler that fires when it does. The camera streams RTSP over the home network. The model runs in the cloud via API — no need to run inference locally on the Pi. The sprinkler is controlled by a GPIO relay.

![Hardware chain](https://github.com/user-attachments/assets/c858b5ad-f7e6-4a01-8866-c37de9c4c621)

## Hardware

- **Raspberry Pi 4** on home Wi-Fi — runs the Python service as a systemd unit
- **RTSP IP camera** — aimed at the pool, reachable on the local network
- **Relay module** — wired to GPIO pin 17 and ground; active-high or active-low is configurable
- **12V solenoid valve** — on its own power supply, switched by the relay; never wired to the Pi directly
- **Sprinkler head** — aimed to spray over and around the pool surface
- **Status LED** — heartbeat-blinks while the service is running, rapid-blinks on a detection
- **Momentary button** — manual trigger that runs the same capture/spray/record flow as a real detection

![Pi enclosure](https://github.com/user-attachments/assets/63336784-0bd0-4567-83ed-71b2f77b96e4)

The key constraint: the Pi only switches the relay's signal line. The solenoid's 12V supply is entirely separate. Running high-current DC through GPIO is a reliable way to destroy a Pi — the pins are rated for 8mA, and a typical solenoid pulls 200–400mA.

## How It Works

The service runs as a Python process under systemd, looping every 60 seconds (configurable). Each iteration passes through four gates before anything physical happens:

```
RTSP camera (background thread, persistent connection)
     │
     ▼
daytime gate (07:00–19:00) ───── outside hours → sleep
     │
     ▼
motion gate (local frame-diff) ── quiet scene → sleep
     │
     ▼
vision model (Claude via OpenRouter) ── "no bird" → sleep
     │
     ▼
_handle_event()
  ├── save snapshot JPEG
  ├── start ffmpeg recorder (pre-roll + spray + post-spray)
  ├── fire sprinkler relay
  └── upload to Cloudflare R2
```

### Camera thread

A background thread holds one long-lived RTSP/TCP session using [PyAV](https://pyav.org) and continuously decodes frames into a shared buffer. The main loop calls `cam.capture_frame()` to pull the most recent JPEG from RAM — no per-iteration reconnect, no handshake delay. When an event fires and ffmpeg needs its own RTSP session, the background thread pauses temporarily to avoid saturating the camera's connection limit (most cheap IP cameras allow one or two sessions).

### Daytime gate

Detection only runs between 07:00 and 19:00 local time. Birds aren't active at 3am, and skipping the night saves cycles. A single `datetime.now().hour` check at the top of the loop is all it takes.

### Motion gate

Before calling any external API, the service compares consecutive frames using a local pixel-diff. If the mean per-pixel change is below a configurable threshold (default `5.0` on a 0–255 scale), nothing moved and the API call is skipped entirely.

This gate eliminates the vast majority of API calls on calm days — pool water changes very little between frames when nothing disturbs it. A stationary bird that landed between iterations will still be caught, because it will be moving on the next frame (shifting, preening, causing water ripples).

### Vision model

When the motion gate opens, the frame is downscaled to 512px on its longest edge — cheaper to process, still enough resolution to distinguish a heron from a pool toy — and sent to Claude via [OpenRouter](https://openrouter.ai):

```python
resp = client.chat.completions.create(
    model="anthropic/claude-haiku-4-5",
    max_tokens=4,
    messages=[
        {"role": "system", "content": detector_prompt},
        {
            "role": "user",
            "content": [{"type": "image_url", "image_url": {"url": data_uri}}],
        },
    ],
)
```

`max_tokens=4` is intentional — the answer is one word, and capping tokens keeps cost and latency low. The system prompt ends with *"Output only the single word."* The parser checks `response.strip().lower().startswith("yes")`, which tolerates a slightly verbose "Yes." without needing exact matching.

The default model is `anthropic/claude-haiku-4-5`, which is fast, cheap, and accurate enough for the vast majority of cases. The config accepts any OpenRouter model ID, so switching to a stronger model for tricky scenes is one line in `config.yaml`.

### Sprinkler and recording

On a "yes", `_handle_event()` runs:

1. Save a JPEG snapshot of the trigger frame
2. Start an ffmpeg subprocess recording from the RTSP stream
3. Wait `pre_spray_seconds` (default 3) — so the clip captures the moment *before* the deterrent fires
4. Open the relay via [gpiozero](https://gpiozero.readthedocs.io) for `spray_duration` seconds (default 3)
5. Wait for the recording to finish (`post_spray_seconds` after the spray)
6. Resume the background camera thread
7. Upload snapshot + clip to Cloudflare R2

The pre-roll is the part I'm most glad I added. Without it, the clip starts when the spray fires — you see a startled bird launching from the water but not what it was doing or how it approached. With 3 seconds of pre-roll you see the whole sequence.

![Bird detection event](https://github.com/user-attachments/assets/2fb09415-a968-4a3c-9c0a-fbbbb147f2fd)

*A snapshot from a detection event — the frame that triggered the spray*

## Cloud Layer

Cloudflare R2 stores the snapshots and video clips. R2 is S3-compatible, has no egress fees, and the free tier (10 GB) easily covers typical deterrent-system traffic with a 30-day lifecycle rule. A `manifest.json` index in the bucket tracks the last 500 events, with public CDN URLs for each file.

A static web dashboard reads the manifest and renders an event timeline — thumbnail cards for each detection, date and trigger-type filtering, and a lightbox for full-resolution viewing. The dashboard is a single `index.html` with no build step, deployed to Vercel.

![Event dashboard](https://github.com/user-attachments/assets/b501166a-2edc-4928-a1f7-470b0a64b1cd)

*The web dashboard: each card is a detection with a thumbnail, timestamp, and link to the video clip*

## Engineering Notes

### The motion gate cuts API costs to near zero on quiet days

Without the local frame-diff gate, you'd pay for a vision API call every 60 seconds all day. Haiku is cheap, but at 60-second intervals that's 600 calls during a 10-hour daytime window — every day of summer. With the motion gate, a calm day with nothing happening at the pool costs essentially nothing. The gate opens when something actually changes: a bird lands, the water ripples, an object crosses the frame.

There's a secondary benefit: fewer API calls also means fewer opportunities for the model to hallucinate a bird on a borderline frame. The gate filters out the lowest-information frames before the model ever sees them.

### Zero SD card writes

Raspberry Pi SD cards die from write exhaustion. A system recording video clips all summer is a real write workload — every detection produces a JPEG and an MP4. Two config settings together eliminate almost all of it:

```yaml
capture_dir: /dev/shm/bird-away   # Linux tmpfs — lives in RAM, never touches flash
delete_after_upload: true          # remove local copy after R2 confirms receipt
```

`/dev/shm` is a kernel-managed RAM filesystem, sized to half of system RAM by default. With `delete_after_upload: true`, the JPEG snapshot bytes go straight from memory to R2 without touching disk at all. The MP4 lives briefly in RAM while ffmpeg writes it, then is deleted once R2 confirms the upload.

The tradeoff is clear: an extended R2 outage loses clips. For a deterrent system — where the deterrence matters more than archiving every clip — that's the right trade. SD card failure is permanent; a missed video is not.

### Two watchdogs, one liveness check

For unattended deployments that need to run for months, resilience matters. The service has three layers:

**Software watchdog.** The systemd unit uses `Type=notify` with `WatchdogSec=120`. The main loop pings `WATCHDOG=1` at the top of each iteration and once per second while sleeping. If the loop hangs for more than two minutes, systemd kills and restarts the process automatically.

**Hardware watchdog.** The Pi's BCM watchdog resets the SoC if even the kernel hangs. Enabling it requires one line in `/etc/systemd/system.conf`:

```
RuntimeWatchdogSec=15
```

**Liveness alerting.** [healthchecks.io](https://healthchecks.io) gets a ping after each successful loop iteration. This catches the failure mode the watchdogs miss: the loop is alive and pinging the watchdog, but every iteration is failing — camera unreachable, API down, RTSP credentials expired.

### Tuning the detector prompt

The model's system prompt is the main accuracy lever. A few things that matter:

**Be specific about what counts.** "A bird" is ambiguous — does a distant seagull in flight above the pool count? A reflection? The default prompt explicitly covers *"birds in, on, or near the pool (including birds in flight directly above it)"* to reduce these edge cases.

**Pin the output format.** Ending the prompt with *"Output only the single word: yes or no."* prevents the occasional verbose response from being parsed as `no`. Without it, a hedged *"I'm not entirely certain…"* would fail the `startswith("yes")` check even if the model genuinely meant yes.

**Iterate against saved frames.** Every detection saves a JPEG to `captures/`. The repo includes a script that runs any prompt against a single saved frame and prints the raw response:

```bash
python scripts/test_detector.py captures/detection-20260622T192023Z.jpg
```

Running this against a handful of known bird and non-bird frames before restarting the service is the fastest way to validate a prompt change.

## Results

The system has been running for several weeks. Detection accuracy is high enough that false positives — sprays when no bird is present — are rare. The occasional false trigger comes from large leaves blown across the field of view during a motion-gate window, or a visitor's dog approaching the pool edge.

The sprinkler is effective. Birds sprayed once tend to approach much more cautiously on subsequent visits. Some don't return at all. The pre-roll clips showing the seconds before each spray are useful for reviewing edge cases and entertaining to watch.

The full source, wiring notes, install instructions, and tuning guide are at [github.com/mattsahn/bird-away](https://github.com/mattsahn/bird-away).

---

*Questions or feedback: [open an issue on GitHub](https://github.com/mattsahn/bird-away/issues).*
