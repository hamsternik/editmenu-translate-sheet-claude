# The iOS text selection callout bar is a closed system

**Third-party apps cannot add custom buttons to the system-wide iOS text selection callout bar.** No public API, extension point, entitlement, or MDM profile exists to inject items into the dark pill menu that appears above selected text in other apps like Safari, Notes, or Mail. This has been true across every iOS version from 3.0 through 18.x, and remains architecturally unchanged in iOS 26. The closest system-wide integration available to third parties is the **Action Extension in the Share Sheet**, reachable via the "Share…" item in the callout overflow — a minimum of **3 taps** from text selection. Writing Tools, the only non-standard item Apple has added to this surface, is a private first-party integration that third parties cannot replicate. For a senior engineer making an architectural decision: plan around the Share Sheet, not the callout bar.

---

## From UIMenuController to UIEditMenuInteraction: a 15-year evolution

The text selection callout bar has been governed by three distinct API generations:

**iOS 3.0–15.x: UIMenuController era.** Introduced in 2009 alongside copy/paste support, `UIMenuController` was a **global singleton** (`UIMenuController.shared`) managing the dark pill menu. Developers could add custom items via `UIMenuController.shared.menuItems = [UIMenuItem(title:action:)]`, but these only supported a title and action selector — no images, no submenus. The system determined which items appeared by walking the **responder chain**, sending `canPerformAction(_:withSender:)` to each responder starting from the first responder. This mechanism was entirely app-scoped: each app had its own singleton instance.

**iOS 16.0: UIEditMenuInteraction replaces everything.** At WWDC 2022, Apple deprecated `UIMenuController`, `UIMenuItem`, and all associated APIs in one sweep. The replacement, `UIEditMenuInteraction` (introduced in **session 10071, "Adopt desktop-class editing interactions"**), is instance-based rather than singleton-based. Key architectural improvements include support for the full `UIMenuElement` hierarchy (`UIAction`, `UIMenu` with submenus, images, states), adaptive presentation based on input method (compact pill for touch, context menu for pointer/keyboard, native menu for Mac Catalyst), and a clean delegate pattern via `UIEditMenuInteractionDelegate`. For text views specifically, new delegate methods were added: `UITextViewDelegate.textView(_:editMenuForTextIn:suggestedActions:) -> UIMenu?` lets developers append custom `UIAction` items alongside system-suggested actions. The `UIMenu.preferredElementSize` property (`.small`, `.medium`, `.large`) controls layout density within the pill.

**iOS 17.0: UITextItem and selection display separation.** WWDC 2023 session 10058 ("What's new with text and text interactions") introduced `UITextItem`, representing interactive content in UITextView — links, text attachments, and **custom-tagged ranges** (new). Two delegate methods handle these: `textView(_:menuConfigurationFor:defaultMenu:) -> UITextItem.MenuConfiguration?` for context menus on specific text items, and `textView(_:primaryActionFor:defaultAction:) -> UIAction?` for tap actions. Separately, `UITextSelectionDisplayInteraction` decoupled selection UI rendering (cursor, handles, highlight) from gesture handling, enabling custom text views to get system-consistent selection UI without adopting the full `UITextInteraction`. **All of these APIs operate exclusively within the developer's own app.**

**iOS 18.0–18.1: Writing Tools enters the callout bar.** The Writing Tools item (the sparkle/rotating-arrows icon) appeared in the callout bar as a system-provided feature requiring **no developer adoption** for standard text views (WWDC 2024, session 10168, "Get started with Writing Tools"). The public API surface is for *configuring* Writing Tools behavior, not replicating its placement: `UITextView.writingToolsBehavior` (`.complete`, `.limited`, `.none`), delegate methods `textViewWritingToolsWillBegin/DidEnd`, and `writingToolsIgnoredRangesIn:` for protecting code blocks. iOS 18.2 added `UIWritingToolsCoordinator` for custom text engines, and a `UIBarButtonItem.SystemItem.writingTools` convenience. Writing Tools requires Apple Silicon (iPhone 15 Pro+, M1+ iPad/Mac) and runs Apple's on-device LLM.

---

## Why third parties cannot reach the system-wide callout bar

The impossibility of system-wide callout bar injection stems from three reinforcing constraints:

