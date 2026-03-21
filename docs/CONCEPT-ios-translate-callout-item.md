# iOS Translate callout, Look Up, and third-party text-processing surfaces

**The system "Translate" callout item was fully locked to Apple until iOS 18.4 (March 2025), when the new `TranslationUIProvider` framework opened it to third-party default translation apps. The "Look Up" callout remains completely closed — no extension point, no API, no third-party hook exists.** Both features use private remote-view-controller architecture (`_UIRemoteViewController` over XPC) that third parties cannot replicate. The public `Translation` framework (iOS 17.4+) is strictly for in-app consumption. This report covers every relevant API, entitlement, extension point, WWDC session, and architectural detail across all three topics.

---

## The Translation framework is powerful but in-app only

Apple introduced the public `Translation` framework in **iOS 17.4** and expanded it significantly in **iOS 18**. It exposes two distinct API surfaces, both SwiftUI-exclusive:

**`.translationPresentation(isPresented:text:attachmentAnchor:arrowEdge:replacementAction:)`** (iOS 17.4+) presents a system-owned overlay — a bottom sheet on iPhone, a popover on iPad/Mac — showing the same UI as Apple Translate. The optional `replacementAction` closure receives translated text so the app can replace the original inline. The UI is **not customizable** and uses Apple's on-device CoreML models. There is no `TranslationPresenter` class; this is purely a SwiftUI view modifier.

**`TranslationSession`** (iOS 18+) is the programmatic API. You never instantiate it directly — you obtain it through `.translationTask(_:action:)` or `.translationTask(source:target:action:)` view modifiers, which vend a session in their async closure. Key methods include `translate(_:)` for single strings, `translations(from:)` for ordered batch translation, and `translate(batch:)` returning an `AsyncThrowingStream` for streaming results. Each session is tied to a SwiftUI view's lifetime and must not be persisted beyond it. **All requests in a single batch must share the same source language**; mixing languages produces garbage output. The companion `LanguageAvailability` class exposes `supportedLanguages()` and `status(from:to:)` to check whether language models are `.installed`, `.supported` (downloadable), or `.unsupported`.

`TranslationSession.Configuration` holds optional `source` and `target` (`Locale.Language?`). Passing `nil` for source triggers auto-detection; `nil` for target picks the best match from the user's preferred languages. Calling `invalidate()` re-triggers the SwiftUI task. The framework supports **30+ languages** including Hindi (added in iOS 18). It requires a physical device — **the Simulator is unsupported**.

Critically, none of these APIs allow a third-party app to intercept or replace the system-wide "Translate" callout action. The `Translation` framework is for consuming Apple's translation engine within your own app's UI. `NLLanguageRecognizer` (NaturalLanguage framework) is a standalone language-detection utility with no hooks into the callout system. An Apple Engineer confirmed on the developer forums (thread 756837) that the Translation APIs are SwiftUI-only, requiring `UIHostingController` bridges for UIKit apps.

---

## Translation Framework Implementation Guide

This section provides practical code examples for implementing translation using Apple's Translation framework. Relevant for adding translation as a feature in your TranslationUIProvider extension.

### Two Approaches: Overlay vs Programmatic

**1. Translation Overlay (Simple)**

The `.translationPresentation()` modifier displays Apple's system translation UI:

```swift
import SwiftUI
import Translation

struct ContentView: View {
    @State private var showTranslation = false
    @State private var textToTranslate = "Hello, world!"

    var body: some View {
        VStack {
            Text(textToTranslate)
            Button("Translate") {
                showTranslation = true
            }
        }
        .translationPresentation(isPresented: $showTranslation, text: textToTranslate)
    }
}
```

With replacement action (allows replacing original text with translation):

```swift
.translationPresentation(isPresented: $showTranslation, text: textToTranslate) { translatedText in
    textToTranslate = translatedText
}
```

**2. Programmatic Translation (TranslationSession)**

For custom UI and batch processing:

```swift
import SwiftUI
import Translation

struct CustomTranslationView: View {
    @State private var configuration: TranslationSession.Configuration?
    @State private var sourceText = "Hello"
    @State private var translatedText = ""

    var body: some View {
        VStack {
            Text(translatedText)
            Button("Translate to Spanish") {
                configuration = TranslationSession.Configuration(
                    source: Locale.Language(identifier: "en"),
                    target: Locale.Language(identifier: "es")
                )
            }
        }
        .translationTask(configuration) { session in
            let response = try await session.translate(sourceText)
            translatedText = response.targetText
        }
    }
}
```

### Batch Translation

For translating multiple strings simultaneously:

