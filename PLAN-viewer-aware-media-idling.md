# Viewer-aware media idling and capture pacing

## Goal and status

Make Waymote inexpensive when nobody consumes media, without taking its HTTP,
input, or clipboard services offline. Reconnecting to a static desktop must
produce a fresh decodable frame without requiring mouse movement.

This is an implementation plan, not an implemented fix. On October 1, 2026,
`oxmgr stop waymote-hyprland` succeeded; `oxmgr status` reports stopped and a
process check found no remaining Waymote gateway, stream daemon, or ffmpeg.
Leave that service stopped until the user authorizes deployment.

Investigation used working-tree source at HEAD `9476b17`. Preserve the existing
uncommitted changes in `main.zig`, `gateway/main.go`, `gateway/main_test.go`, and
`gateway/examples/web/client.js`. They add encode-scale ceilings and remote
resize protection among other changes. This planning pass adds only this file.

## Findings and root cause

- The process chain was oxmgr → `waymote-gateway` → `waymote-streamd` → ffmpeg.
  The supervisor definition is `/home/shuv/.config/oxmgr/waymote-hyprland.toml`;
  `/home/shuv/.local/bin/waymote-hyprland-session` launches the gateway with
  10 FPS, 50% encoded scale, and resize disabled. Its restart policy is always.
  This explains service persistence, not the missing application idle behavior.
- ffmpeg PID 141757 began September 27 around 13:20 PDT. The last logged viewer
  disconnected September 27 at 13:20:21 PDT. No established connections existed
  on gateway port 8241 or tailnet port 8240 during investigation.
- Even without viewers, ffmpeg read approximately 2,024 MB of raw 4K BGRA input
  in six seconds, about 10.2 frames/s. A short CPU sample showed ffmpeg using
  32% of one core and streamd about 12%. These are samples, not idle guarantees.
  `/proc/PID/io` write counters are not proof of UDP traffic volume.
- `gateway/main.go:main` starts `runEncoder` unconditionally. That function
  launches a single persistent daemon and attaches its stdin to `inputBridge`.
- `main.zig:connect` starts audio when configured, starts the video worker, and
  immediately calls `requestFrame`. `submitFrame` always requests another frame.
  Subscriber counts never reach the daemon. Neither `hub.unsubscribe` nor
  `audioHub.unsubscribe` stops media production.
- `videoPublisher.publish` acknowledges the gateway's cached keyframe, not the
  presence of viewers. `waitForDamage` selects damage-based screencopy after
  this acknowledgement, but screen activity still causes capture and encoding
  with zero subscribers. A cache-ready flag cannot substitute for demand.
- `VideoEncoder.zig:startProcess` supplies `-re` to the rawvideo input.
  Capture has no explicit FPS deadline: `frameListener` → `submitFrame` →
  `requestFrame` runs as quickly as screencopy permits. The three-slot encoder
  pool bounds queued latency; it does not limit capture/copy work.
- The inherited stderr log contained about 23 MB / 213,000 lines, almost all
  ffmpeg `Resumed reading ... after a lag ...` warnings. `-re` measures synthetic
  rawvideo time against elapsed time; damage-driven gaps make it the wrong
  clock for this live producer. Remove it only after native pacing exists.
- The observed bitrate of 3072 kbps followed adaptive cuts from 6000 → 4800 →
  3840 → 3072 during the last connection. `qualityPolicy.resetBaseline` explicitly
  preserves quality across controller handoff. Carry-over is not the cause of
  idle encoding and is not changed by this plan.

## Decisions and scope

1. **Keep streamd alive; suspend its media resources.** Input and clipboard share
   this daemon. Lazily killing/relaunching all of `runEncoder` would interrupt
   control, clipboard state, and resize handling and complicate independent
   audio consumers.
2. **Track video and audio demand independently.** A successful `/stream`
   subscription demands video; an enabled `/audio` subscription demands audio.
   `/control`, `/healthz`, static assets, and disabled-audio sockets demand neither.
   Connected audio sockets count even when playback is locally muted: the SDK
   currently connects audio before the user enables playback. Changing that
   browser contract is outside scope.
