# Hey Jarvis — a wake-word nightstand gadget for Muse

A firmware overlay for the **Muse Gadgets SDK** that turns a Waveshare ESP32-S3
round-display board into a bedside voice device for your own Muse agent:

- **"Hey Jarvis" wake word, on-device** — say it and a live hands-free voice
  call with your Muse starts. No phone, no button.
- **Alexa-style sessions** — the call stays open for follow-ups and closes when
  you say *thank you* (or *that's all*, *I'm done*, *goodbye*, *good night*),
  after 8 s of quiet, or if you never start speaking (5 s).
- **A hi-fi animated avatar** — embedded portrait frames with blink, listening,
  thinking and speaking states, drawn with region-only updates so the display
  never starves the audio path.
- **Live spoken replies** — this is the SDK's realtime voice mode, not
  transcription-only voice notes.
- **Hardening from a week of debugging** — a settings clamp that can't be
  silently muted by a stray tap, and a memory fix (see below) that keeps call
  audio clean over long uptimes.

Tested as a daily-use nightstand device. Current build: **wake-v8**.

## Hardware

| | |
|---|---|
| Board | Waveshare **ESP32-S3 Touch AMOLED 1.75C** (466×466 round display, dual mics, 8 MB PSRAM, 32 MB flash) |
| Price | ~$55 |
| ⚠️ | The plain **1.75** (non-C) is a *different board*. This overlay is for the **1.75C** only. |

## How it fits together

This repo is an **overlay**, not a fork. You clone Meta's SDK at a pinned
commit, copy `overlay/esp32/...` over it, and build. Everything under
`overlay/` is either a new file or a file that replaces its upstream
counterpart at the same path.

- Upstream: `facebookincubator/muse-gadget-sdk` @ `463bd2a`
- Toolchain: **ESP-IDF v6.0.3** (exact)
- Wake model: ESPHome microWakeWord `hey_jarvis.tflite` (Apache-2.0)

## What each piece does

| File (under `overlay/esp32/`) | What it is |
|---|---|
| `components/muse/muse_wake.cpp/.h` | On-device wake-word detector (microWakeWord runtime + TFLite). Adapted from the Hey-Muse project's module for this board; retargeted to "Hey Jarvis", cutoff 0.97 / window 5 per the model manifest, 24 KB tensor arena. |
| `components/muse/models/` | The model's manifest + notes. The `hey_jarvis.tflite` binary itself (52 KB) is **fetched from its canonical ESPHome source** during setup (AGENTS.md step 2, SHA-256 pinned) rather than stored here. Embedded into the firmware by the component CMake. |
| `components/muse/muse_voice.c` | Wake-session policy (Design C): detector feed from the idle capture path, end-phrase matching on the heard transcript, idle timeouts, settings-safe call flow. In v8 the detector stays **resident** during calls. |
| `components/muse/muse_ui.c` + `hifi_frames.h` | Hi-fi avatar: embedded 384×384 RGB565 frames (base/blink/wide/pulse/speaking), state machine, region-only invalidation. |
| `components/muse/avatar/muse_pixel.c` + `muse_pixel.h` | The custom pixel-avatar renderer at a 128 grid (fallback path; also where the portrait framing lives). |
| `components/muse/muse_settings.c` | Boot clamps: speaker forced ON, nonzero sleep floored to 600 s (a stray settings tap once muted the device for an evening). |
| `components/muse/muse_pmu.c/.h`, `boards/board_waveshare_s3_175c.c` | Board power/PMU wiring for the 1.75C. |
| `components/muse/CMakeLists.txt`, `idf_component.yml` | Build wiring: wake module + model embedding, TFLite flags, and the added deps (`esp-tflite-micro 1.3.3~1`, `esp-nn 1.1.2`, `esp-micro-speech-features 1.2.3`, all scoped to the 1.75C). |
| `managed_components/waveshare__.../esp32_s3_touch_amoled_1_75c.c` | 1.75C BSP fixes: ES7210 mic codec at its real address (`0x82`), native I2S format (16 kHz/16-bit/stereo), smaller audio DMA, graceful codec error returns. Without these the mic crashes or the realtime build panics. |
| `managed_components/esphome__esp-micro-speech-features/CMakeLists.txt` | One line that matters enormously: defines `USE_ESP32` so the wake frontend allocates in PSRAM (see "The memory story"). |

