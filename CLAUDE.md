# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

VoiceInput is a macOS menu-bar application (Swift 5.9, macOS 14+) that converts speech to text in real time using Apple's on-device Speech Recognition framework. Hold Fn, speak, release — transcribed text is injected into the focused text field. The full source code lives at `https://github.com/yetone/voice-input-src`; this repo is the distributable app bundle artifact.

## Build / Run

```bash
make build      # compile release build → VoiceInput.app (ad-hoc codesigned)
make run        # build and launch
make install    # copy VoiceInput.app to /Applications
make clean      # delete .build/ directory and VoiceInput.app
```

There are no tests in this repo — the test suite (if any) lives in the source repo.

## Architecture

The app is a single executable target (`Sources/VoiceInput/`) with no external Swift package dependencies beyond the macOS SDK.

### Entry point → Coordinator pattern

`main.swift` creates an `.accessory` NSApplication (menu-bar only, no Dock icon) and hands control to `AppDelegate`. AppDelegate is the central coordinator — it wires together every subsystem and makes all flow-control decisions (when to record, when to refine, when to inject).

### Component map & data flow

```
Fn key held → KeyMonitor (CGEvent tap, flagsChanged)
    → AppDelegate.fnDown()
        → SpeechEngine.startRecording()     # AVAudioEngine + SFSpeechRecognizer
        → OverlayPanel.show("Listening...")  # floating NSPanel, waveform animating

SpeechEngine audio buffer callback
    → onPartialResult → AppDelegate → OverlayPanel.updateText(text)
    → onAudioLevel → AppDelegate → OverlayPanel.updateAudioLevel(normalizedRMS)

Fn released → KeyMonitor → AppDelegate.fnUp()
    → SpeechEngine.stopRecording()
    → waits up to 2s for onFinalResult, then:
        → (optional) LLMRefiner.refine(text)  # POST to OpenAI-compatible /chat/completions
        → TextInjector.inject(finalText)       # clipboard save → Cmd+V → clipboard restore
        → OverlayPanel.dismiss()
```

### Key subsystems

| File | Responsibility | Key APIs used |
|------|---------------|---------------|
| `KeyMonitor.swift` | Monitors Fn key via CGEvent tap; suppresses Fn to prevent emoji picker. Requires Accessibility permission. | `CGEvent.tapCreate`, `CGEventFlags.maskSecondaryFn` |
| `SpeechEngine.swift` | Owns AVAudioEngine (mic capture) and SFSpeechRecognizer (on-device transcription). Computes RMS audio level for visualization. Manages locale switching. | `SFSpeechRecognizer`, `AVAudioEngine` |
| `TextInjector.swift` | Injects text via clipboard + simulated Cmd+V keystroke. Temporarily switches non-ASCII IMEs (e.g. Chinese) to an ASCII layout to prevent interception of the paste. Restores original clipboard and input source after injection. | `NSPasteboard`, `CGEvent` (keyboard), `TIS` (Text Input Source) |
| `OverlayPanel.swift` | Borderless, non-activating NSPanel centered at screen bottom. Contains animated audio waveform bars (5 vertical CALayer bars driven by smoothed RMS level) and a text label. Animated show/dismiss via NSAnimationContext. | `NSPanel` (`.borderless`, `.nonactivatingPanel`), `CALayer`, `NSVisualEffectView` (.hudWindow) |
| `LLMRefiner.swift` | Singleton. Sends transcribed text to an OpenAI-compatible API with a conservative correction prompt (fixes homophone errors, English-as-Chinese rendering). Configurable via menu bar → settings. Logs to `~/Library/Logs/VoiceInput.log`. | `URLSession`, `UserDefaults` |
| `AppDelegate.swift` | NSApplicationDelegate. Builds status bar menu (enable/disable, language selector, LLM refinement toggle/settings, quit). Orchestrates fnDown/fnUp flow, speech callbacks, refinement, and text injection. | `NSStatusBar`, `NSMenu`, `UserDefaults` |
| `SettingsWindow.swift` | NSPanel form for LLM API configuration (base URL, key, model). Test button sends a probe request. Save persists to UserDefaults. | `NSGridView`, `UserDefaults` |

### UserDefaults keys

All settings are persisted via `UserDefaults.standard`:
- `selectedLocaleCode` — BCP-47 locale string (default `"zh-CN"`)
- `llmEnabled` — Bool, whether LLM refinement is on
- `llmAPIBaseURL` — String, defaults to `https://api.openai.com/v1`
- `llmAPIKey` — String
- `llmModel` — String, defaults to `gpt-4o-mini`

### Permissions required at runtime

1. **Accessibility** — CGEvent tap for Fn key monitoring
2. **Speech Recognition** — SFSpeechRecognizer
3. **Microphone** — AVAudioEngine input

Failure to grant any of these produces a modal alert directing the user to System Settings.

### LLM Refinement

The system prompt instructs the model to fix only clear recognition errors (English words rendered as Chinese characters, obvious homophone mistakes, broken English words) — explicitly forbidding rephrasing, rewriting, or "improving" text. The API call has a 10-second timeout. If refinement fails or returns empty, the original transcription is used unchanged.

## Info.plist notable keys

- `LSUIElement` = true → no Dock icon
- `NSMicrophoneUsageDescription` and `NSSpeechRecognitionUsageDescription` → permission prompt strings
- `LSMinimumSystemVersion` = 14.0
