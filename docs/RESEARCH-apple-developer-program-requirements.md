# Apple Developer Program Requirements for TranslationUIProvider

**Research Date**: 2026-03-20
**Status**: Partially resolved - strong indicators but no definitive confirmation

---

## Summary

Testing the TranslationUIProvider extension on a real device **likely requires a paid Apple Developer Program membership** ($99/year). While no official documentation explicitly states this, multiple indicators suggest the entitlement is not available to free Personal Team accounts.

---

## TranslationUIProvider Technical Requirements

Per [Apple Developer Documentation](https://developer.apple.com/documentation/TranslationUIProvider/Preparing-your-app-to-be-the-default-translation-app) and [Slator analysis](https://slator.com/how-to-become-the-default-translation-app-on-the-iphone/):

| Requirement | Description |
|-------------|-------------|
| **Entitlement** | Signals to OS that app can appear in default translation list |
| **Network Access Key** | TranslationUIProvider network access permission |
| **App Extension** | TranslationUIProviderScene (SwiftUI Scene) |
| **UI Implementation** | Minimal SwiftUI interface (~40 lines per Apple sample) |
| **Real Translation** | Must have genuine translation capability (local model or cloud API) |

---

## Free Account (Personal Team) Capabilities

Per [Apple Membership Comparison](https://developer.apple.com/support/compare-memberships/) and [Forum discussions](https://developer.apple.com/forums/thread/669516):

### What's Included (Free)
- Xcode and beta releases
- On-device testing (with limitations)
- Apple Developer Forums
- Bug reporting via Feedback Assistant
- OS beta releases

### Limitations
| Constraint | Limit |
|------------|-------|
| App IDs | 10 max, expire after 7 days |
| Test Devices | 3 per platform, expire after 7 days |
| Provisioning Profiles | 7-day validity |
| App Store Distribution | Not available |
| TestFlight | Not available |
| Advanced Capabilities | Limited subset |

---

## Evidence: Paid Account Likely Required

### 1. TranslationUIProvider Not in Standard Capabilities List

Checked [Supported capabilities (iOS)](https://developer.apple.com/help/account/reference/supported-capabilities-ios/) - TranslationUIProvider/Default Translation is **not listed** among standard iOS capabilities. This suggests it may be:
- A managed capability requiring explicit approval
- Only available to paid developer accounts
- Handled through a separate entitlement assignment process

### 2. Entitlement Language Suggests Managed Process

Apple documentation states apps need "an entitlement that tells the OS the bundle can appear in the user's default list." The language around entitlements being "assigned to your account" is typically used for [managed capabilities](https://developer.apple.com/help/account/capabilities/capability-requests/) that require Apple approval.

### 3. System-Level Integration Pattern

Other default app capabilities (Browser, Email, etc.) require paid membership. Default Translation follows the same pattern - it's a system settings integration, not just an in-app feature.

### 4. First Adopters Are Established Companies

[DeepL](https://support.deepl.com/hc/en-us/articles/19917774341916-Select-DeepL-as-default-translation-app) and [Google Translate](https://www.macrumors.com/2025/05/19/google-translate-default-option-ios/) were first to support this feature. Both are established companies with Apple Developer Program memberships.

### 5. Extension Entitlements Generally Require Paid Account

App extensions that integrate with system features typically require capabilities not available to free Personal Team accounts.

---

## What You CAN Do Without Paid Account

| Task | Feasible |
|------|----------|
| Build main app target | Yes |
| Build TranslationFeature SPM library | Yes |
| Write and compile extension code | Yes |
| Test Claude API integration | Yes |
| Test SwiftUI UI in app context | Yes |
| Run app on device (7-day limit) | Yes |
| Test "Translate" button → extension flow | **No** |
| Appear in Settings → Default Apps | **No** |
| Submit to App Store | **No** |

---

## Recommended Approach

### Option A: Start Free, Upgrade When Needed
1. Build the app architecture and UI with free account
2. Test Claude API integration in a standalone app context
3. Upgrade to paid account ($99) when ready to test actual TranslationUIProvider flow
4. Submit to TestFlight/App Store

### Option B: Start with Paid Account
1. Join Apple Developer Program immediately
2. Full access to all capabilities from day one
3. Test complete flow including system integration

---

## Open Questions (Unresolved)

1. **Exact entitlement key name** - Not publicly documented (likely `com.apple.developer.translation-ui-provider` or similar)

2. **Is capability request required?** - Unknown if developers must apply via [Capability Requests](https://developer.apple.com/help/account/capabilities/capability-requests/) tab or if it's automatically available to paid accounts

3. **Extension testing workaround** - Can the extension code be tested in isolation without the system "Translate" trigger?

4. **Review timeline** - If capability request is needed, how long does Apple take to approve?

---

## Sources

- [TranslationUIProvider Documentation](https://developer.apple.com/documentation/TranslationUIProvider)
- [Preparing your app to be the default translation app](https://developer.apple.com/documentation/TranslationUIProvider/Preparing-your-app-to-be-the-default-translation-app)
- [How to Become the Default Translation App on the iPhone - Slator](https://slator.com/how-to-become-the-default-translation-app-on-the-iphone/)
- [Apple Developer Membership Comparison](https://developer.apple.com/support/compare-memberships/)
- [Free Provisioning Profile Limitations - Apple Forums](https://developer.apple.com/forums/thread/669516)
- [Capability Requests - Apple Developer Help](https://developer.apple.com/help/account/capabilities/capability-requests/)
- [Supported capabilities (iOS)](https://developer.apple.com/help/account/reference/supported-capabilities-ios/)
- [Google Translate Default Option - MacRumors](https://www.macrumors.com/2025/05/19/google-translate-default-option-ios/)
- [DeepL Default Translation App - Help Center](https://support.deepl.com/hc/en-us/articles/19917774341916-Select-DeepL-as-default-translation-app)
