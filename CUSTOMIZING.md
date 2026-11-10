# Customizing: wake word, avatar, voice

Everything below is written for the agent doing the work. All paths are
relative to the SDK tree after the overlay is applied.

## Voice — nothing to change in firmware

The puck streams audio to **your own Muse account** over the realtime
service; the speaking voice is whatever voice your Muse is configured
with. The firmware sends no session configuration at all (verified in
`components/muse/muse_voice_call.cpp` and `muse_voice_call_core.c`: no
`session.update`, no voice, no instructions). To change how the device
sounds, change your Muse's voice in the Muse app — the device picks it
up automatically because it is paired to your account, not to a voice.

Server-side behavior that is also **not** firmware-controlled: end-of-turn
detection (~2 s), session lifetime, and the service ending a call itself
(`session_control stop`). The device-side mitigations for that are the
wake-session policy in `muse_voice.c` (see README).

## Wake word — swap the microWakeWord model

The detector is microWakeWord running in `components/muse/muse_wake.cpp`:

- Model: `components/muse/models/hey_jarvis.tflite`, embedded by
  `components/muse/CMakeLists.txt`; `muse_wake.cpp` reads it through the
  `_binary_hey_jarvis_tflite_start` symbol.
- Manifest: `components/muse/models/hey_jarvis.json` — its
  `probability_cutoff` (0.97) and `sliding_window_size` (5) are the values
  compiled into `muse_wake.cpp` (`PROBABILITY_CUTOFF`,
  `PROBABILITY_WINDOW`).

To change the wake phrase:

1. Get another microWakeWord-format `.tflite` — ESPHome's
   `esphome/micro-wake-word-models` repo publishes several (e.g.
   `okay_nabu`, `hey_mycroft`, `alexa`, `hey_luna`), or train your own
   with the microWakeWord training pipeline.
2. Easiest path: replace the **contents** of `models/hey_jarvis.tflite`
   with the new model, keeping the filename. No code changes needed.
   If you rename the file instead, update the embed path in
   `CMakeLists.txt` and the `_binary_<name>_tflite_start` symbol in
   `muse_wake.cpp` to match.
3. Copy the new model's `probability_cutoff` and `sliding_window_size`
   from its manifest JSON into `muse_wake.cpp`, and check its
   `tensor_arena_size`: `TENSOR_ARENA_SIZE` is 24 KB, sized for
   hey_jarvis (22,860 B). Raise it if the new model needs more.
4. The end phrases ("thank you", "that's all", "i'm done", "goodbye",
   "good night") and the session timeouts live in `muse_voice.c`
   (`jarvis_end_phrase()` and the wake-session policy in `run_call()`) —
   they are wake-phrase agnostic; edit there if you want different
   closers.

## Avatar image — regenerate `hifi_frames.h`

Two avatar systems ship; the hi-fi one is active
(`MUSE_HIFI_AVATAR 1` in `components/muse/muse_ui.c`).

The hi-fi portrait is five embedded frames in
`components/muse/hifi_frames.h`: `hifi_frame_base`, `hifi_frame_blink`,
`hifi_frame_wide`, `hifi_frame_pulse`, `hifi_frame_spk1` — each a
`static const uint16_t[]` of 384×384 RGB565 pixels, row-major
(`HIFI_FW 384`). `muse_ui.c`'s frame state machine drives them:
base = idle, blink = ~120 ms every ~3.7 s, wide = listening,
pulse = thinking / idle glow, spk1 = speaking (audio level > 0.35).
Redraws are region-only (eyes / chin / core rectangles).

To use your own artwork:

1. Start from a square image with the subject centered — the display is
   round, so corners are cropped by the bezel.
2. Produce the five variants of the same portrait: eyelids closed
   (blink), alert/wide eyes (wide), brighter core glow (pulse), and a
   speaking variant (spk1 — this character has no mouth, so the chin
   slit and core carry the speaking motion; design yours accordingly).
3. Resize each variant to 384×384, convert to RGB565, and emit C arrays
   with the **exact same symbol names** into a new `hifi_frames.h`.
   Any image library does this (e.g. PIL: `resize((384,384))`, then
   `((r & 0xF8) << 8) | ((g & 0xFC) << 3) | (b >> 3)` per pixel).
4. Mind the size budget: each frame is ~1 MB as C text and the whole
   firmware must fit the 4 MB app partition — the current build has
   ~252 KiB spare. Keep five frames or fewer at 384×384, or shrink
   `HIFI_FW` (and the region coordinates in `muse_ui.c`) together.
5. Do not add periodic full-canvas redraws while changing the state
   machine — on this board the display budget *is* the audio budget
   (a full-canvas invalidate every frame measurably starved call audio).

Alternative: set `MUSE_HIFI_AVATAR` to 0 and customize the procedural
pixel avatar in `components/muse/avatar/muse_pixel.c` instead. Its grid
is capped at 128 (`MUSE_PX_W/H` in `muse_pixel.h`) — the screen map
stores the cell index in a `uint8` with `0x80` stolen as an edge flag.

## One warning that applies to all three

However you customize it, keep the memory fix intact: the wake frontend
must stay in PSRAM (the `USE_ESP32` define in
`managed_components/esphome__esp-micro-speech-features/CMakeLists.txt`)
and the detector must stay resident during calls. Moving the frontend
back into internal RAM re-creates the starvation and heap-fragmentation
failures this build was hardened against — see AGENTS.md.
