# EditMenu Translate Sheet - Claude AI Research Agent

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

**Minimum iOS Version**: 18.4
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

## App Store Review Risk

**Critical Consideration**: This app uses "Translate" as an entry point but provides AI research chat instead of translation. Apple may reject because:
- Misleading use of translation entry point
- No actual translation functionality
- Users expect translation when tapping "Translate"

**Possible Mitigations**:
- Include basic translation as a secondary feature
- Frame the app as "Translate & Research"
- Be transparent in App Store description

## Project Structure

```
editmenu-translate-sheet-claude/
├── docs/                           # Research and planning documents
│   ├── PLAN.md                     # Core workflow options and decisions
│   ├── CONCEPT-ios-text-selection-callout-bar.md   # Callout bar research
│   └── CONCEPT-ios-translate-callout-item.md       # TranslationUIProvider research
├── CLAUDE.md                       # This file
└── [App source - TBD]
```

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

## Commands

```bash
# Build for device (Simulator unsupported)
xcodebuild -scheme [SchemeName] -destination 'platform=iOS,name=[DeviceName]'

# Verify entitlements
codesign -d --entitlements - [AppPath]
```
