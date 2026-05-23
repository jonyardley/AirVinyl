# AirPass Development Guidelines

> Last reviewed: 2026-05-23.

## Project Overview

AirPass is an iPad app that takes a wired audio input (USB-C audio interface,
turntable via a class-compliant DAC, lightning-to-3.5mm, etc.) and broadcasts
it to an AirPlay destination such as a Sonos speaker. Pure passthrough —
no effects, no recording, no monitoring. Replacement for Aircord 2, which
is buggy.

See `DESIGN.md` for the v1 design doc, goals, non-goals, and open questions.

**Platform**: iPadOS 17+ only in v1. iPhone is out of scope until v2.

## Project Structure

```text
AirPass/
  AirPassApp.swift             # @main entry, app lifecycle
  Audio/
    AudioEngineManager.swift   # AVAudioEngine + AVAudioSession owner
    AudioSessionConfig.swift   # Category/mode/options, sample rate, buffer
    RouteObserver.swift        # Interruption + route-change notifications
  Views/
    ContentView.swift          # Single screen: input + AirPlay picker + Start/Stop
    RoutePickerView.swift      # UIViewRepresentable for AVRoutePickerView
    StatusIndicator.swift      # Idle / Running / Error pill
  Resources/
    Info.plist                 # NSMicrophoneUsageDescription, UIBackgroundModes
AirPassTests/                  # XCTest target
AirPassUITests/                # XCUITest target (manual-run only — see Testing)
DESIGN.md                      # v1 design doc
```

Keep the audio layer behind a thin protocol that views consume via
`@StateObject` / `@Observable`. Views never touch `AVAudioEngine` or
`AVAudioSession` directly.

## Tech Stack

- **Swift** 5.10+, **Xcode** 16+
- **UI**: SwiftUI, iPadOS 17+
- **Audio**: AVFoundation (`AVAudioEngine`, `AVAudioSession`, `AVAudioInputNode`,
  `AVAudioMixerNode`), `AVRoutePickerView` (UIKit, bridged via `UIViewRepresentable`)
