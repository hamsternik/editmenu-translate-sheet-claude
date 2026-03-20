# EditMenu Translate Sheet - Claude AI Research Agent

## Rules for Claude

1. **Ask questions when uncertain.** Do not guess or assume. If information is incomplete or ambiguous, ask clarifying questions. Generate "Open Questions" when discovering unknowns during research or implementation.

2. **Update CLAUDE.md on explicit request.** When the user says "keep in mind", "remember", or similar phrases indicating a persistent rule, add it to this file immediately.

---

## Project Overview

iOS proof-of-concept app that hijacks the system "Translate" callout (iOS 18.4+) to provide quick access to Claude AI for researching selected text topics.

**User Flow**: Select text anywhere in iOS (Safari, Notes, etc.) → Tap "Translate" → App's sheet appears → Chat with Claude AI about the selected text.

## Technical Architecture

### TranslationUIProvider Extension (iOS 18.4+)

The app must implement a `TranslationUIProvider` app extension to intercept the system "Translate" action:

```
Required Components:
├── Entitlement (from Apple Developer Portal)
├── Network access key for TranslationUIProvider
├── TranslationUIProviderScene (SwiftUI Scene)
└── TranslationUIProviderContext → inputText property
```

**Platform**: iPhone only (no iPad/Mac support for now)
**Minimum iOS Version**: 18.4 (TranslationUIProvider not available in earlier 18.x)
**Framework**: SwiftUI required (no UIKit direct support)

### Claude API Integration

Using SwiftAnthropic (most active community SDK):
- Repository: https://github.com/jamesrochabrun/SwiftAnthropic
- Author's guide: https://jamesrochabrun.medium.com/anthropic-ios-sdk-032e1dc6afd8

Alternative SDKs (less recommended):
- SwiftClaude: https://github.com/GeorgeLyon/SwiftClaude (stale)
- anthropic-swift-sdk: https://github.com/tthew/anthropic-swift-sdk (minimal activity)

## Platform Constraints

### What's Possible
- Register as default translation app (Settings → Apps → Default Apps → Translation)
- Receive selected text from any iOS app via `TranslationUIProviderContext.inputText`
- Present custom SwiftUI UI as system-controlled sheet
- Full network access for Claude API calls

### What's NOT Possible
- Adding custom buttons to system-wide callout bar (closed API)
- Customizing the sheet presentation (system-controlled via XPC)
- Using Look Up panel (completely closed, no extension point)
- Intercepting text selection without "Translate" tap

### Presentation Behavior
The system controls presentation via private `_UIRemoteViewController` over XPC. Your extension's SwiftUI view is rendered in a separate process. You cannot customize detents, grabbers, or presentation style - the system handles this.

## App Store Configuration

**Platform:** iPhone only
**Category:** Reference (primary), Productivity (secondary)
- Reference aligns with where translation apps (DeepL, Google Translate) are listed
- Users searching for translation alternatives browse Reference
- Matches the TranslationUIProvider system integration

**Minimum iOS Version:** 18.4 (TranslationUIProvider introduced March 31, 2025)

---

## App Store Review Risk

**Critical Consideration**: This app uses "Translate" as an entry point but provides AI research chat instead of translation. Apple may reject because:
- Misleading use of translation entry point
- No actual translation functionality
- Users expect translation when tapping "Translate"

**Possible Mitigations**:
- Include basic translation as the first step in the workflow
- Frame the app as "Translate then Discuss" - translation is the entry point, discussion is the value-add
- Be transparent in App Store description

## Project Structure

Hybrid approach: extension logic in SPM package, Xcode-required files in target directories.
Swift package isolated in `/translate-then-discuss` to separate from docs and other project assets.

```
editmenu-translate-sheet-claude/           # Repository root
├── CLAUDE.md                              # This file
├── docs/                                  # Research and planning documents
│   ├── PLAN.md
│   ├── CONCEPT-ios-text-selection-callout-bar.md
│   ├── CONCEPT-ios-translate-callout-item.md
│   └── RESEARCH-apple-developer-program-requirements.md
├── landing/                               # (future) Landing page assets
└── translate-then-discuss/                # Swift package root
    ├── Package.swift                      # SPM manifest
    ├── Sources/
    │   └── TranslationFeature/            # ALL extension logic (testable)
    │       ├── TranslationProviderScene.swift
    │       ├── ChatView.swift
    │       ├── ClaudeService.swift
    │       ├── Models/
    │       └── Resources/
    │           └── Assets.xcassets
    └── App/
        ├── EditMenuTranslate.xcodeproj    # Xcode project (user-managed)
        ├── iOS/                           # Main app target (minimal)
        │   ├── App.swift
        │   ├── Info.plist                 # MUST be here (build-time config)
        │   └── App.entitlements           # MUST be here (code signing)
        └── TranslationExtension/          # Extension target (thin wrapper)
            ├── Main.swift                 # Imports TranslationFeature
            ├── Info.plist                 # MUST be here (extension config)
            └── Extension.entitlements     # MUST be here (extension entitlements)
```

**Why Info.plist and .entitlements can't be in a package:**
- Info.plist: Xcode substitutes build variables like `$(PRODUCT_BUNDLE_IDENTIFIER)`
- .entitlements: Embedded into binary signature by `codesign` during build
- Both must be associated with an Xcode target, not a Swift Package product

## Development Guidelines

### SwiftUI First
TranslationUIProvider requires SwiftUI. Use UIHostingController only if bridging to UIKit is absolutely necessary.

### Testing
- Physical device required (Simulator unsupported for Translation framework)
- Test across iOS 18.4+ devices
- Verify Settings → Apps → Default Apps → Translation shows your app

### Key APIs to Study
| API | Purpose |
|-----|---------|
| `TranslationUIProvider` | Extension framework for default translation apps |
| `TranslationUIProviderScene` | SwiftUI Scene for the extension |
| `TranslationUIProviderContext` | Access to `inputText` (selected text) |
| `Translation` framework (iOS 17.4+) | Apple's translation engine (if adding translation feature) |

## WWDC Sessions Reference
- **WWDC 2024 Session 10117** - "Meet the Translation API"
- **WWDC 2022 Session 10071** - "Adopt desktop-class editing interactions"
- **WWDC 2023 Session 10058** - "What's new with text and text interactions"

## Open Questions

1. Exact entitlement key name for TranslationUIProvider (obtained via Developer Portal)
2. How to structure the Claude chat experience within the constrained sheet UI
3. Whether to include basic translation to satisfy App Store review
4. Chat history persistence strategy (local vs. synced)
5. Does TranslationUIProvider require capability request approval, or is it auto-available to paid accounts?
6. Can extension code be tested in isolation without the system "Translate" trigger?

## Research Documents

- [Apple Developer Program Requirements](docs/RESEARCH-apple-developer-program-requirements.md) - Analysis of free vs paid account capabilities for TranslationUIProvider

## Commands

```bash
# Build for device (Simulator unsupported)
xcodebuild -scheme [SchemeName] -destination 'platform=iOS,name=[DeviceName]'

# Verify entitlements
codesign -d --entitlements - [AppPath]
```
