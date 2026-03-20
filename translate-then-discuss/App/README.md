# App Targets (iOS, TranslationExtension)

This directory contains all of the app targets that can be actually run on the iOS Simulator or on a physical device. Things like `TranslationExtension` could be run **only** on a physical device. There are no much things to discover in platform-specific targets, because almost all source code and resources are in SPM module being kept as project's root directory. Some things that might be interested based on the application specifics:

- `TranslationExtension`: Extension target (think wrapper).