```swift
.translationTask(configuration) { session in
    let requests = textsToTranslate.enumerated().map { index, text in
        TranslationSession.Request(sourceText: text, clientIdentifier: "\(index)")
    }

    // Option 1: Get all at once (maintains order)
    let responses = try await session.translations(from: requests)

    // Option 2: Stream results
    for try await response in session.translate(batch: requests) {
        // Handle each response as it arrives
    }
}
```

**Constraint:** All requests in a batch must share the same source language.

### Language Availability

Check if translation models are available before translating:

```swift
let availability = LanguageAvailability()
let status = await availability.status(
    from: Locale.Language(identifier: "en"),
    to: Locale.Language(identifier: "ja")
)

switch status {
case .installed:
    // Ready to translate offline
case .supported:
    // Model available for download
case .unsupported:
    // Language pair not supported
}
```

Pre-download models for offline use:

```swift
.translationTask(configuration) { session in
    try await session.prepareTranslation()
    // Now models are cached locally
}
```

### Key Constraints

| Constraint | Details |
|------------|---------|
| **Device only** | Simulator unsupported; must test on physical device |
| **SwiftUI only** | No UIKit API; use `UIHostingController` to bridge |
| **Single source language per batch** | Cannot mix English and Spanish strings in same batch |
| **Session lifecycle** | Tied to SwiftUI view; do not persist beyond view lifetime |
| **~20-30 languages** | Same languages as Apple Translate app |

### Sources

