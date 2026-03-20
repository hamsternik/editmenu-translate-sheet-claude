# iOS Application Core Workflow PLAN

There are 2 main scenarios of the core feature I see possible to implement. Each scenario requires different build approach and kind of effort.

User flow:

User open an article in the iOS application (in browser, in PDF, in notes), he wants to read and research the theme of the text he is reading.

## Option #1

1. user selects text are in any iOS app, e.g. Safari browser.
2. show custom button e.g. "Research" in the `UIEditMenuInteraction`.
3. user taps Research button, custom `UISheetPresentationController` shows up with the UI to work with ai agent, e.g. Anthropic's claude via api.

Problem: all steps require build your own iOS application. No way to add menu item or show up sheet controller from another app. No access to the `UIEditMenuInteraction` or `UISheetPresentationController` directly or via any public accessible api, e.g. via XPC services, etc.

## Option #2

1. user selects text in any iOS app, e.g. Safari browser.
2. taps on "Translate" built-in menu item button in the `UIEditMenuInteraction`. Translate button is available by-default for any selected text throughout iOS system.
3. display custom `UISheetPresentationController` above the active app with selected text, copied selected text to active modal controller and translates it (by design). Notice the translation app is configurable in iOS systems settings starting from iOS 17.4+ having public access to the Translation framework.
4. create new chat with ai agent, e.g. new chat with claude, where ai agent will search the information about the selected text's topic and dive into it while browsing initial one.

Problem: based on the [[file:CONCEPT-ios-translate-callout-item.md]] research the `TranslationUIProvider` extension point opened to third-party default translation apps in iOS 18.4 that require to build your own translation application, that constraint the UI/UX and product concept of the own application. I do not know how strict Apple would review the app core idea and its functionality, where the Translate is just the way to work around the quick access to ai agent chat (no translation designed in my app by default, as part of the core feature) to discuss the topic instead of translation.

### Alternatives

1. Build your own browser. :)

That option theoretically allows to manage the core user scenario: find the source of information, read it, keep it to read it later and deal with text from iOS standpoint. Again, the own application like browser gives full control over `UIEditMenuProvider`, `UISHeetPresentationController` instead of abusing and working around `TranslationUIProvider` public api.

## Open Questions

1. How to access claude.ai chat history?
2. How to create new chat with claude?
