# License notices

This repository is an overlay of changes on top of other people's work.
It is shared for **personal, non-commercial** use.

- **Muse Gadgets SDK** — © Meta Platforms, Inc. Code under the Apache
  License 2.0. Source: https://github.com/facebookincubator/muse-gadget-sdk
  (pinned at commit `463bd2a`). Muse gadget access tokens are personal and
  non-commercial under Meta's terms: do not sell devices running this, do
  not share or publish tokens. Nothing in this repo contains a token.
- **Wake-word module design** (`muse_wake.cpp/.h`) — adapted from
  **Hey-Muse** by leungcheukfai (a wake-word implementation for this same
  board), retargeted from "Hey Muse" to "Hey Jarvis" with different
  model parameters. https://github.com/leungcheukfai/Hey-Muse
- **"Hey Jarvis" wake model** (`models/hey_jarvis.tflite`) — from ESPHome's
  **microWakeWord models** (esphome/micro-wake-word-models), © Kevin
  Ahrendt, Apache License 2.0.
- **Audio frontend** (the `esp-micro-speech-features` component whose
  CMakeLists this overlay adjusts) — ESPHome port of the TensorFlow Lite
  Micro microfrontend, Apache License 2.0.
- **Board support package** (the Waveshare 1.75C component whose driver
  this overlay patches) — Waveshare / Espressif component-registry terms
  as published upstream.
- **Avatar artwork** — frames in `hifi_frames.h` are derived from the
  device owner's own avatar artwork; shared here as part of the project
  for personal use.
- The overlay's own original changes (session policy, settings clamps,
  hi-fi avatar integration, build fixes) are offered under the same
  personal, non-commercial terms as the SDK they extend.