- [Using Translation API in Swift and SwiftUI - AppCoda](https://www.appcoda.com/translation-api/)
- [Free, on-device translations with the Swift Translation API - polpiella.dev](https://www.polpiella.dev/swift-translation-api/)
- [WWDC 2024 Session 10117 - Meet the Translation API](https://developer.apple.com/videos/play/wwdc2024/10117/)

---

## iOS 18.4 opened the Translate callout via `TranslationUIProvider`

The breakthrough came in **iOS 18.4** (March 31, 2025), when Apple introduced a separate framework — **`TranslationUIProvider`** — that lets third-party apps register as the system's default translation provider. This is entirely distinct from the `Translation` framework. When a user sets a third-party app as default at **Settings → Apps → Default Apps → Translation**, the system "Translate" callout item in text selection across all apps (Safari, Notes, Messages, etc.) routes to that app's extension instead of Apple Translate.

Becoming a default translation app requires four components:

- **An entitlement** that tells iOS the app can appear in the default translation app list (the exact key follows Apple's pattern but is vended through the developer portal, not a classic `NSExtensionPointIdentifier`)
- **A network access key** for `TranslationUIProvider` (for cloud-backed translation engines)
- **A `TranslationUIProvider` app extension** containing a `TranslationUIProviderScene` (a SwiftUI `Scene`)
- **A minimal SwiftUI UI** (~40 lines in Apple's sample code) that receives a `TranslationUIProviderContext` with the `inputText` property containing the selected text

Apple's documentation lives at `developer.apple.com/documentation/translationuiprovider/preparing-your-app-to-be-the-default-translation-app`. This was rolled out **globally**, not just in the EU. As of early 2026, **Google Translate** (updated ~May 2025), **DeepL** (announced April 24, 2025), and **Reverso Context** have adopted it. There is no legacy `com.apple.developer.translation-extension` entitlement — this mechanism uses the newer TranslationUIProvider extension system.

This is the only approved path for a third-party translation app to appear in the system callout. Before iOS 18.4, no public API, extension point, or entitlement existed. The "Translate" callout item was injected by the OS itself through `UIMenuController`/`UIEditMenuInteraction` internals and routed exclusively to Apple Translate.

---

## Safari's translate panel runs on private remote view controllers

The bottom-sheet translate panel shown when tapping "Translate" in Safari is **not** powered by the public `Translation` framework. Reverse engineering documented on Michael Tsai's blog (July 2024) confirmed that Safari soft-links to a **private framework called `TranslationUIServices`**, loading private Objective-C classes **`LTUITranslationViewController`** and **`LTUISourceMeta`**. This is entirely separate from the public API surface.

The presentation architecture uses Apple's **`_UIRemoteViewController`** pattern — the same XPC-based mechanism Apple has employed since iOS 6 for system-composed UI like `MFMailComposeViewController` and the share sheet. The translation UI runs in a **separate process**. The host app's view hierarchy terminates in a `_UIRemoteView` backed by a `CALayerHost`; the actual UI elements (source text, translated text, language pickers) live in the remote process. The factory method is `+[_UIRemoteViewController requestViewController:fromServiceWithBundleIdentifier:connectionHandler:]`.

**Third parties cannot create remote view controllers.** Experiments on jailbroken devices (documented by pixelomer's RemoteViewFun project) show that even with correct Info.plist entries (`UIViewServiceUsesNSXPCConnection: true`, Mach service registration under `com.apple.uikit.viewservice.*`), custom remote view services fail with `_UIViewServiceInterfaceErrorDomain error code 2`. The system restricts this to Apple-signed services. This means the specific "inline panel stays within host app" UX of the Translate feature is **architecturally impossible to replicate** from third-party code.

When a third-party app becomes the default translation provider via `TranslationUIProvider`, the system presumably routes the XPC call to that app's extension process instead of Apple Translate's service — but the presentation mechanism is still system-controlled.

---

## Look Up is a completely closed system with no third-party path

The "Look Up" callout item (rebranded from "Define" in iOS 10) is governed entirely by the OS. It is a system-provided action in `UIEditMenuInteraction` (or the deprecated `UIMenuController`), historically using the private selector `_define:` through the UIResponder chain. The presented controller is **not** `UIReferenceLibraryViewController` — since iOS 10, it's a separate internal controller that renders a blurred overlay sheet with rich content from multiple sources: system dictionaries, Wikipedia, Siri Knowledge, iTunes/App Store suggestions, Apple Maps nearby locations, and web search fallback. **Every data source is Apple-controlled. There is no extension point, no API, and no configuration mechanism for third parties to contribute content.**

**`UIReferenceLibraryViewController`** (UIKit, iOS 5+, still not deprecated) is the only public API touching dictionary data. It provides exactly two methods: `dictionaryHasDefinition(forTerm:)` (class method, synchronous — known to block the main thread for seconds via XPC) and `init(term:)`. It displays **dictionary definitions only** from system-installed dictionaries managed at Settings → General → Dictionary — the New Oxford American Dictionary, language-specific dictionaries, thesauruses, etc. It does not show Wikipedia, Siri Knowledge, or any of the richer Look Up panel content. The UI is opaque and non-customizable. Notably, Oxford University Press has asserted that dictionary definitions accessed through this API are "only licensed to Apple and not further sub-licensed to developers," leading to at least one app (Compact Dictionary) being pulled from the App Store.

**`DictionaryServices`** exists only on macOS as part of CoreServices. Its single public function, `DCSCopyTextDefinition()`, returns plain-text definitions. Private macOS APIs (`DCSCopyAvailableDictionaries`, `DCSCopyRecordsForSearchString`, `DCSRecordCopyData`) allow richer access including HTML definitions, but these are App Store-ineligible. **No iOS equivalent of DictionaryServices exists.**

The downloadable dictionaries in Settings → General → Dictionary are managed as **MobileAsset** downloads stored at system-protected paths. Over 70 dictionaries are available. **There is no API for third-party apps to register custom dictionaries**, and no mechanism to inject content into Look Up results. Apple Developer Forums threads requesting this (thread 132600, May 2020; thread 81207, June 2017) have **zero replies from Apple**. The macOS Dictionary Development Kit (part of Additional Tools for Xcode) allows creating `.dictionary` bundles for macOS Dictionary.app, but this has no iOS counterpart and the associated programming guide is described as "long defunct."

Dictionary/Look Up is **not** a default app category on iOS. The default app system (Settings → Apps → Default Apps) covers browser, email, messaging, calling, call filtering, keyboards, passwords, contactless payments, translation (iOS 18.4+), and navigation (EU/Japan only). There is no indication Apple plans to open Look Up to third parties.

---

## No extension type can inject items into another app's callout bar

**`UIEditMenuInteraction`** (iOS 16+, replacing the deprecated `UIMenuController`) allows custom menu items only within the app that owns the text view. The key delegate method `editMenuInteraction(_:menuFor:suggestedActions:) -> UIMenu?` is called on the host app's delegate. There is **no cross-app extension mechanism** — a third-party extension cannot inject items into Safari's or any other app's text selection callout. System-wide items like "Translate," "Look Up," "Writing Tools," and "Share" are added by the OS itself.

**Action extensions** (`com.apple.ui-services`) can process text but appear in the **UIActivityViewController share sheet**, triggered only after the user taps "Share" from the callout, then selects the extension. They present as modal form sheets (default) or full-screen (`NSExtensionActionWantsFullScreenPresentation = YES`). There is **no option to present as a bottom sheet** or inline panel. The host app controls extension presentation; extensions cannot customize the presentation controller.

**Share extensions** (`com.apple.share-services`) follow the same constraints — they appear in the activity view controller's top row and cannot present as inline panels.

**App Intents** surface through Siri, Spotlight, Shortcuts, widgets, Control Center, the Action Button, and Apple Pencil squeeze — but **not the text selection callout**. There is no "Translation" or "Dictionary" App Intent domain among the 12 domains introduced in iOS 18 (Mail, Photos, Books, Browsers, Camera, Documents, File Management, Journal, Presentations, Spreadsheets, Whiteboards, Word Processors).

**Keyboard extensions** can insert text via `textDocumentProxy.insertText()` and make network requests with Full Access, but they **cannot read selected text**, cannot access the edit menu, and cannot present overlays or bottom sheets outside their keyboard view bounds.

There is **no iOS equivalent of macOS Services**, which allows third-party apps to advertise text-processing operations that appear in context menus and can replace selected text inline. iOS's sandboxing model is fundamentally incompatible with this architecture.

---

## Every relevant WWDC session and what each covers

**WWDC 2024 Session 10117 — "Meet the Translation API"** (speaker: Louie Livon-Bemel) introduced `TranslationSession` and demonstrated `.translationPresentation()`. No extension points for third parties were mentioned. Best practices: attach modifiers to content views, don't mix languages in batches, use `nil` source for auto-detection, call `prepareTranslation()` for offline use, test on physical devices only.

**WWDC 2024 Session 10168 — "Get started with Writing Tools"** covered Apple Intelligence's Writing Tools system feature in the callout bar. It uses `UIWritingToolsBehavior` and integrates with `UITextView`, `WKWebView`, and custom text views via `UITextInteraction` + `UIEditMenuInteraction`. Writing Tools has **no third-party extension point** — it is a system feature, not a platform for third-party text processing.

**WWDC 2023 Session 10058 — "What's new with text and text interactions"** introduced `UITextSelectionDisplayInteraction`, redesigned text cursor and selection handles, and text item actions/menus in UITextView. It covered customizing text menus within your own app but announced no mechanism for cross-app menu injection.

**WWDC 2022 Session 10071 — "Adopt desktop-class editing interactions"** introduced `UIEditMenuInteraction` as the replacement for `UIMenuController`, covering `UIEditMenuConfiguration`, the delegate pattern, and find-and-replace UI. All customization is app-scoped.

**No WWDC 2023 or 2024 session announces opening the text-selection callout bar or Look Up panel to third-party extensions.** The only "opening" is the `TranslationUIProvider` framework (iOS 18.4), which was not covered in a WWDC session but documented through Apple's developer portal.

---

## What a third-party developer can actually achieve today

The architectural reality is that the "inline panel stays within host app" pattern used by Translate and Look Up relies on private `_UIRemoteViewController` XPC infrastructure unavailable to third parties. However, the `TranslationUIProvider` framework (iOS 18.4+) represents a genuine opening — it is the first time Apple has allowed a third-party app to handle a system callout action. For developers building AI-powered text processing, the realistic options are:

**Within your own app**, `UISheetPresentationController` (iOS 15+) or SwiftUI's `.presentationDetents()` can replicate the bottom-sheet UX perfectly — medium/large detents, grabber visibility, undimmed background interaction. Combined with `TranslationSession` for translation or custom ML models for other processing, this achieves the exact visual result inside your app.

**Across apps**, the `TranslationUIProvider` extension is the only approved path for appearing in the system Translate callout. For non-translation text processing, Action extensions via the share sheet remain the only cross-app mechanism, with the constraint of modal presentation rather than inline panels. Custom `UIMenuItem`/`UIEditMenuInteraction` items work within your own app only.

**For dictionary/lookup functionality**, there is no integration path. `UIReferenceLibraryViewController` provides read-only access to system dictionaries with licensing concerns. The Look Up panel remains fully closed. The most viable approach is adding a custom callout menu item within your own app that presents your own lookup UI.

## Conclusion

iOS's text-processing callout architecture has a clear asymmetry: **translation was opened to third parties in iOS 18.4 via `TranslationUIProvider`, but dictionary/Look Up remains entirely closed**. The inline-panel UX that makes both features elegant depends on private remote view controller infrastructure that third parties cannot access. The `Translation` framework is useful but limited to in-app consumption of Apple's engine. For developers seeking the "Translate panel" UX pattern, the path forward depends on context: within your own app, standard sheet presentation APIs replicate it perfectly; across apps, `TranslationUIProvider` is the only approved entry point, limited to the translation use case. Apple has shown no indication of generalizing this pattern to arbitrary text-processing extensions — there is no "Services for iOS" on the horizon.