3. **Idle immediately at startup; use a two-second last-subscriber grace period.**
   The grace is per medium, cancels on reconnect, and prevents connection flaps
   from constantly respawning encoders. All resources must be suspended within
   three seconds of the last subscriber leaving on a healthy local system.
   No new configuration knob is needed for the initial fix; inject time in tests.
4. **Gateway enables a proposed `--on-demand` streamd option.** Direct daemon
   invocations retain their current continuous-capture default, including raw
   Annex-B stdout mode. In on-demand mode, video and audio begin disabled.
5. **Native capture deadlines replace ffmpeg input pacing.** Maintain the raw
   `-framerate` setting and current no-B-frame, low-latency codec configuration.
   Keep the RTP/metadata one-frame mapping; do not change browser timestamps to
   synthetic rawvideo time.
6. **Use fresh media identities after actual suspension.** Restarted video must
   have a new nonzero generation and SSRC, cleared keyframe readiness, and an
   unconditional bootstrap capture. Preserve quality and configured ceilings.

In scope: demand accounting, independent media lifecycle, bounded cancellation,
fresh reconnect bootstrap, capture pacing, regression tests, and documentation.
Out of scope: hardware encoding, adaptive-quality redesign, authentication,
supervisor log rotation/config changes, changing monitor selection, browser API
or UI changes, commits, releases, and restarting the stopped deployment.

## Architecture and invariants

### Gateway demand coordinator

Proposed new files: `gateway/media_demand.go` and
`gateway/media_demand_test.go`. Neither exists yet. Put a small serialized demand
coordinator here rather than scattering timers through HTTP handlers.

- Retain current `hub` / `audioHub` membership as the authority. Register demand
  only after WebSocket acceptance and configuration succeed and subscription is
  installed; release it exactly once on every handler exit. Second subscribers
  must not spawn second encoders.
- Publish ordered membership revisions while changing membership, but perform
  pipe writes after releasing hub locks. Concurrent subscribes/unsubscribes and
  stale grace timers cannot turn media off after newer demand turns it on.
- Maintain latest desired video/audio state even if streamd stdin has not been
  attached yet. Replay it when `runEncoder` attaches the writer; startup viewers
  must not lose their wake request. Use a bounded latest-state mailbox, not an
  unbounded queue or pipe write inside a hub/controller lock.
- Stop and join the coordinator on gateway shutdown. Failed daemon writes are
  surfaced through existing fatal daemon supervision, not silently forgotten.
- Add cancellation/deadline handling for media WebSocket writes and a liveness
  check so a half-open silent connection cannot hold demand forever. Healthy
  static viewers stay subscribed; lack of encoded frames is not disconnection.

### Private daemon protocol

Modify existing `gateway/main.go`, `main.zig`, and `docs/protocol.md` together.
Names and wire additions below are proposed, not existing symbols.

- Add private control type 12 (`media demand`): 16 bytes, version 2, state and
  reserved bytes zero, desired media mask in `a` (video bit 0, audio bit 1),
  nonzero revision in `b`, final word zero. Validate mask, version, padding,
  revision ordering/wrap, duplicates, and on-demand mode explicitly.
- Add daemon event type 4 (`media state`) for applied transitions. Document its
  fixed payload before implementation: revision, active mask, video generation,
  video SSRC, and audio SSRC, each `u32`. Zero SSRC means inactive. A resumed
  process is announced before its first raw input / emitted media is admitted.
  An idle acknowledgement means the corresponding child has been reaped and
  capture/pending work canceled, not just that a pause request was received.
- Add encode/decode tests and reject type 12 in `validControlRecord`, as type 11
  is already rejected. Browsers cannot control demand by sending binary records.
- Keep browser video/audio headers and protocol version unchanged. Deploy the
  gateway and daemon as a matched pair; this private addition is not compatible
  with an old daemon that lacks `--on-demand`.

### Native media lifecycle

Modify `main.zig` and `VideoEncoder.zig`; preserve the existing three-slot worker
model, eventfds, pidfd supervision, and main-loop notification handshake.

- On-demand `connect` binds Wayland/input/clipboard and starts the idle worker,
  but requests no screencopy and launches neither ffmpeg child.
