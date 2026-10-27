# AGENTS.md — Flycut

Flycut is a menu-bar clipboard manager for developers (fork of TermiT/Flycut, itself based on Jumpcut). **Scope: macOS app only.** The repo also has an iOS app (`Flycut-iOS*`, `InAppSettingsKit/`, `MGSwipeTableCell/`, `help.iOS.md`); ignore it and don't modify it. License: MIT. Current version: see `CHANGELOG.md`.

## Tech stack
- **macOS app**: Objective-C, AppKit, **manual retain/release** (the Flycut target doesn't set `CLANG_ENABLE_OBJC_ARC`, so ARC is off; code uses explicit `retain`/`release`). Deployment target macOS 10.13. UI is a `.nib` (`English.lproj/MainMenu.nib`) — edit with Xcode, not by hand.
- Project: `Flycut.xcodeproj` (no workspace/SPM/CocoaPods; dependencies are vendored). Building needs full Xcode (`xcodebuild` doesn't work with CLT only).
- Relevant targets: **Flycut** (the app) and **FlycutHelper** (launch-at-login helper, ARC).
- No macOS unit tests exist. No shared `.xcscheme` files are checked in, so Xcode generates the schemes.

## Layout
| Path | Role |
| --- | --- |
| `main.m`, `AppController.{h,m}` | macOS app controller (~1500 lines): menu-bar item, menu building/search, bezel, hotkeys, pasteboard polling, preferences, paste synthesis. Also the delegate for store updates. |
| `FlycutOperator.{h,m}` | Core logic, separate from the UI: owns three stores, stack position, add/dedupe/skip rules, save/load, iCloud sync. |
| `FlycutEngine/FlycutStore.*`, `FlycutClipping.*` | Data model: `FlycutStore` is an ordered `jcList` of `FlycutClipping` (contents, type, source app name/bundle URL, timestamp, cached display string), with a remember limit and delegate callbacks (`insert/delete/move/reload`). |
| `UI/` | `BezelWindow` (the HUD shown on hotkey), rounded-rect helpers, zoom effect. |
| `FlycutHelper/`, `LoginItem/` | Launch-at-login helper app / login-item APIs. |
| `SGHotKeysLib/`, `ShortcutRecorder/` | Global hotkey registration and recorder control (vendored). |
| `UKPrefsPanel/` | Preferences toolbar panel (vendored). |
| `MJCloudKitUserDefaultsSync/` | Syncs UserDefaults keys via CloudKit (vendored). |
| `*.entitlements`, `configureAppSandboxing.sh` | Sandboxed (App Store), Developer ID, and Debug entitlements. |
| `.github/workflows/release.yml`, `RELEASE_SETUP.md` | Tag-triggered (`v*`) build → sign → notarize → DMG → GitHub Release. |
| `help.md`, `CHANGELOG.md` | User docs / release notes — update when behavior changes. |

## How it works
1. **Capture**: `AppController -pollPB:` runs on a 1s `NSTimer` (common run-loop modes so it still fires while the menu is open). It compares `NSPasteboard.changeCount`, reads the string on a background queue (can block for large/remote copies), runs `FlycutOperator -shouldSkip:…` (password/pasteboard-type skip lists from prefs), then **dispatches back to the main queue** to call `addClipping:`. Only plain-text (`NSStringPboardType`) is captured. By default, `skipPboardTypesList` already skips concealed, transient, and auto-generated types (`org.nspasteboard.*`), `PasswordPboardType`, and 1Password.
2. **Store**: Three `FlycutStore`s — `clippingStore` (main history), `favoritesStore`, `stashedStore` (holds the main store while viewing favorites). `stackPosition` is the current selection. Duplicates are kept by default. With `removeDuplicates` on, a repeated clip moves to the top. `pasteMovesToTop` is also off by default.
3. **Display**: Status-bar menu (with search field; activates the app briefly to give the field focus, then reactivates the prior app) and the **Bezel** (⇧⌘V default; arrows, `j`/`k`, Home/End, PageUp/PageDown, and digits navigate; Return pastes; `s`/`S` saves, `f`/`F` handles favorites, and Backspace removes the item (forward-delete is ignored). All of this is in `processBezelKeyDown:`. The bezel has no search field; search exists only in the menu).
4. **Paste**: Puts the item on `NSPasteboard`, then synthesizes ⌘V with `CGEvent` via `fakeKey:withCommandFlag:` using the physical key code `kVK_ANSI_V` (needs Accessibility permission).
5. **Persistence**: `-saveEngine` serializes stores into a dictionary (`version 0.7`, `jcList`, `favoritesList`) stored in **NSUserDefaults key `store`** (plain, unencrypted), gated by the `savePreference` pref (0 = never, 1 = on exit, 2 = after each clip). Optional file export via `saveToLocation`/`autoSaveToLocation`.
6. **Sync**: Optional iCloud sync of settings and clippings through `MJCloudKitUserDefaultsSync` (degrades gracefully in unsigned builds without entitlements).

## Conventions & gotchas
- MRC: balance `retain`/`release`; don't paste ARC-only idioms into macOS files.
- `FlycutOperator` and `FlycutEngine/` are also compiled into the iOS target. Only the macOS build needs to keep working; breaking the iOS build is acceptable.
- Thread safety: `FlycutStore.jcList` is not thread-safe; mutate only on the main thread (previous crash fixes in 1.9.7).
- Preferences are `NSUserDefaults` keys with defaults registered in `AppController -init` (UI/hotkey/bezel) and `FlycutOperator -init` (store, skip, and sync keys); many keys also bound in the nib and appear in `settingsSyncList`.
- UI changes usually require nib edits; menu-bar click behavior is fragile on newer macOS (see latest commit).
- Header comments credit "Gennadiy Potapov and contributors. Based on Jumpcut by Steve Cook" — keep on new files.
- Verify with a manual run in Xcode: copy text, open bezel, paste, check menu search; no automated tests cover this.

## Build / release
- Open `Flycut.xcodeproj` in Xcode, then build the **Flycut** target (Xcode generates the schemes). Requires full Xcode; Command Line Tools alone can't run `xcodebuild`.
- Release: push a `v*` tag; see `RELEASE_SETUP.md` for required secrets.
