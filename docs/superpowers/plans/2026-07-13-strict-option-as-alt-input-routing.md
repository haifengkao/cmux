# Strict Option-as-Alt Input Routing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Route every terminal Option key that Ghostty configures as Alt directly to embedded Ghostty, bypassing AppKit dead-key and text interpretation.

**Architecture:** Add a tested production routing value beside the existing Ghostty modifier conversion functions. In `GhosttyNSView.keyDown(with:)`, use that value to either send the translated key directly to Ghostty and return, or continue through the existing AppKit/IME path.

**Tech Stack:** Swift, AppKit `NSEvent`, GhosttyKit C API, Swift Testing, XCTest, Xcode `cmux-unit` scheme.

## Global Constraints

- `macos-option-as-alt = true` applies to both Option keys; `left` and `right` apply only to the configured physical side.
- A qualifying Option event never falls back to AppKit, even when Ghostty reports it as unhandled.
- Option sides that Ghostty does not treat as Alt retain existing AppKit dead-key composition.
- Non-Option IME input and cmux application-level shortcut precedence remain unchanged.
- No DEBUG-only routing overrides or test-only production APIs.

---

### Task 1: Model and test the Option input route

**Files:**
- Modify: `Sources/GhosttyKeyModifiers.swift`
- Test: `cmuxTests/GhosttyOptionAsAltModsTests.swift`

**Interfaces:**
- Consumes: an `NSEvent`, original `ghostty_input_mods_e`, Ghostty translation modifiers, and translated AppKit modifier flags.
- Produces: `CmuxOptionKeyInputRoute` and `cmuxOptionKeyInputRoute(event:originalMods:ghosttyTranslationMods:translationFlags:)`.

- [ ] **Step 1: Write failing routing tests**

Append these helpers and tests to `GhosttyOptionAsAltModsTests`:

```swift
    @Test func optionAsAltRoutesEAndNDirectlyWithUnmodifiedText() throws {
        let cases: [(UInt16, String)] = [(14, "e"), (45, "n")]

        for (keyCode, unmodifiedText) in cases {
            let event = try #require(NSEvent.keyEvent(
                with: .keyDown,
                location: .zero,
                modifierFlags: [.option],
                timestamp: 1,
                windowNumber: 0,
                context: nil,
                characters: "",
                charactersIgnoringModifiers: unmodifiedText,
                isARepeat: false,
                keyCode: keyCode
            ))

            #expect(cmuxOptionKeyInputRoute(
                event: event,
                originalMods: ghostty_input_mods_e(rawValue: GHOSTTY_MODS_ALT.rawValue),
                ghosttyTranslationMods: GHOSTTY_MODS_NONE,
                translationFlags: []
            ) == .ghostty(text: unmodifiedText))
        }
    }

    @Test func optionRetainedForCompositionUsesAppKit() throws {
        let event = try #require(NSEvent.keyEvent(
            with: .keyDown,
            location: .zero,
            modifierFlags: [.option],
            timestamp: 1,
            windowNumber: 0,
            context: nil,
            characters: "…",
            charactersIgnoringModifiers: ";",
            isARepeat: false,
            keyCode: 41
        ))

        #expect(cmuxOptionKeyInputRoute(
            event: event,
            originalMods: ghostty_input_mods_e(rawValue: GHOSTTY_MODS_ALT.rawValue),
            ghosttyTranslationMods: ghostty_input_mods_e(rawValue: GHOSTTY_MODS_ALT.rawValue),
            translationFlags: [.option]
        ) == .appKit)
    }

    @Test func nonOptionInputUsesAppKit() throws {
        let event = try #require(NSEvent.keyEvent(
            with: .keyDown,
            location: .zero,
            modifierFlags: [.shift],
            timestamp: 1,
            windowNumber: 0,
            context: nil,
            characters: "E",
            charactersIgnoringModifiers: "e",
            isARepeat: false,
            keyCode: 14
        ))

        #expect(cmuxOptionKeyInputRoute(
            event: event,
            originalMods: ghostty_input_mods_e(rawValue: GHOSTTY_MODS_SHIFT.rawValue),
            ghosttyTranslationMods: ghostty_input_mods_e(rawValue: GHOSTTY_MODS_SHIFT.rawValue),
            translationFlags: [.shift]
        ) == .appKit)
    }
```

- [ ] **Step 2: Run the focused tests and confirm the expected RED state**

```bash
xcodebuild -project cmux.xcodeproj -scheme cmux-unit -configuration Debug \
  -derivedDataPath build/strict-option-as-alt-tests -destination 'platform=macOS' \
  CMUX_SKIP_ZIG_BUILD=1 \
  -only-testing:cmuxTests/GhosttyOptionAsAltModsTests test
```

Expected: compilation fails because `CmuxOptionKeyInputRoute` and `cmuxOptionKeyInputRoute` do not exist.

- [ ] **Step 3: Implement the minimal production routing value**

Append this code to `Sources/GhosttyKeyModifiers.swift`:

```swift
enum CmuxOptionKeyInputRoute: Equatable {
    case appKit
    case ghostty(text: String?)
}

nonisolated func cmuxOptionKeyInputRoute(
    event: NSEvent,
    originalMods: ghostty_input_mods_e,
    ghosttyTranslationMods: ghostty_input_mods_e,
    translationFlags: NSEvent.ModifierFlags
) -> CmuxOptionKeyInputRoute {
    let originalHasAlt = (originalMods.rawValue & GHOSTTY_MODS_ALT.rawValue) != 0
    let translationHasAlt = (ghosttyTranslationMods.rawValue & GHOSTTY_MODS_ALT.rawValue) != 0
    guard originalHasAlt, !translationHasAlt else { return .appKit }

    let translatedText = event.characters(byApplyingModifiers: translationFlags)
        .flatMap { $0.isEmpty ? nil : $0 }
        ?? event.charactersIgnoringModifiers
    return .ghostty(text: translatedText)
}
```