- Video enable resets capture baseline/readiness, advances generation after a
  real suspension, reserves a fresh SSRC, and requests a frame even on a static
  output. Duplicate enable is a no-op. `requestFrame`, `rearmCapture`, cursor
  changes, resize success/failure, generation changes, and `.ready` all obey
  demand; none may accidentally resurrect capture while paused.
- Video disable cancels any outstanding screencopy and scheduled deadline,
  drops pending frames, and asynchronously interrupts the worker. The worker
  owns stopping/reaping its ffmpeg child. Release pending notification waiters
  and handle pause during a partially written raw frame; that process cannot
  be reused after a truncated frame. Main-loop input/clipboard stays responsive.
- Suspend retry/backoff while disabled. Pause must also interrupt `writeFrame`
  polling and `postNotification` waits; the existing final `deinit` cancellation
  path is the exemplar, not a sufficient pause implementation by itself.
- Audio enable/disable operates independently. Add an intentional-stop path
  distinct from `reapAudioEncoder` crash recovery; clear its retry deadline,
  close pidfd/child handles, and gate `maybeRestartAudio` and
  `scheduleAudioRestart` on demand. Use bounded child termination, following
  video pidfd signaling rather than an unbounded wait. Supply a fresh explicit
  audio SSRC on restart so the gateway can identify its current stream.
- An encoder crash while demanded retains existing recovery behavior. A crash
  or expired retry while idle cannot spawn a child. Re-enable starts promptly
  rather than inheriting a 30-second retry delay.

### Cache, RTP, and notification boundaries

Lifecycle correctness includes transport state, not just process counts.

- On actual video suspension, atomically mark its cache unavailable and clear
  `hub.latestGOP`. No trailing packet may repopulate an idle cache. A reconnect
  after suspension subscribes without the old GOP and waits for the fresh IDR;
  extra viewers during an active epoch still use the current cached GOP.
- Reconcile ordered media-state events, outstanding metadata, and RTP admission
  in one receiver-owned lifecycle path. Filter old SSRC packets before they can
  call `takeFrameMetadata`. Reset assembler/timestamp-gap state and publisher
  readiness at the epoch boundary. Old UDP datagrams must never consume new
  metadata or block waiting for metadata discarded by pause.
- Because stdout and UDP are distinct transports, allow a bounded staging area
  for a new SSRC whose lifecycle event is not processed yet. Do not merely drain
  UDP once and assume all old packets are gone. Give lifecycle events priority
  when resolving staged packets and discard stale epochs explicitly.
- Declare a process's media identity before feeding it so first media can be
  correlated. Test first-ever startup, restart before any completed access unit,
  and suspend during an incomplete raw frame. Preserve the existing restart
  generation handshake for genuine encoder failures and quality changes.
- Audio suspension similarly clears queued packets and resets tracker admission;
  the first resumed packet carries discontinuity. Preserve sequence/generation
  behavior across the long idle and the independent video transition.

### Native capture pacing

- Use `CLOCK_MONOTONIC` deadlines in `main.zig:run`, combining the next eligible
  capture time with `audioRestartTimeout` when choosing poll timeout. Request at
  most one outstanding frame and schedule the next request no sooner than one
  current FPS interval after the preceding ready frame.
- Damage waiting remains after keyframe readiness. A late frame or hours-long
  idle resets the next deadline from current time; never accumulate catch-up
  tokens or a burst of missed frames. Resume bootstrap is eligible immediately.
- Recompute/cancel deadlines on quality changes, resize, cursor rearm, and pause.
  Control/clipboard events must still dispatch while waiting for a frame deadline.
- After deadline tests pass, remove `-re` from `VideoEncoder.startProcess`.
  Verify ffmpeg still emits exactly one access unit per submitted complete frame
  (no automatic CFR duplication/dropping). Add a passthrough option only if an
  actual supported-FFmpeg test demonstrates it is necessary.
- Preserve rawvideo RTP tick increments used by `metadataCountForTimestampGap`
  and real capture timestamps used by the browser. Avoid hiding warnings with
  `-loglevel error`: genuine codec, pipe, and crash failures remain visible.

