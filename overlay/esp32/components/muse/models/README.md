# Wake-word model

`hey_jarvis.tflite` — the ESPHome microWakeWord v2 "Hey Jarvis" model
(Kevin Ahrendt), from https://github.com/esphome/micro-wake-word-models
(models/v2/), Apache-2.0. Manifest (`hey_jarvis.json`, kept alongside for
reference): probability_cutoff 0.97, feature_step_size 10 ms,
sliding_window_size 5, tensor_arena_size 22860.

Embedded into the firmware by CMake (`EMBED_FILES`) and consumed by
`muse_wake.cpp`. JARVIS PATCH 20261109.