- **Testing**: XCTest + XCUITest
- **Lint/format**: `swift-format` (Apple's, ships with Xcode 16) — `.swift-format`
  config at repo root

## Commands

```bash
# Build
xcodebuild -scheme AirPass -destination 'platform=iOS Simulator,name=iPad Pro 13-inch (M4)' build

# Unit tests
xcodebuild test -scheme AirPass -destination 'platform=iOS Simulator,name=iPad Pro 13-inch (M4)' -only-testing:AirPassTests

# Format check — must pass before push
swift-format lint --recursive --strict AirPass AirPassTests

# Format in place
swift-format format --in-place --recursive AirPass AirPassTests
```

Run `swift-format lint --strict` and the unit-test target *locally before
pushing*, not just before committing — CI runs both and a wasted round trip
is ~3 minutes. The simulator-only build is what CI exercises; the device
build matters for real audio verification (see Testing).

## Architecture (Non-Negotiables)

### One engine, one session, one owner

`AudioEngineManager` is the only type that touches `AVAudioEngine` or
`AVAudioSession.sharedInstance()`. It's an `@MainActor` `ObservableObject`
that publishes `isRunning`, `currentInput`, `currentRoute`, and `error`.
Views observe; views never instantiate engines or call `setActive(true)`.

```text
User intent (Start/Stop) → AudioEngineManager → AVAudioEngine → AirPlay route
                                ↓
                          @Published state → SwiftUI views
```

### Audio pipeline

```text
inputNode (hardware format) → mixerNode → mainMixerNode → outputNode (AirPlay)
```

Single chain. No tap blocks, no `installTap(onBus:)` in v1 — we are not
inspecting samples. If a feature ever needs a tap, it goes through a new
abstraction; do not sprinkle tap installs across the codebase.

### State boundary

| State kind | Where it lives |
|------------|---------------|
| Engine state (running, route, input) | `AudioEngineManager` `@Published` |
| Transient UI (sheet presented, animations) | View-local `@State` |
| User preferences (v2) | `UserDefaults` via a `Preferences` actor |

No domain state in views. No `AVAudioSession` polling from views — observe
the manager's published properties.

### Threading

- Audio render thread is real-time. **Never** allocate, lock, log, or call
  Objective-C runtime methods from a tap or render callback. v1 has none;
  if you add one, treat it like signal handler code.
- `AudioEngineManager` methods that mutate engine state run on `@MainActor`.
  Notification callbacks (`AVAudioSession.interruptionNotification`,
  `routeChangeNotification`) arrive on arbitrary threads — hop to main
  before mutating published state.
- `AVAudioEngine.start()` is synchronous and can block briefly. It's
  acceptable on main for v1 (single-shot at user tap); if it ever becomes
  a frame hitch, move to a background actor.

### Audio session configuration

Single source of truth: `AudioSessionConfig.apply()`.

```swift
let session = AVAudioSession.sharedInstance()
try session.setCategory(
    .playAndRecord,
    mode: .default,
    options: [.allowAirPlay, .defaultToSpeaker, .mixWithOthers]
)
try session.setPreferredSampleRate(48_000)
try session.setPreferredIOBufferDuration(0.005)
try session.setActive(true)
```

Do not call `setCategory` from anywhere else. If a feature needs a different
mode, add a variant to `AudioSessionConfig` rather than calling the session
API ad hoc.

### Input format

`AVAudioEngine` input format must match the hardware. Always query
`inputNode.inputFormat(forBus: 0)` and connect using that format. Never
hardcode 44.1 kHz / 2 channels — USB interfaces vary.

```swift
let format = engine.inputNode.inputFormat(forBus: 0)
engine.connect(engine.inputNode, to: engine.mainMixerNode, format: format)
```

## iOS-specific rules

### Info.plist requirements

- `NSMicrophoneUsageDescription` — required for `.playAndRecord`. Copy:
  "AirPass needs microphone access to read your wired audio input
  (turntable, audio interface) and stream it to AirPlay."
- `UIBackgroundModes` includes `audio` — required for background playback.
- `UIRequiresFullScreen` = `false` (Split View, Slide Over compatible).

### Background audio

The `audio` background mode plus `.playAndRecord` category keeps the engine
running when the screen locks or the user switches apps. Verify on a
physical device — simulator background behaviour does not match device.

### Interruption handling

Observe `AVAudioSession.interruptionNotification`:

- `.began`: stop the engine, set `isRunning = false`, surface a non-modal
  status. Do not tear down the session.
- `.ended` with `.shouldResume`: attempt `engine.start()`. If it throws,
  publish the error and leave the user to restart manually — do not retry
  in a loop.

### Route change handling

Observe `AVAudioSession.routeChangeNotification`. Reasons we care about:

- `.oldDeviceUnavailable` (input unplugged): stop cleanly, surface the
  reason. Do not silently restart on a different input.
- `.newDeviceAvailable` (input plugged in while idle): update
  `currentInput`. Do not auto-start.
- `.routeConfigurationChange`: usually benign; log only.

Restart the engine *only* if the new route is incompatible — e.g. the
output format changed underneath us. Reconnect with a freshly queried
input format.

### AirPlay specifics

- AirPlay route can vanish mid-stream (Wi-Fi blip, receiver sleep). Treat
  it as a route change; the engine may need a fresh `start()`.
- `.playAndRecord` + `.allowAirPlay` is the combination that lets the
  AirPlay picker offer non-mirrored routes. Other category/option combos
  silently grey out AirPlay; do not "simplify" the category.
- The simulator does not expose AirPlay routes. AirPlay verification is
  physical-device-only.

### Testing on device

Manual verification on a physical iPad with:
1. A USB-C class-compliant audio interface (any iCONNECTIVITY / Scarlett /
   Audient with the iPad camera kit or USB-C cable).
2. A Sonos speaker (or any AirPlay 2 receiver) on the same Wi-Fi.

Verify before declaring an audio change shipped:
- Audio flows from input to AirPlay receiver.
- Locking the iPad does not interrupt audio.
- Unplugging the input mid-stream surfaces a clean stop, not a crash.
- Switching AirPlay destination via the picker resumes within 2 seconds.

## Code Style

- Swift 5.10, strict concurrency where Xcode flags it. No `@unchecked Sendable`
  without a comment justifying why.
- `swift-format lint --strict` must pass.
- No force-unwraps (`!`) outside of `IBOutlet`-style guaranteed-set
  references — and we don't have any of those. Prefer `guard let` with a
  meaningful failure path.
- No `try!` in shipping code. Audio session APIs throw; surface the error
  through `@Published var error`.
- Prefer Apple frameworks over third-party for anything in the audio path.
  Stability matters more than convenience here.

### Comments

Default to **no comments**. Self-explanatory code with well-named identifiers
beats commented code.

Add a comment only when one of these holds:

- **Non-obvious WHY** — a hidden constraint, subtle invariant, workaround
  for a specific bug, or framework quirk that would surprise a reader.
  Cite the reason concretely: a radar number, a doc link, a `BUG:` tag.
  Audio code attracts these — "iOS 17.2 returns the wrong format here,
  rdar://..." is a comment that earns its keep.
- **Cross-file context** — pointing at a related file, a CLAUDE.md rule,
  or an external doc the reader could miss. One line max.
- **Section structure** — single-line dividers like `// ── Route changes ──`
  in a long file. Never more than one line.

Do **not** write a comment that:

- Restates WHAT the code does (`// Start the engine` above `engine.start()`)
- References the current task / PR (`// Added for #42`) — rots, belongs in
  the PR description
- Apologises or hedges (`// quick fix`, `// TODO come back`) — open a
  tracked issue instead
- Is a `///` doc comment on a private function whose signature is
  self-evident

Two-line cap as a smell test: if a comment is more than two lines, ask
"can this be a function name? a type? a CLAUDE.md entry?". Usually yes.

## Testing

**Default: ship tests with new non-trivial logic.** Audio work has a large
manual-verification component (see iOS-specific rules → Testing on device),
but the pure logic around it is testable and should be tested.

What to unit test:
- Route-change reason → action mapping (a pure function fed by the
  notification's `userInfo`).
- Interruption begin/end → engine state transitions, via a mockable
  engine protocol.
- Input format queries and connection logic, given a fake `AVAudioFormat`.

What to manually verify (XCUITest is not enough — audio doesn't render
in the test runner):
- Anything in "Testing on device" above.

When skipping tests, say so explicitly in the PR description with the
reason (e.g. "behaviour only reproducible with real USB audio hardware").

## Project-specific gotchas

Bear-traps that have caught us at least once.

### `inputNode.inputFormat(forBus: 0)` returns the **current** hardware format

Query it *after* `AVAudioSession.setActive(true)` and *after* the input
device is connected. Querying earlier returns a default (often 44.1 kHz
stereo) that does not match the actual hardware, and `connect(_:to:format:)`
will silently produce no audio. If you change the input device mid-session,
re-query and reconnect.

### `.playAndRecord` is required even for output-only-feeling features

It feels wrong — we're "just sending audio out" — but `.allowAirPlay`
only works with categories that include record. `.playback` greys out
the AirPlay route. Do not "fix" the category to match intuition.

### Simulator lies about routes

The iOS Simulator does not enumerate real AirPlay devices and will not
hand off audio to one. Any AirPlay-touching change requires device
verification. The simulator is fine for SwiftUI, layout, and pure-logic
unit tests.

### USB audio on iPad follows a "last plugged in" wins rule

If two USB audio devices are connected, iOS picks the most recent one as
the input. There is no API to enumerate them. v1 accepts this; v2 may
expose a selector. Do not add code that assumes multi-device enumeration
is possible.

### `AVRoutePickerView` is UIKit-only

There is no SwiftUI-native AirPlay picker as of iPadOS 17. Wrap it in a
`UIViewRepresentable` (`RoutePickerView.swift`). Resist the urge to
"build our own" picker — Apple gates the route-selection sheet behind
this view, and rolling your own means re-implementing private API.

### Notification callbacks run on arbitrary threads

`AVAudioSession.interruptionNotification` and `routeChangeNotification`
may deliver on background threads. Hop to `@MainActor` before mutating
`@Published` state. Forgetting this surfaces as "view doesn't update
when I unplug the cable" — the publish happened, just not on a thread
SwiftUI was watching.

### Don't call `setActive(false)` on stop unless you mean it

Deactivating the session releases the audio route. If the user stops and
starts again within a few seconds, the AirPlay receiver may have to
re-negotiate. v1 keeps the session active across Stop and only deactivates
on app termination. Revisit if battery becomes an issue.

## Workflow

Match ceremony to scope. Default to less. Escalate only when work demands it.

### Tier 1 — Just do it
Copy/text tweaks, swift-format fixups, single-file refactors, dependency
bumps, doc updates, simple bug fixes that don't touch the audio pipeline.

No spec, no design doc update. Read enough to confirm, make the change,
verify, ship.

### Tier 2 — Plan first (default for feature work)
New view, new published property, new notification handler following an
existing pattern, settings screen, error-state UI.

Plan the approach in chat first (file paths, the shape of the change),
then implement. Update `DESIGN.md` if the change moves a v2 candidate
into v1 or vice versa.

### Tier 3 — Design doc update (rare; architectural)
Anything that changes the audio pipeline shape, the session category,
the threading model, or adds a second input/output mode. Anything that
moves a non-goal into scope (effects, recording, Bluetooth).

Update `DESIGN.md` *before* implementing. The doc and the code change
ship in the same PR — reviewers sanity-check intent against
implementation.

### Domain sensitivity override
Changes to `AudioSessionConfig`, the `AVAudioEngine` setup, interruption /
route-change handling, or `Info.plist` audio-related keys go up at least
one tier regardless of file count.

### Decision rule
If unsure between tiers, go one tier lighter. Drift up if scope expands.

### Always

1. Never push to main. Always a feature branch + PR.
2. Run `swift-format lint --strict` and `xcodebuild test` locally before
   pushing.
3. For any change touching the audio pipeline, verify on a physical iPad
   with a real input + real AirPlay receiver before marking the PR ready.
   State "device-verified: yes/no" in the PR description.
4. Self-review every non-trivial PR. For Tier 2+, post the review summary
   as a PR comment so it's visible alongside CI.
5. Open a tracked issue for every deferred / out-of-scope item. PR
   descriptions get auto-collapsed after merge; issues persist.

### After completing work

1. Update `DESIGN.md` if architecture, goals, or non-goals shifted.
2. Update this file if patterns or gotchas changed.
3. Close the GitHub issue.