- [ ] **Step 4: Run the focused tests and confirm GREEN**

Run the command from Step 2.

Expected: `GhosttyOptionAsAltModsTests` passes with 18 tests.

- [ ] **Step 5: Commit the routing model**

```bash
git add Sources/GhosttyKeyModifiers.swift cmuxTests/GhosttyOptionAsAltModsTests.swift
git commit -m "test: define strict Option-as-Alt routing"
```

---

### Task 2: Apply the route before AppKit interpretation

**Files:**
- Modify: `Sources/GhosttyTerminalView.swift`
- Modify: `Sources/GhosttyNSView+IMEComposition.swift`
- Test: `cmuxTests/CJKIMEInputTests.swift`

**Interfaces:**
- Consumes: `cmuxOptionKeyInputRoute(event:originalMods:ghosttyTranslationMods:translationFlags:)`, `ghosttyKeyEvent(for:surface:)`, `shouldSendText(_:)`, and `sendGhosttyKey(_:_:)`.
- Produces: a strict direct-send branch before `textInputInterpretationEvent` and `interpretKeyEvents`.

- [ ] **Step 1: Insert the tested routing decision before AppKit**

In `keyDown(with:)`, retain the original Ghostty modifiers and use them for translation:

```swift
        let originalMods = modsFromEvent(event)
        let translationModsGhostty = ghostty_surface_key_translation_mods(surface, originalMods)
```

After constructing `translationEvent`, switch on the tested route before constructing `textInputEvent`:

```swift
        switch cmuxOptionKeyInputRoute(
            event: event,
            originalMods: originalMods,
            ghosttyTranslationMods: translationModsGhostty,
            translationFlags: translationMods
        ) {
        case .appKit:
            break
        case .ghostty(let directText):
            var keyEvent = ghosttyKeyEvent(for: event, surface: surface)
            keyEvent.action = action
            keyEvent.composing = false

            if let directText, shouldSendText(directText) {
                directText.withCString { ptr in
                    keyEvent.text = ptr
                    _ = sendGhosttyKey(surface, keyEvent)
                }
            } else {
                keyEvent.text = nil
                _ = sendGhosttyKey(surface, keyEvent)
            }
            return
        }
```

The unconditional return is required: strict Option-as-Alt input must not fall back to AppKit when Ghostty returns false.

- [ ] **Step 2: Remove the obsolete configured-Alt dead-key regression**

Delete `DeadKeyCompositionRegressionTests.testOptionTildeDeadKeyUsesOriginalEventBeforeAltTranslation()` from `cmuxTests/CJKIMEInputTests.swift`. Its assertion that configured Alt-N must compose `ã` is the behavior this change intentionally removes. Retained Option composition remains covered by `optionRetainedForCompositionUsesAppKit` and the keyboard-layout composition tests.

The direct branch makes `textInputInterpretationEvent` unreachable for an Option side whose Alt modifier Ghostty removed. Replace its only call with `translationEvent` and remove the obsolete helper from `Sources/GhosttyNSView+IMEComposition.swift`.

- [ ] **Step 3: Run routing and neighboring IME regression tests**

```bash
xcodebuild -project cmux.xcodeproj -scheme cmux-unit -configuration Debug \
  -derivedDataPath build/strict-option-as-alt-tests -destination 'platform=macOS' \
  CMUX_SKIP_ZIG_BUILD=1 \
  -only-testing:cmuxTests/GhosttyOptionAsAltModsTests \
  -only-testing:cmuxTests/TraditionalChineseIMENumpadRegressionTests test
```

Expected: all selected suites pass. The routing suite proves E/N direct routing and retained Option composition; the existing IME suite proves non-Option Chinese input remains unchanged.

- [ ] **Step 4: Commit the direct routing implementation**

```bash
git add Sources/GhosttyTerminalView.swift Sources/GhosttyNSView+IMEComposition.swift \
  cmuxTests/CJKIMEInputTests.swift \
  docs/superpowers/plans/2026-07-13-strict-option-as-alt-input-routing.md
git commit -m "fix: route Option-as-Alt directly to Ghostty"
```

---

### Task 3: Final verification

**Files:**
- Verify: `Sources/GhosttyKeyModifiers.swift`
- Verify: `Sources/GhosttyTerminalView.swift`
- Verify: `Sources/GhosttyNSView+IMEComposition.swift`
- Verify: `cmuxTests/GhosttyOptionAsAltModsTests.swift`
- Verify: `cmuxTests/CJKIMEInputTests.swift`

**Interfaces:**
- Consumes: completed routing model and keyDown integration.
- Produces: a clean build and committed worktree.

- [ ] **Step 1: Validate diffs**

```bash
git diff --check HEAD~2..HEAD
```

Expected: no output and exit status 0.

- [ ] **Step 2: Build the macOS app target**

```bash
xcodebuild -project cmux.xcodeproj -scheme cmux -configuration Debug \
  -derivedDataPath build/strict-option-as-alt-app -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO CMUX_SKIP_ZIG_BUILD=1 build
```

Expected: `** BUILD SUCCEEDED **`.

- [ ] **Step 3: Confirm worktree state**

```bash
git status --short --branch
git log -4 --oneline
```

Expected: no uncommitted source or test changes; the latest commits contain the strict routing model and implementation.