## The memory story (read this before changing anything)

The wake frontend (TFLM microfrontend) holds ~11.4 KB of small allocations.
Two facts collided:

1. ESP-IDF was configured with `SPIRAM_MALLOC_ALWAYSINTERNAL=4096`, so small
   `malloc`s land in scarce **internal** RAM — and internal RAM is what the
   link and the call media path also need. With the detector resident, calls
   starved (v2: session sends failing, 13 s call setups).
2. The frontend component *has* PSRAM-first allocation paths — but they're
   gated behind `#ifdef USE_ESP32`, and **nothing defined it**. Dead code.

v3–v7 worked around it by tearing the detector down for every call and
rebuilding it after — which fragmented the heap: over ~40 minutes free PSRAM
slid 1,276K → 1,084K and DMA-capable RAM fell to 13K, and call audio went
scratchy and slow.

**v8 is the structural fix:** define `USE_ESP32` for the component (its
buffers move to PSRAM; the frontend's internal cost at init measures ~0) and
keep the detector resident — no teardown cycle at all. Verified on device:
armed internal free 31K (vs 21K), DMA 23K (vs 13K), and heartbeats flat
(internal 34K / DMA 26K / PSRAM 1,684K) across a 25-minute soak where v7 was
already degrading.

## Build & flash

Full step-by-step, including every gotcha we hit, is in **[AGENTS.md](AGENTS.md)**
— written so a Muse (or other) agent can execute it end-to-end. The short
version:

```bash
git clone https://github.com/facebookincubator/muse-gadget-sdk
cd muse-gadget-sdk && git checkout 463bd2a
cp -r /path/to/this-repo/overlay/esp32/* esp32/
# apply sdkconfig.defaults.jarvis values (realtime voice, NVS encryption)
cd esp32
. ~/esp/esp-idf-v6/export.sh          # ESP-IDF v6.0.3, real Python env first
idf.py -B build-muse-waveshare-s3-175c build
# flash the app only — preserves pairing + Wi-Fi:
python -m esptool --chip esp32s3 --port /dev/cu.usbmodemXXXX -b 460800 \
  write-flash 0x20000 build-muse-waveshare-s3-175c/muse-gadget.bin
```

Three rules that will save you an evening:

1. **Never run `tools/muse/board.sh`** — it wipes `managed_components`, which
   deletes the BSP and frontend patches. After *any* component refetch,
   re-apply the whole overlay.
2. **Always pass `-B build-muse-waveshare-s3-175c`** — the default build dir
   may hold another chip's config.
3. **Never `erase-flash`** after pairing — app-only flashes at `0x20000` keep
   the pairing and Wi-Fi credentials.

## Using it

- Pair once over BLE in the Muse app (Settings → Devices → Add Device), with
  **your own** Muse account — the gadget token is personal and is *not* in
  this repo.
- **"Hey Jarvis"** → live call. Say **"thank you"** to end it gracefully.
- Top button (PWR): tap = call, hold from idle = voice note, press in a call =
  hang up. ~10 s hold cuts power in hardware. Bottom button (BOOT), 5 s hold =
  setup reset (unpair + forget Wi-Fi).
- Wake detection runs while the screen is asleep **on USB power**; on battery
  with the screen asleep it does not (v1 limitation).

## Known limitations

- **Meta's realtime service ends calls itself** (`session_control stop`,
  often right after the second answer). The device sends no session
  configuration — end-of-turn and session lifetime are server-governed — so a
  follow-up sometimes needs a fresh "Hey Jarvis". The detector re-arms in
  under a second.
- The board's speaker is small; at max volume it's still modest.
- The avatar is a personal design baked into the firmware as frames —
  changing it means regenerating `hifi_frames.h` and reflashing.

## Credits & licenses

See [LICENSE-NOTICES.md](LICENSE-NOTICES.md). In short: the Muse Gadgets SDK
is Meta's (Apache-2.0 code; gadget tokens are personal/non-commercial under
Meta's terms — don't sell devices or share tokens). The wake module design
comes from the Hey-Muse project; the "Hey Jarvis" model is from ESPHome's
microWakeWord models (Apache-2.0, Kevin Ahrendt). This overlay's own changes
are shared for personal, non-commercial use.
