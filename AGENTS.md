# AGENTS.md — build protocol for an AI agent

You are building the "Hey Jarvis" nightstand firmware for your human.
Execute the steps in order. Do not improvise around the **HARD RULES** —
each one cost a debugging session.

## Target

- Board: Waveshare **ESP32-S3 Touch AMOLED 1.75C** (466×466, 8 MB PSRAM,
  32 MB flash). The non-C 1.75 is a different board; stop if that's what's
  in hand.
- Result: the device pairs to your human's Muse account, answers to
  "Hey Jarvis" with a live voice call, and shows the hi-fi avatar.

## HARD RULES

1. **Never run `tools/muse/board.sh`.** It deletes `managed_components`
   before and after building, wiping the patched BSP and the frontend
   CMake fix. Always invoke `idf.py` directly.
2. **Always pass `-B build-muse-waveshare-s3-175c`** to `idf.py`. The
   default `./build` directory can hold a different chip's config.
3. **Never `erase-flash`**, and never flash offset 0 on a paired device.
   App-only flashes go to **0x20000** and preserve pairing + Wi-Fi.
4. **Back up the factory flash before the first flash of anything:**
   `esptool --chip esp32s3 read-flash 0 0x2000000 factory-backup.bin`
   (33,554,432 bytes). That file is the full restore path.
5. The gadget token is your human's personal credential. It is provisioned
   during pairing and lives in the device's encrypted NVS. **Never ask for
   it in chat, never put it in a file or a repo.**
6. After **any** component fetch/refetch, re-apply the entire overlay —
   managed-component patches do not survive refetches.

## Prerequisites

- ESP-IDF **v6.0.3** exactly (other versions were not used; the SDK pins
  its expectations to it).
- A terminal on the build host with the board on USB. Discover the port,
  never hardcode it: `ls /dev/cu.usbmodem*` (macOS) — the name changes
  between replugs.
- Python for serial tools: a venv with `pyserial` + `esptool` 5.x works.

## Steps

### 1. Clone and pin the SDK

```bash
git clone https://github.com/facebookincubator/muse-gadget-sdk
cd muse-gadget-sdk
git checkout 463bd2a
```

### 2. Apply the overlay

Copy this repo's `overlay/esp32/` over the SDK's `esp32/`, preserving paths
(every file replaces or adds at the same relative path). This includes two
files under `managed_components/` — apply them *after* the first component
fetch (step 4's first build performs it) and again after any later refetch.

**One file is not stored in this repo** (it's a binary): fetch the wake
model from its canonical source and verify it before building —

```bash
curl -L -o esp32/components/muse/models/hey_jarvis.tflite \
  https://raw.githubusercontent.com/esphome/micro-wake-word-models/main/models/v2/hey_jarvis.tflite
# expect: 52,272 bytes, SHA-256 21a7976add39ee24ec96c63d96b7aaa18e24d1d9824b963e451da8feb4b78b77
sha256sum esp32/components/muse/models/hey_jarvis.tflite
```

### 3. Configure

Apply the values in `sdkconfig.defaults.jarvis` (append them to the
project's sdkconfig defaults or set them in menuconfig):

- `CONFIG_MUSE_REALTIME_VOICE=y` — live voice calls (the whole point).
- `CONFIG_HOMEHUB_NVS_ENCRYPTION=y`, `CONFIG_NVS_ENCRYPTION=y`,
  `CONFIG_NVS_SEC_HMAC_EFUSE_KEY_ID=5` — encrypted NVS for the token.
- Secure Boot and flash encryption stay **off**; pairing attestation off.

### 4. Build

On macOS the helper shell's `python3` may be the system 3.9, which breaks
ESP-IDF's `export.sh`. Put the ESP-IDF Python env first:

```bash
PYBIN=$(ls -d ~/.espressif/python_env/*/bin | head -1)
export PATH="$PYBIN:/opt/homebrew/bin:$PATH"
. ~/esp/esp-idf-v6/export.sh
cd esp32
idf.py -B build-muse-waveshare-s3-175c build
```

Expected: `muse-gadget.bin` ≈ **3,936,256 bytes** (fits the 4 MB app
partition with ~252 KiB spare). If the link overflows the partition, check
that the speaking-frame set matches this repo (a sixth frame doesn't fit).

### 5. Flash (app only)

```bash
PORT=$(ls /dev/cu.usbmodem* | head -1)
python -m esptool --chip esp32s3 --port "$PORT" -b 460800 \
  write-flash 0x20000 build-muse-waveshare-s3-175c/muse-gadget.bin
```

### 6. Pair

On the device: if it's not already offering pairing, hold the **bottom**
button 5 s (setup reset). In your human's Muse app: Settings → Devices →
Add Device, and complete the BLE pairing + Wi-Fi join with *their* account
and *their* 2.4 GHz network.

### 7. Verify from the boot log

Capture serial at 115200 from a cold boot (toggling DTR restarts it) and
check **all** of these before calling it done:

- Exactly **1** ESP-ROM banner; **0** Guru Meditation / panic lines.
- `audio ready: 16000 Hz, 2 ch` and the mic self-test line.
- Wake telemetry (three lines, internal free should barely move at the
  frontend stage — that's the PSRAM fix working):
  `wake init: frontend start, internal free ~31000` →
  `wake init: frontend up, internal free ~33000, psram free ~2040000` →
  `wake init: interpreter up, ...` and `Hey Jarvis model ready (arena 24576 bytes, stride 3)`.
- Hatch state **Connected**; heartbeat (`link.heartbeat`) shows wifi/ws up
  and, armed, roughly **int=31–34K, dma≥23K, psram≈1266–1684K**.
- Then the real test: your human says **"Hey Jarvis"**, asks something,
  hears a spoken reply, and says **"thank you"**.

### 8. If something's wrong

| Symptom | Likely cause |
|---|---|
| Chat session loops "Can't reach Muse's server" | Detector holding internal RAM (a pre-v8 build, or the overlay's frontend CMake fix was wiped by a refetch). Re-apply overlay; confirm `USE_ESP32` is in the frontend component's CMakeLists. |
| Calls connect but replies are silent | Speaker setting is off (the boot clamp in `muse_settings.c` should force it on — check the overlay file is present). |
| Audio degrades over uptime | Same memory story — verify the armed heartbeat numbers against step 7. |
| Mic crashes / boot loop after flash | BSP patch missing (ES7210 address / I2S format). Re-apply the managed-components overlay. |
| Link "Online" but hatch "Can't connect" | Stale device↔account binding. Re-pair in the Muse app; reflashing does not fix this. |
| Device won't wake on battery with screen asleep | Expected v1 limitation. On USB power it works screen-asleep. |

## Behavior spec (what you built)

- "Hey Jarvis" → hands-free live call (button-started calls unchanged).
- End phrases (matched on the device from the heard transcript): anything
  containing *thank*, *that's all*, *i'm done*, *goodbye*, *good night*.
  Plain "stop" is deliberately **not** an end phrase device-side.
- Wake-session timing: 5 s to produce first speech, 8 s idle after last
  activity, end ~1.2 s after an end-phrase reply plays.
- Stock limits still apply underneath: 45 s quiet / 15 min max for calls.
- Meta's realtime service may end the call first (`session_control stop`) —
  that's server policy, not a bug; the detector re-arms in <0.5 s.
- Settings clamps at boot: `speaker_on=true`; nonzero sleep floored to
  600 s (0 = never remains an explicit choice).
