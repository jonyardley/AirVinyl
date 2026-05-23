# AirPass

iPad app that passes a wired audio input (USB-C interface, turntable, etc.) to an AirPlay receiver such as a Sonos speaker. Replacement for Aircord 2.

- Design: [DESIGN.md](DESIGN.md)
- Conventions and gotchas: [CLAUDE.md](CLAUDE.md)

## Requirements

- Xcode 16+
- iPadOS 17+ deployment target
- `swift-format` (ships with Xcode 16)

## Build & test

```bash
xcodebuild -scheme AirPass \
  -destination 'platform=iOS Simulator,name=iPad Pro 13-inch (M4)' build

xcodebuild test -scheme AirPass \
  -destination 'platform=iOS Simulator,name=iPad Pro 13-inch (M4)' \
  -only-testing:AirPassTests

swift-format lint --recursive --strict AirPass AirPassTests
swift-format format --in-place --recursive AirPass AirPassTests
```

## Bootstrap (one-time, manual in Xcode)

The repo currently has docs and tooling only — no `.xcodeproj` yet. To create it:

1. **File → New → Project → iOS → App**
   - Product name: `AirPass`
   - Interface: SwiftUI, Language: Swift
   - Include Tests checked (creates `AirPassTests` and `AirPassUITests`)
   - Save into this repo root (will create `AirPass.xcodeproj` alongside `AirPass/` source folder)
2. **Target settings → General**
   - Deployment target: iPadOS 17.0
   - Supported destinations: iPad only (remove iPhone, Mac)
   - Bundle identifier: pick something stable (e.g. `com.<you>.airpass`)
3. **Signing & Capabilities**
   - Add **Background Modes** → check **Audio, AirPlay, and Picture in Picture**
   - Automatic signing with your personal team for device runs
4. **Info.plist additions**
   - `NSMicrophoneUsageDescription` — copy from [CLAUDE.md](CLAUDE.md)
5. **Privacy manifest**
   - Add a `PrivacyInfo.xcprivacy` file (empty arrays initially) — required for TestFlight
6. **Rearrange source folders** to match the layout in [CLAUDE.md](CLAUDE.md):

   ```text
   AirPass/
     AirPassApp.swift
     Audio/
     Views/
     Resources/
   ```

7. **Scheme**: make sure `AirPass` scheme is shared (Manage Schemes → Shared) so CI can find it.

After committing the project, CI will start running build + test automatically.

## CI

GitHub Actions runs `swift-format lint --strict` and `xcodebuild test` on every PR. See [.github/workflows/ci.yml](.github/workflows/ci.yml). Lint and test steps each no-op until the corresponding source/project exists, so this branch is green before bootstrap.

## Branching

- Never push to `main`. Feature branches + PRs.
- Set `main` as the default branch and require the CI checks before merge.

## Device verification

Audio-pipeline changes require physical-iPad verification. See "Testing on device" in [CLAUDE.md](CLAUDE.md). PR descriptions must state `device-verified: yes/no`.