## Ordered implementation milestones

### 1. Define demand state and protocol with tests

Create the two proposed Go demand files. Modify `gateway/main.go`,
`gateway/main_test.go`, `main.zig`, and `docs/protocol.md` for private records and
on-demand argument parsing. Use injectable clock/timer and writer seams.

Acceptance: independent reference counts; startup desired-state replay; ordered
0→1→2→1→0 transitions; duplicate releases; stale timer cancellation; shutdown;
disabled audio and control-only connections; invalid private records rejected
both natively and at the public endpoint. At this milestone no live deployment
or global supervisor edits are needed.

### 2. Implement suspend/resume and fresh-media boundaries

Modify `main.zig`, `VideoEncoder.zig`, and `gateway/main.go`; wire demand through
`stream`, `streamAudio`, `runEncoder`, and `receiveStreamEvents`. Extend existing
tests in `gateway/main_test.go` and inline Zig tests; use a fake encoder process
to exercise blocked writes, notifications, stop/reap, and restart counts.

Acceptance: zero media children at on-demand startup; independent audio/video
pause; input/clipboard remain usable while both are idle; no child survives a
completed idle transition; fresh IDR on a motionless reconnect; old SSRC packets
cannot recache stale media or steal new metadata; two viewers share one encoder;
disconnecting one of them does not stop it. Existing standalone behavior stays.

Current test exemplars: `TestVideoPublisherTracksCachedKeyframeReadiness`,
`TestHubBootstrapsIdleSubscriberFromLatestGOP`,
`TestTakeFrameMetadataDiscardsOldGenerationAfterEncoderRestart`,
`TestAudioHubDropsQueuedLatencyAndMarksDiscontinuity`, and
`TestControlConnectionsHandOffLeaseWithoutReconnect`. Preserve bootstrap while
active, but extend its tests to distinguish an actually suspended epoch.
The inline Zig test `new captures replace only the pending encoder frame` and
the keyframe-readiness tests are the native exemplars.

### 3. Replace input pacing with capture deadlines

Modify `main.zig` and `VideoEncoder.zig`; add deterministic deadline tests and
argument-construction coverage. Exercise 10/30/60 FPS, time jumps, changing FPS,
static damage waits, cursor rearm, resize failure, and input during the deadline.

Acceptance: no `-re`; no catch-up burst after a long gap; capture requests limited
to the configured FPS within one scheduling/frame tolerance; one-to-one
submitted-frame/RTP-metadata correlation; genuine errors still logged.

### 4. Add repeatable integration coverage and document behavior

Proposed new integration test: `gateway/media_lifecycle_test.go`, using the
existing `httptest` + `websocket.Dial` pattern and an explicit fake daemon/encoder
seam. Proposed optional live harness: `scripts/check-media-idle` (new), owned by
this task if automated Wayland coverage needs it. It must run an isolated gateway
on an ephemeral loopback port, clean up only its own process group, and never
start the stopped oxmgr service or reconfigure the user's displays.

Modify `README.md` with a short product-facing idle behavior description and
`AGENTS.md` with validation/deployment instructions; finish `docs/protocol.md`.
No SDK or example UI changes are expected; run existing SDK tests for regression.

Acceptance: repeat 100 connect/disconnect cycles with no leaked children, fds,
goroutines, or timers; daemon unavailable/startup race tested; control-only and
audio-only sessions tested; recovery from demanded ffmpeg crashes still works;
shutdown with pending media writes cannot hang.

## Validation commands and expected signals

These checks are for the executing agent; they were not run by this planning
pass. Run narrow tests during milestones, then all checks below from the repo.

```sh
# Go lifecycle, race, and existing protocol coverage
(cd gateway && go test -race ./...)
(cd gateway && go vet ./...)

# Full supported native/gateway build and test targets
zig build test -Doptimize=ReleaseSafe
zig build install -Doptimize=ReleaseSafe

# Existing browser regressions (no browser API changes intended)
node --test gateway/sdk/waymote.test.mjs
```

