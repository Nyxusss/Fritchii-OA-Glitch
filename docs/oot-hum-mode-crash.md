# Ocarina of Time: crash when enabling the ocarina's hum mode

Program id `0400486024DEC000`. Reported symptom: the game crashes when hum mode
is selected on the ocarina during gameplay.

**Status: not root-caused.** No `eden_log.txt` was supplied with the report, so
the faulting frame is unknown. Everything below is what the tree shows; none of
it is confirmed as the cause, and no fix is written yet.

## What the report still needs

1. `eden_log.txt` and `eden_log.txt.old` from the session that crashed. The
   `Booting game:` line and the `Lichii Version: ... | nx2-<sha>` header pin
   the build.
2. Whether the log *ends* abruptly or ends with an error. A log that just stops
   means the host process died: the Windows minidump in
   `%LOCALAPPDATA%\CrashDumps\lichii.exe.<pid>.dmp` is then the primary
   evidence (read with the `minidump` package, symbolize against the build's
   `lichii.pdb`).
3. A run with `EDEN_NX2_IPC_TRACE=1` and the filter
   `*:Info Service_Audio:Debug`, to settle whether hum mode opens `audin:u` at
   all. That decides between the audio-capture hypothesis below and the GPU
   paths.

Build and package it on the owner's PC with:

```
bash C:/lx/build-fix.sh <branch> oot-hum-trace "Ocarina of Time - hum mode trace" \
  "EDEN_NX2_IPC_TRACE=1" "*:Info Service_Audio:Debug"
```

## Audio capture path, as it stands

If hum mode listens on a microphone it runs `audin:u` → `IAudioInManager`
(`src/core/hle/service/audio/audio_in_manager.cpp`) → `AudioIn::System`
(`src/audio_core/in/audio_in_system.cpp`) → `DeviceSession` →
`SinkStream::ProcessAudioIn`. That chain is equivalent to upstream; upstream's
microphone release-buffer fix (Eglantine `84e11a52ba`, eden PR #3336) is
already present in `src/audio_core/sink/sink_stream.cpp`.

Two real defects sit on that path. Both are inherited from upstream (upstream
`nx2` has the same code today), and neither is proven to crash — they make the
microphone deliver nothing:

- `CubebSink::CubebSink` takes `device_channels` from
  `cubeb_get_max_channel_count`, which reports the *output* device. The same
  class backs the input sink (`AudioCore::CreateSinks`), so on a 5.1 or 7.1
  output the capture stream is opened with 6 channels and
  `CUBEB_LAYOUT_3F2_LFE`. `cubeb_stream_init` rejects that for a mono or
  stereo microphone, `stream_backend` stays null, and the game reads silence.
  The SDL3 sink does not have this problem: it pins `device_channels = 2`.
- The same constructor only enumerates `CUBEB_DEVICE_TYPE_OUTPUT`, so a
  microphone picked in the UI (`audio_input_device_id`) is looked up among
  output devices, never found, and `input_device` is left unset.

## Upstream gap

`nx2` forked from `upstream/nx2` at `a666e4400b` (2026-11-04). Upstream's tip is
2026-11-08 (`0.1.0-rc3`) and carries Ocarina of Time work this branch does not,
including:

- `c6d2acd37a` — "nx2: Ocarina of Time stability, shader and performance
  fixes" (PR #794). Its crash list covers the VP9 movie race in the
  dirty-memory gather, deleted buffer ids per channel, unmapped GPU writes and
  address ranges, null addresses in compute reductions, render-target bounds,
  a bounded query-result wait, and the `clip_distance_written` compile race.
- `9a69d8eaf0`, `900ab5a825`, `de4ed9d9d7`, `031e07436c`, `a51a69fbfb` —
  further Ocarina shader lowering and buffer binding.

Per `AGENTS.md`, nx2 is merged with upstream before a new fix is written, and
an existing upstream fix is ported rather than rewritten. Note that an unlowered
instruction alone does not crash the host: shader exceptions are caught in
`vk_pipeline_cache.cpp` and the program is rejected, which shows as a wrong
image, not a termination.
