# AirPass: Design Doc

A simple iPad app that takes a hardware audio input (e.g. a turntable via USB) and broadcasts it to AirPlay destinations such as Sonos speakers. Replacement for Aircord 2, which is buggy.

## Goals

- High quality, low overhead audio passthrough from a wired input to AirPlay.
- Stable, predictable behaviour where Aircord 2 falls short.
- Continues running in the background.

## Non-goals (v1)

- No effects, EQ, or any signal processing. Pure passthrough.
- No Bluetooth output (planned for v2).
- No multi-channel routing, recording, or monitoring features.
- No multiple input handling. One input at a time.

## User flow

1. User connects audio source to iPad (USB-C audio interface, lightning adapter, etc).
1. Opens AirPass.
1. App detects the connected input and shows it.
1. User taps AirPlay picker, selects Sonos (or other AirPlay device).
1. User taps Start. Audio begins flowing.
1. User can lock the iPad or switch apps. Audio keeps playing.

## Architecture

### Audio pipeline

```
[Input device] -> AVAudioInputNode -> AVAudioMixerNode -> Output (AirPlay route)
```

Built on AVAudioEngine. Single chain, no intermediate processing nodes.

### Components

**AudioEngineManager** (ObservableObject)

- Owns the AVAudioEngine instance.
- Configures AVAudioSession (category, mode, options).
- Starts and stops the engine.
- Exposes published state: `isRunning`, `currentInput`, `currentRoute`, `error`.
- Handles route changes and interruption notifications.

**ContentView** (SwiftUI)

- Displays current input device.
- Embeds AVRoutePickerView for AirPlay destination selection.
- Start / Stop button.
- Status indicator (running, idle, error).

### Audio session configuration

```swift
let session = AVAudioSession.sharedInstance()
try session.setCategory(
    .playAndRecord,
    mode: .default,
    options: [.allowAirPlay, .defaultToSpeaker, .mixWithOthers]
)
try session.setPreferredSampleRate(48_000)
try session.setPreferredIOBufferDuration(0.005) // ~256 samples at 48kHz
try session.setActive(true)
```

Notes:

- `.playAndRecord` is required for any input usage.
- `.allowAirPlay` enables non-mirrored AirPlay output (iOS 10+).
- Small buffer duration favours latency over CPU headroom. Acceptable for passthrough.

### Background audio

Enable Background Modes capability in Xcode with "Audio, AirPlay, and Picture in Picture" checked. With the session category above, audio continues when the app is backgrounded or the screen locks.

### Interruption and route change handling

Observe two notifications:

- `AVAudioSession.interruptionNotification`: pause on `.began`, attempt resume on `.ended` with `.shouldResume` option.
- `AVAudioSession.routeChangeNotification`: log the reason, restart the engine if the new route is incompatible (e.g. input removed).

## Key design decisions

- **Swift native, no Rust.** AVAudioEngine and AVRoutePickerView integration is tight and well-documented. Rust via FFI adds friction for no real benefit in v1.
- **AVAudioEngine over raw Core Audio.** Sufficient for passthrough, far less ceremony. Drop down to Audio Units only if latency becomes a real issue.
- **Single input model.** If a new input is plugged in, switch to it. No UI for selecting between multiple inputs in v1.
- **No persistence.** App state resets each launch. Add session memory only if it becomes annoying.

## Open questions

- Behaviour when the input device is unplugged mid-session: stop cleanly, or wait for reconnect?
- Should the app warn the user if no AirPlay destination is selected before they tap Start?
- Volume: leave it to the receiving device (Sonos), or expose a gain control in the app?

## Known gotchas

- USB audio input on iPad follows a "last plugged in" rule. If multiple audio peripherals are connected, behaviour can be surprising.
- `AVAudioEngine` input format must match the hardware format. Query `inputNode.inputFormat(forBus: 0)` and connect using that format, do not assume 44.1kHz stereo.
- AirPlay route can become unavailable mid-stream (network blip, device sleeps). Engine may need restarting.
- iOS sometimes greys out AirPlay for apps using certain audio categories. `.playAndRecord` with `.allowAirPlay` is the combination that works.

## v2 candidates

- Bluetooth output (swap `.allowAirPlay` for `.allowBluetoothA2DP` or both, expose in route picker).
- Input selector when multiple inputs are connected.
- Optional gain control.
- Persisted preferences (last used input, last AirPlay destination).
- Lock screen and Control Center now-playing integration.