Expected: all tests and vet exit zero; both binaries build; no races. Use the
documented Zig 0.16.0 and Go 1.26.5 toolchains for release validation. This host
currently has Zig 0.16.0, Go 1.27.1-X:nodwarf5, and FFmpeg n9.0.2; passing only
with its newer Go is not evidence of supported-toolchain validation.

Live validation requires a compatible Wayland session (or the documented Labwc
demo), libx264, and a Pulse monitor for audio cases. Use an isolated supervised
demo where available, or a non-resizing local gateway. Record:

1. Start on-demand with no clients; wait 30 seconds. `/healthz` returns `ok`,
   input/clipboard daemon stays alive, no video/audio ffmpeg child appears.
2. Connect a real WebCodecs browser to a static desktop: fresh IDR displays within
   two seconds on local loopback, without input. Connect a second viewer and
   verify one video encoder; close one and confirm uninterrupted frames.
3. Close the final viewer; within three seconds the video encoder disappears,
   capture requests cease, and no video RTP is sent. Observe for another minute
   with local screen activity; it must not resurrect media. After settling,
   gateway + streamd should consume less than 1% of one core averaged over a
   minute on this host; record actual samples and investigate exceptions.
4. Keep a controller open without media: key input, release-all, clipboard,
   ping/pong, and allowed resize handling still work; media stays idle.
5. Reconnect during the grace interval, then after one minute fully idle. Verify
   latest screen contents, correct dimensions, new epoch after real suspension,
   decoder discontinuity, no old GOP flash, and no startup metadata deadlock.
6. With Pulse configured, test audio-only, video-only, and both, then drop each
   independently. Disabled-audio sockets must never spawn an encoder. Resumed
   audio resets decoding cleanly and remains synchronized to capture time.
7. Animate the desktop at 10/30/60 FPS; measure capture rate, CPU, frame sequence,
   latency, and log size. Insert a 60-second no-damage interval then change the
   screen. No `Resumed reading` warning or backlog/catch-up burst should occur.
8. Kill a test-owned demanded encoder, then disconnect during retry; verify
   demanded recovery and idle cancellation. Stop the test gateway and confirm
   its complete descendant process group is gone.

Collect evidence from process trees, lifecycle events, frame metadata, and UDP
measurement. Do not infer network bytes from `/proc/PID/io:wchar`. Log only state
transitions and meaningful failures, not frames or per-second idle messages.

## Risks, dependencies, and rollback

- **Deadlock risk:** the worker waits for main-loop metadata completion while
  the main loop may be asked to pause it. Pause must wake that wait without a
  blocking join inside a handshake. Test all notification/write phases.
- **Transport race:** killing a child is not a packet/metadata barrier. Match
  admitted SSRCs to applied lifecycle state and test late old datagrams.
- **Reconnection race:** cache invalidation, subscription insertion, timer expiry,
  and readiness acknowledgements must have a defined serialization order.
- **Pacing risk:** simply deleting `-re` can encode at compositor refresh rate.
  Keep the pacing milestone and mapping regression tests mandatory.
- **Protocol dependency:** paired gateway/daemon rollout is required; validate
  unknown option/protocol failure is visible rather than idle forever.
- **Audio termination risk:** existing `Child.kill` can wait; use bounded pidfd
  supervision and intentional-stop semantics before asserting idling works.
- **Deployment:** build/test in isolation first. After separate user authorization,
  deploy both binaries together while oxmgr is stopped and retain the previous
  pair. The external launch script/config need no behavioral change because the
  gateway opts into on-demand mode itself.
- **Rollback:** stop the affected supervised app, restore the previous matched
  binaries from that backup, and leave it stopped by default. Restart only if
  explicitly authorized, since the old pair reintroduces idle encoding. Keep
  existing working-tree edits intact; no blanket git reset/revert.

## Unresolved choices

No user decision blocks this plan. Proposed record/event layouts, deadline and
coordinator symbols, fake-process seam, and integration helper paths must be
finalized and tested in milestone 1; their required behavior is fixed above.
If a supported compositor cannot safely cancel an outstanding screencopy, stop
implementation at that seam and report the observed protocol failure before
choosing a different daemon-lifecycle design.