**No extension point exists.** Apple's documented `NSExtensionPointIdentifier` values include widget, share (`com.apple.share-services`), action (`com.apple.ui-services`), custom keyboard (`com.apple.keyboard-service`), content blocker, intents, and several others. **None relate to text selection menus or edit menus.** There is no `com.apple.text-selection-action` or equivalent. Every Apple Developer Forums thread about customizing the edit menu (e.g., threads 733086, 773860, 88465, 125511, 31230) involves customization within the developer's own app — no Apple engineer has ever hinted at cross-app capability.

**iOS sandboxing prevents UI injection.** Unlike macOS, where apps can register `NSServices` in Info.plist to appear in any app's Services menu, iOS's process isolation model prevents any app from modifying another app's view hierarchy or responder chain. The callout bar is rendered within the foreground app's process by UIKit internals. There is no inter-process communication mechanism for injecting menu items. Even `UIMenuBuilder` and `UIMenuSystem.context` build menus for the app's own responder chain only.

**Writing Tools confirms the asymmetry.** Writing Tools' placement in the callout bar is hardcoded into UIKit's internal text selection handling. As Apple stated in session 10168: "As long as your custom text view adopts UITextInteraction, you'll get Writing Tools in the callout bar or context menu for free." The word "free" is revealing — it's something Apple gives you, not something you can build. No third-party equivalent placement mechanism exists. Internally, Writing Tools is injected via the system's own responder chain before the app sees the menu, using private classes prefixed with `_UIEditMenu` and a magic configuration identifier `UITextContextMenuInteraction.TextSelectionMenu`.

**Jailbreak-only alternatives confirm the lock.** The only known system-wide edit menu modifications come from jailbreak tweaks like **MenuSupport** (by r-plus, GitHub `r-plus/MenuSupport`) and **ActionBar**, which use Objective-C runtime method swizzling via Substrate/Theos to hook UIKit internals. These require root-level access impossible through the App Store. No App Store app has ever placed itself in the system-wide callout bar — not Terminology, not Copied, not any dictionary app.

---

## What the chevron reveals and how overflow works

The callout bar displays a limited number of items that fit its compact pill width. **Left/right chevron arrows (‹ ›) enable pagination** through additional "pages" of menu items. The overflow is not a dropdown or context menu — it's horizontal paging within the same pill UI (though iOS 26 reportedly changes this to a full secondary context menu).

A typical item order across pages in iOS 18:

- **Page 1:** Cut · Copy · Paste (context-dependent)
- **Page 2+:** Replace… · Look Up · Translate · Writing Tools (iOS 18.1+) · Share… · Speak · BIU (Bold/Italic/Underline)
- **After system items:** Any custom items added by the foreground app via `editMenuForTextIn:suggestedActions:`
- **Data detector items (iOS 16+):** Context-sensitive actions like "Get Directions" for addresses or inline unit/currency conversions

**Critical distinction:** The "Share…" item in the overflow opens a `UIActivityViewController` (the Share Sheet), which is the only place third-party extensions appear. The Share Sheet is a completely separate UI from the callout bar. Items in the overflow pages are exclusively system-provided items plus the foreground app's own custom items — no third-party cross-app items ever appear here.

---

## macOS Services versus the iOS gap

macOS has had system-wide text processing since NeXTSTEP. Any app can register services via `NSServices` in Info.plist, specifying `NSSendTypes` and `NSReturnTypes` for pasteboard data. A registered service appears in the **right-click context menu → Services submenu** across all Cocoa apps, can receive selected text, transform it, and return modified text to the originating app — functioning like a "GUI pipe." Users can assign keyboard shortcuts in System Settings → Keyboard → Keyboard Shortcuts → Services. Shortcuts.app on macOS also surfaces user-created shortcuts in the Services menu.

**iOS has never received an equivalent.** No WWDC session has announced bringing Services to iOS. The architectural reasons are clear: iOS's sandboxing model, which isolates app processes far more strictly than macOS, makes the seamless bidirectional data flow of Services fundamentally incompatible with iOS's security model. Mac Catalyst apps don't get Services automatically either — providing services requires an embedded AppKit plugin bundle that calls `NSRegisterServicesProvider()`, a workaround documented by developers like Conrad Stoll for the Grocery app.

The practical consequence is stark: on macOS, a text transformation from selection to result takes **1–2 interactions** (right-click → service, or keyboard shortcut). On iOS, the equivalent flow requires **3+ taps** minimum through the Share Sheet, and even then, **round-trip text modification rarely works** because most host apps don't implement `completionWithItemsHandler` on their `UIActivityViewController`.

---

## Ranked integration points for selected text processing on iOS

From most integrated (fewest taps, least friction) to least:

