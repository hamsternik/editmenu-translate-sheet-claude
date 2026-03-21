# iOS Application Core Workflow PLAN

There are 2 main scenarios of the core feature I see possible to implement. Each scenario requires different build approach and kind of effort.

---

## Decision

**Selected:** Option #2 - TranslationUIProvider Extension

**App Name:** Translate then Discuss

**Rationale:** Option #1 is technically impossible (no public API for cross-app callout injection). Option #2 is the only viable path for system-wide text access via the callout bar.

---

User flow:

User open an article in the iOS application (in browser, in PDF, in notes), he wants to read and research the theme of the text he is reading.

## Option #1 (REJECTED)

1. user selects text are in any iOS app, e.g. Safari browser.
2. show custom button e.g. "Research" in the `UIEditMenuInteraction`.
3. user taps Research button, custom `UISheetPresentationController` shows up with the UI to work with ai agent, e.g. Anthropic's claude via api.

Problem: all steps require build your own iOS application. No way to add menu item or show up sheet controller from another app. No access to the `UIEditMenuInteraction` or `UISheetPresentationController` directly or via any public accessible api, e.g. via XPC services, etc.

**Status:** Rejected - technically impossible with current iOS APIs.

## Option #2 (SELECTED)

1. user selects text in any iOS app, e.g. Safari browser.
2. taps on "Translate" built-in menu item button in the `UIEditMenuInteraction`. Translate button is available by-default for any selected text throughout iOS system.
3. display custom `UISheetPresentationController` above the active app with selected text, copied selected text to active modal controller and translates it (by design). Notice the translation app is configurable in iOS systems settings starting from iOS 17.4+ having public access to the Translation framework.
4. create new chat with ai agent, e.g. new chat with claude, where ai agent will search the information about the selected text's topic and dive into it while browsing initial one.

Problem: based on the [[file:CONCEPT-ios-translate-callout-item.md]] research the `TranslationUIProvider` extension point opened to third-party default translation apps in iOS 18.4 that require to build your own translation application, that constraint the UI/UX and product concept of the own application. I do not know how strict Apple would review the app core idea and its functionality, where the Translate is just the way to work around the quick access to ai agent chat (no translation designed in my app by default, as part of the core feature) to discuss the topic instead of translation.

To master `UIEditMenuInteraction` latest api check out [WWDC22: Adopt desktop-class editing interactions | Apple] (https://www.youtube.com/watch?v=j6qExADFrQ8).

**Status:** Selected

### Implementation Details

| Parameter | Value |
|-----------|-------|
| Platform | iPhone only |
| Minimum iOS | 18.4 |
| App Store Category | Reference (primary), Productivity (secondary) |
| Developer Program | Paid ($99/year) - likely required |

### App Store Review Strategy

**Mitigation for "Translate → AI chat" concern:**
1. Include actual translation as the **first step** (using Apple's Translation framework)
2. Name "Translate then Discuss" sets correct user expectations
3. Translation is baseline feature, AI discussion is value-add
4. Be transparent in App Store description

### Project Structure

Hybrid SPM + Xcode approach in `/translate-then-discuss` directory:
- `Package.swift` + `Sources/TranslationFeature/` - all Swift code (testable)
- `App/*.xcodeproj` - Xcode project with targets
- `App/iOS/` - main app (Info.plist, entitlements)
- `App/TranslationExtension/` - extension (Info.plist, entitlements)

Info.plist and .entitlements must stay in Xcode targets (build-time configs). 

### Alternatives

1. Build your own browser. :)

That option theoretically allows to manage the core user scenario: find the source of information, read it, keep it to read it later and deal with text from iOS standpoint. Again, the own application like browser gives full control over `UIEditMenuProvider`, `UISHeetPresentationController` instead of abusing and working around `TranslationUIProvider` public api.

## Open Questions

### Resolved

- ~~How to access selected text system-wide?~~ → TranslationUIProvider extension with `TranslationUIProviderContext.inputText`
- ~~Project structure?~~ → Hybrid SPM + Xcode in `/translate-then-discuss`
- ~~App Store category?~~ → Reference (primary)
- ~~Minimum iOS version?~~ → 18.4 (TranslationUIProvider requirement)
- ~~Platform support?~~ → iPhone only
- ~~Which Claude SDK?~~ → SwiftAnthropic (most active)

### Unresolved

1. Exact entitlement key name for TranslationUIProvider
2. Does entitlement require capability request approval or auto-granted to paid accounts?
3. Can extension code be tested without system "Translate" trigger?
4. Chat history persistence strategy (local vs synced)
5. How to structure Claude chat UX within constrained sheet UI
6. API key storage strategy (keychain, secure enclave?)

### Claude API Integration

Official Claude's client SDKs, https://platform.claude.com/docs/en/api/client-sdks.

Swift libraries (non-official):
- **SwiftAnthropic** (RECOMMENDED): https://github.com/jamesrochabrun/SwiftAnthropic, 232 stars, active
  - Author's guide: https://jamesrochabrun.medium.com/anthropic-ios-sdk-032e1dc6afd8
- SwiftClaude: https://github.com/GeorgeLyon/SwiftClaude (stale)
- anthropic-swift-sdk: https://github.com/tthew/anthropic-swift-sdk (minimal activity)

## References

### Research Documents
- The iOS text selection callout bar is a closed system. [[file:CONCEPT-ios-text-selection-callout-bar.md]]
- iOS Translate callout, Look Up, and 3rd-party text-processing surfaces. [[file:CONCEPT-ios-translate-callout-item.md]]
- Apple Developer Program requirements analysis. [[file:RESEARCH-apple-developer-program-requirements.md]]

### Apple Documentation
- [TranslationUIProvider](https://developer.apple.com/documentation/TranslationUIProvider)
- [Preparing your app to be the default translation app](https://developer.apple.com/documentation/TranslationUIProvider/Preparing-your-app-to-be-the-default-translation-app)
- [Translating text within your app](https://developer.apple.com/documentation/Translation/translating-text-within-your-app)
- [WWDC 2024 Session 10117 - Meet the Translation API](https://developer.apple.com/videos/play/wwdc2024/10117/)

### Inspiration
- [Reddit: Level Up iOS Translation 18.4](https://www.reddit.com/r/iosapps/comments/1k9yucg/level_up_ios_translation_184_ai_translation/) - Similar TranslationUIProvider approach
- [Reddit: AIBoard](https://www.reddit.com/r/iosapps/comments/1btv926/introducing_an_ai_keyboard_on_ios_aiboard/) - Alternative keyboard approach

