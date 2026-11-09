# Ocarina of Time: the ocarina's hum mode panics the game

Program id `0400486024DEC000`. Reported symptom: selecting hum mode on the
ocarina during gameplay stops the game.

## Root cause

The guest aborts; the host does not. The Switch 2 audio entry point
`aud:u` is served by `INx2AudioUserClient`
(`src/core/hle/service/audio/nx2_audio_user_client.h`), which implements two
commands: 1 returns an `IAudioOutManager` and 2 an `IAudioRendererManager`.
There is no command for the audio **input** manager — the header already
includes `audio_in_manager.h` without using it.

Unknown commands on that interface are auto-stubbed, because
`ReportUnimplementedFunction` in `src/core/hle/service/service.cpp` stubs every
service whose name starts with `nx2:` (and every caller process that
`IsSwitch2()`). The stub replies `ResultSuccess` followed by zeroed words —
success, but no session handle for the object the caller asked for. The SDK then
uses that handle, finds nothing behind it, and aborts the title.

So hum mode asks `aud:u` for the capture manager, is told it succeeded, and gets
an empty object. The fix is to wire that command to `IAudioInManager`, the way
commands 1 and 2 are already wired.

### Corroboration

`Frizti/Fritchii` PR #16, "Switch 2 microphone capture on v4.1 (ocarina panic
C8099)", reports the same thing from the other side: on the official v4.1
release the ocarina stops the game with `Userspace PANIC C8099` with either a
webcam or a DualSense microphone, and the patch adds a Switch 2 capture path
plus silent capture when the host microphone cannot be opened. That PR is open,
targets `hotfix/v4.1` rather than `nx2`, cherry-picks two commits from a
repository this branch cannot reach, and its author states that tests were not
built or run and that playing a song on the ocarina was never confirmed. Treat
its claims as a second report, not as a verified fix.

### Still missing

The command index. The log names it: the first unknown command on a service is
reported once with its number and command buffer, as

```
Unknown / unimplemented function '<N>(<unknown>)': port='nx2:aud:u' cmd_buf={...}
```

Capture it with `EDEN_NX2_IPC_TRACE=1` and the filter
`*:Info Service_Audio:Debug`:

```
bash C:/lx/build-fix.sh <branch> oot-hum-trace "Ocarina of Time - hum mode trace" \
  "EDEN_NX2_IPC_TRACE=1" "*:Info Service_Audio:Debug"
```

Wiring a guessed index would send a build out for nothing, so the handler waits
on that line.

## Fixed here: the capture sink

With the manager in place the microphone still has to open, and two defects on
that path stopped it. Both were inherited from upstream, which has the same code
today, and both are corrected in `src/audio_core/sink/cubeb_sink.cpp`:

- `CubebSink::AcquireSinkStream` passed `device_channels` to capture streams.
  That value comes from `cubeb_get_max_channel_count`, which reports the
  *output* device, so on a 5.1 or 7.1 output the microphone was opened with six
  channels and `CUBEB_LAYOUT_3F2_LFE`. `cubeb_stream_init` rejects that for a
  mono or stereo microphone, `stream_backend` stayed null, and the game read
  silence. Input streams now open with two channels, as the SDL3 sink already
  did.
- `CubebSink::CubebSink` only enumerated `CUBEB_DEVICE_TYPE_OUTPUT`, so a
  microphone chosen in the UI (`audio_input_device_id`) was looked up among
  output devices, never found, and `input_device` was left unset. Both device
  types are enumerated now, disabled devices are skipped, and the name is
  assigned to the side it was found on.

`ListCubebSinkDevices` also called `CoUninitialize()` before using cubeb, so the
microphone list in the settings UI enumerated WASAPI devices and destroyed the
cubeb context on a thread whose COM had already been torn down. COM now outlives
`cubeb_destroy`.

None of these three changes is compile-tested: this container has no cubeb
headers and no configured build directory.

## Not the cause

- An unlowered shader instruction. Shader exceptions are caught in
  `vk_pipeline_cache.cpp` and the program is rejected, which shows as a wrong
  image, not a stop.
- The microphone release path. Upstream's fix (Eglantine `84e11a52ba`, eden PR
  #3336) is already present in `src/audio_core/sink/sink_stream.cpp`.

## Upstream gap, unrelated to this bug but outstanding

`nx2` forked from `upstream/nx2` at `a666e4400b` (2026-11-04); upstream's tip is
2026-11-08 (`0.1.0-rc3`) and carries Ocarina of Time work this branch lacks,
including `c6d2acd37a` ("nx2: Ocarina of Time stability, shader and performance
fixes", PR #794) and the later shader lowering in `9a69d8eaf0`, `900ab5a825`,
`de4ed9d9d7`, `031e07436c` and `a51a69fbfb`. Upstream has no microphone or
silent-capture fix, so there is nothing to port for this bug.