**1. Custom callout bar item — 0 extra taps, but in-app only.** Using `UIEditMenuInteraction` (iOS 16+) or `textView(_:editMenuForTextIn:suggestedActions:)`, a developer can place a custom `UIAction` directly in the callout pill alongside Cut/Copy/Paste within their own app's text views. This is the gold standard for UX but has zero cross-app reach. API: `UIEditMenuInteractionDelegate`, `UITextViewDelegate`.

**2. Action Extension via Share Sheet — 3 taps, system-wide.** Select text → tap "Share…" in callout overflow → tap Action Extension in the bottom row of the Share Sheet. Extension point: `com.apple.ui-services` with `NSExtensionActivationSupportsText = YES`. Can theoretically return modified text via `completeRequest(returningItems:)`, but most host apps ignore the returned items. This is **the most integrated system-wide placement available to third parties today**.

**3. Share Extension via Share Sheet — 3 taps, system-wide, one-way.** Same path as Action Extensions but appears in the top icon row. Extension point: `com.apple.share-services`. Designed for one-way data flow (send content to your app) — cannot return modified text to the host.

**4. Shortcut in the Share Sheet — 3–4 taps, system-wide.** Select text → Share → scroll to Shortcuts section → tap shortcut. The selected text becomes the `Shortcut Input` variable. Powerful for processing pipelines (regex, API calls, ChatGPT) but cannot return text in-place; results go to clipboard, Quick Look, or another app. No Xcode required — users create shortcuts in the Shortcuts app with "Show in Share Sheet" enabled and input type set to "Text."

**5. Shortcut via Siri/Back Tap — 4+ steps, clipboard-based.** Select → Copy → trigger shortcut (voice command, Back Tap, widget, Control Center) → Paste result. The shortcut reads from clipboard via "Get Clipboard" action. Loses selection context entirely.

**6. Custom Keyboard — 3+ taps, limited text access.** Switch to custom keyboard → tap processing button → switch back. Uses `UIInputViewController` and `textDocumentProxy`. **Critical limitation: custom keyboards cannot reliably access the current text selection.** `textDocumentProxy` provides context before/after the cursor but not the selected range, making keyboards unsuitable for text transformation workflows.

**7. Universal Clipboard round-trip — 5+ steps.** Copy on iOS → wait for Handoff sync to Mac → process on Mac (via Services, script, etc.) → copy result → paste on iOS. Requires same iCloud account, Bluetooth, Wi-Fi, and a nearby Mac. Impractical for routine use.

---

## Conclusion: architectural recommendations

The iOS text selection callout bar is and has always been a **closed, first-party-only surface**. No API, extension point, entitlement, MDM profile, or private API accessible from the App Store enables cross-app injection. Writing Tools' privileged placement confirms Apple reserves this surface exclusively for system features.

For a senior iOS engineer deciding on architecture, three paths merit consideration. If your app **owns the text view**, use `UIEditMenuInteraction` and the `editMenuForTextIn:suggestedActions:` delegate to place your action directly in the callout pill — this is the best UX possible at **zero extra taps**. If you need **system-wide reach**, build an Action Extension (`com.apple.ui-services`) that activates for text content; accept the 3-tap minimum and design your extension's UI for speed. If you want to enable **user-customizable text processing pipelines**, expose your functionality as `AppIntents` so users can invoke it from Shortcuts in the Share Sheet or via Siri.

The absence of an iOS Services equivalent is the platform's most significant text-processing gap compared to macOS. Given Apple's trajectory — adding Writing Tools as a system-level callout item rather than opening the surface to developers — there is no indication this will change. Design accordingly: optimize your Share Sheet extension's activation speed and first-run experience, because that 3-tap path is likely your permanent ceiling for system-wide integration.

**Key API reference summary:**

| iOS Version | Key API | WWDC Session |
|---|---|---|
| iOS 3.0 | `UIMenuController` (introduced) | — |
| iOS 13.0 | `UITextInteraction` | WWDC 2019 |
| iOS 16.0 | `UIEditMenuInteraction` (replaces UIMenuController) | WWDC 2022, Session 10071 |
| iOS 17.0 | `UITextItem`, `UITextSelectionDisplayInteraction` | WWDC 2023, Session 10058 |
| iOS 18.1 | Writing Tools (`UIWritingToolsBehavior`) | WWDC 2024, Session 10168 |
| iOS 18.2 | `UIWritingToolsCoordinator` (custom text engines) | WWDC 2024, Session 10168 |
