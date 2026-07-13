# Strict Option-as-Alt Input Routing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Route every terminal Option key that Ghostty configures as Alt directly to embedded Ghostty, bypassing AppKit dead-key and text interpretation.

**Architecture:** Add a pure modifier-decision helper beside the existing Ghostty modifier conversion functions. In `GhosttyNSView.keyDown(with:)`, use Ghostty's effective translation modifiers to select a strict direct-send path before `interpretKeyEvents`; retain the current AppKit path when Ghostty keeps Option for composition.

**Tech Stack:** Swift, AppKit `NSEvent`, GhosttyKit C API, XCTest, Swift Testing, Xcode `cmux-unit` scheme.

## Global Constraints

- `macos-option-as-alt = true` applies to both Option keys; `left` and `right` apply only to the configured physical side.
- A qualifying Option event never falls back to AppKit, even when Ghostty reports it as unhandled.
- Option sides that Ghostty does not treat as Alt retain existing AppKit dead-key composition.
- Non-Option IME input and cmux application-level shortcut precedence remain unchanged.
- Reuse existing Ghostty key construction and send helpers; do not change Ghostty's configuration format or encoding.

---

### Task 1: Identify strict Option-as-Alt events

**Files:**
- Modify: `Sources/GhosttyKeyModifiers.swift`
- Test: `cmuxTests/GhosttyOptionAsAltModsTests.swift`

**Interfaces:**
- Consumes: original `ghostty_input_mods_e` and the value returned by `ghostty_surface_key_translation_mods`.
- Produces: `cmuxShouldRouteOptionAsAltDirectly(originalMods:ghosttyTranslationMods:) -> Bool` for `GhosttyNSView.keyDown(with:)`.

- [ ] **Step 1: Write failing modifier-decision tests**

Append these tests to `GhosttyOptionAsAltModsTests`:

```swift
    @Test func optionRemovedFromTranslationRoutesDirectlyToGhostty() {
        #expect(cmuxShouldRouteOptionAsAltDirectly(
            originalMods: ghostty_input_mods_e(rawValue: GHOSTTY_MODS_ALT.rawValue),
            ghosttyTranslationMods: GHOSTTY_MODS_NONE
        ))
    }

    @Test func optionRetainedForCompositionDoesNotRouteDirectly() {
        #expect(!cmuxShouldRouteOptionAsAltDirectly(
            originalMods: ghostty_input_mods_e(rawValue: GHOSTTY_MODS_ALT.rawValue),
            ghosttyTranslationMods: ghostty_input_mods_e(rawValue: GHOSTTY_MODS_ALT.rawValue)
        ))
    }

    @Test func nonOptionInputNeverRoutesThroughStrictAltPath() {
        #expect(!cmuxShouldRouteOptionAsAltDirectly(
            originalMods: ghostty_input_mods_e(rawValue: GHOSTTY_MODS_SHIFT.rawValue),
            ghosttyTranslationMods: GHOSTTY_MODS_NONE
        ))
    }
```

- [ ] **Step 2: Run the focused tests and confirm the expected failure**

```bash
xcodebuild -project cmux.xcodeproj -scheme cmux-unit -configuration Debug \
  -derivedDataPath build/strict-option-as-alt-tests -destination 'platform=macOS' \
  CMUX_SKIP_ZIG_BUILD=1 \
  -only-testing:cmuxTests/GhosttyOptionAsAltModsTests test
```

Expected: compilation fails because `cmuxShouldRouteOptionAsAltDirectly` is not defined.

- [ ] **Step 3: Add the pure routing decision helper**

Append this function to `Sources/GhosttyKeyModifiers.swift`:

```swift
/// Returns true when the physical event contains Option but libghostty has
/// removed Alt from text translation because that Option side acts as Alt.
nonisolated func cmuxShouldRouteOptionAsAltDirectly(
    originalMods: ghostty_input_mods_e,
    ghosttyTranslationMods: ghostty_input_mods_e
) -> Bool {
    let originalHasAlt = (originalMods.rawValue & GHOSTTY_MODS_ALT.rawValue) != 0
    let translationHasAlt = (ghosttyTranslationMods.rawValue & GHOSTTY_MODS_ALT.rawValue) != 0
    return originalHasAlt && !translationHasAlt
}
```

- [ ] **Step 4: Run the focused tests and confirm they pass**

Run the command from Step 2.

Expected: `GhosttyOptionAsAltModsTests` passes, including the three new routing-decision tests.

- [ ] **Step 5: Commit the decision helper**

```bash
git add Sources/GhosttyKeyModifiers.swift cmuxTests/GhosttyOptionAsAltModsTests.swift
git commit -m "test: define strict Option-as-Alt routing"
```

---

### Task 2: Bypass AppKit for strict Option-as-Alt keyDown events

**Files:**
- Modify: `Sources/GhosttyTerminalView.swift`
- Test: `cmuxTests/CJKIMEInputTests.swift`

**Interfaces:**
- Consumes: `cmuxShouldRouteOptionAsAltDirectly(originalMods:ghosttyTranslationMods:)` from Task 1, `ghosttyKeyEvent(for:surface:)`, `textForKeyEvent(_:)`, `shouldSendText(_:)`, and `sendGhosttyKey(_:_:)`.
- Produces: direct keyDown routing plus DEBUG-only `debugGhosttyTranslationModsOverride` for deterministic regression tests.

- [ ] **Step 1: Add failing direct-routing regression tests**

Append this DEBUG-only XCTest class to `cmuxTests/CJKIMEInputTests.swift`:

```swift
#if DEBUG
@MainActor
final class StrictOptionAsAltRoutingTests: XCTestCase {
    func testStrictOptionAsAltBypassesAppKitAndSendsUnmodifiedText() throws {
        let terminal = try makeTerminal()
        defer { terminal.window.orderOut(nil) }

        let previousTranslationOverride = GhosttyNSView.debugGhosttyTranslationModsOverride
        let previousObserver = GhosttyNSView.debugGhosttySurfaceKeyEventObserver
        let previousInterpretHook = cjkIMEInterpretKeyEventsHook
        defer {
            GhosttyNSView.debugGhosttyTranslationModsOverride = previousTranslationOverride
            GhosttyNSView.debugGhosttySurfaceKeyEventObserver = previousObserver
            cjkIMEInterpretKeyEventsHook = previousInterpretHook
            withExtendedLifetime(terminal.surface) {}
        }

        GhosttyNSView.debugGhosttyTranslationModsOverride = { mods in
            let altMask = GHOSTTY_MODS_ALT.rawValue | GHOSTTY_MODS_ALT_RIGHT.rawValue
            return ghostty_input_mods_e(rawValue: mods.rawValue & ~altMask)
        }

        installCJKIMEInterpretKeyEventsSwizzle()
        var interpretedKeyCodes: [UInt16] = []
        cjkIMEInterpretKeyEventsHook = { candidate, events in
            guard candidate === terminal.view else { return false }
            interpretedKeyCodes.append(contentsOf: events.map(\.keyCode))
            return true
        }

        var forwardedText: [String] = []
        var forwardedMods: [ghostty_input_mods_e] = []
        GhosttyNSView.debugGhosttySurfaceKeyEventObserver = { keyEvent in
            previousObserver?(keyEvent)
            guard keyEvent.action == GHOSTTY_ACTION_PRESS,
                  keyEvent.keycode == 14 || keyEvent.keycode == 45 else { return }
            forwardedMods.append(keyEvent.mods)
            forwardedText.append(keyEvent.text.map(String.init(cString:)) ?? "")
        }

        for (keyCode, unmodifiedText) in [(UInt16(14), "e"), (UInt16(45), "n")] {
            let event = try XCTUnwrap(NSEvent.keyEvent(
                with: .keyDown,
                location: .zero,
                modifierFlags: [.option],
                timestamp: ProcessInfo.processInfo.systemUptime,
                windowNumber: terminal.window.windowNumber,
                context: nil,
                characters: "",
                charactersIgnoringModifiers: unmodifiedText,
                isARepeat: false,
                keyCode: keyCode
            ))
            terminal.view.keyDown(with: event)
        }

        XCTAssertEqual(interpretedKeyCodes, [], "Strict Option-as-Alt must bypass AppKit")
        XCTAssertEqual(forwardedText, ["e", "n"])
        XCTAssertEqual(forwardedMods.count, 2)
        XCTAssertTrue(forwardedMods.allSatisfy {
            ($0.rawValue & GHOSTTY_MODS_ALT.rawValue) != 0
        })
    }

    func testOptionRetainedByGhosttyStillUsesAppKit() throws {
        let terminal = try makeTerminal()
        defer { terminal.window.orderOut(nil) }

        let previousTranslationOverride = GhosttyNSView.debugGhosttyTranslationModsOverride
        let previousInterpretHook = cjkIMEInterpretKeyEventsHook
        defer {
            GhosttyNSView.debugGhosttyTranslationModsOverride = previousTranslationOverride
            cjkIMEInterpretKeyEventsHook = previousInterpretHook
            withExtendedLifetime(terminal.surface) {}
        }

        GhosttyNSView.debugGhosttyTranslationModsOverride = { $0 }
        installCJKIMEInterpretKeyEventsSwizzle()
        var interpreted = false
        cjkIMEInterpretKeyEventsHook = { candidate, _ in
            guard candidate === terminal.view else { return false }
            interpreted = true
            return true
        }

        let event = try XCTUnwrap(NSEvent.keyEvent(
            with: .keyDown,
            location: .zero,
            modifierFlags: [.option],
            timestamp: ProcessInfo.processInfo.systemUptime,
            windowNumber: terminal.window.windowNumber,
            context: nil,
            characters: "…",
            charactersIgnoringModifiers: ";",
            isARepeat: false,
            keyCode: 41
        ))
        terminal.view.keyDown(with: event)

        XCTAssertTrue(interpreted, "An Option side retained for composition must still use AppKit")
    }

    private struct HostedTerminal {
        let surface: TerminalSurface
        let view: GhosttyNSView
        let window: NSWindow
    }

    private func makeTerminal() throws -> HostedTerminal {
        _ = NSApplication.shared
        let surface = TerminalSurface(
            tabId: UUID(),
            context: GHOSTTY_SURFACE_CONTEXT_SPLIT,
            configTemplate: nil,
            workingDirectory: nil
        )
        let hostedView = surface.hostedView
        let window = NSWindow(
            contentRect: NSRect(x: 0, y: 0, width: 360, height: 240),
            styleMask: [.titled, .closable],
            backing: .buffered,
            defer: false
        )
        let contentView = try XCTUnwrap(window.contentView)
        hostedView.frame = contentView.bounds
        hostedView.autoresizingMask = [.width, .height]
        contentView.addSubview(hostedView)
        window.makeKeyAndOrderFront(nil)
        window.displayIfNeeded()
        contentView.layoutSubtreeIfNeeded()
        hostedView.setVisibleInUI(true)
        hostedView.setActive(true)
        RunLoop.current.run(until: Date().addingTimeInterval(0.05))
        let view = try XCTUnwrap(findGhosttyNSView(in: hostedView))
        XCTAssertTrue(window.makeFirstResponder(view))
        return HostedTerminal(surface: surface, view: view, window: window)
    }
}
#endif
```

- [ ] **Step 2: Run the new tests and confirm the expected failure**

```bash
xcodebuild -project cmux.xcodeproj -scheme cmux-unit -configuration Debug \
  -derivedDataPath build/strict-option-as-alt-tests -destination 'platform=macOS' \
  CMUX_SKIP_ZIG_BUILD=1 \
  -only-testing:cmuxTests/StrictOptionAsAltRoutingTests test
```

Expected: compilation fails because `debugGhosttyTranslationModsOverride` does not exist. After adding only the test seam, the first test fails because AppKit receives key codes 14 and 45.

- [ ] **Step 3: Add deterministic translation-modifier injection for DEBUG tests**

Add beside the existing debug observers in `GhosttyNSView`:

```swift
    @MainActor static var debugGhosttyTranslationModsOverride:
        ((ghostty_input_mods_e) -> ghostty_input_mods_e)?
```

Add this helper near `ghosttyKeyEvent(for:surface:)`:

```swift
    private func ghosttyTranslationMods(
        for surface: ghostty_surface_t,
        originalMods: ghostty_input_mods_e
    ) -> ghostty_input_mods_e {
#if DEBUG
        if let override = Self.debugGhosttyTranslationModsOverride {
            return override(originalMods)
        }
#endif
        return ghostty_surface_key_translation_mods(surface, originalMods)
    }
```

Replace both direct calls to `ghostty_surface_key_translation_mods(surface, ...)` in `keyDown(with:)` and `ghosttyKeyEvent(for:surface:)` with this helper.

- [ ] **Step 4: Add the strict direct-send path before AppKit interpretation**

In `keyDown(with:)`, retain `originalMods`, compute translated modifiers and `translationEvent` once, then insert this block before `textInputInterpretationEvent` and `interpretKeyEvents`:

```swift
        if cmuxShouldRouteOptionAsAltDirectly(
            originalMods: originalMods,
            ghosttyTranslationMods: translationModsGhostty
        ) {
            var keyEvent = ghosttyKeyEvent(for: event, surface: surface)
            keyEvent.action = action
            keyEvent.composing = false

            let directText = textForKeyEvent(translationEvent)
                ?? event.charactersIgnoringModifiers
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

Define `originalMods` before translation and use it for the translation query:

```swift
        let originalMods = modsFromEvent(event)
        let translationModsGhostty = ghosttyTranslationMods(
            for: surface,
            originalMods: originalMods
        )
```

Do not check Ghostty's handled return value and do not fall back to `interpretKeyEvents` for this branch.

- [ ] **Step 5: Run focused strict-routing and modifier tests**

```bash
xcodebuild -project cmux.xcodeproj -scheme cmux-unit -configuration Debug \
  -derivedDataPath build/strict-option-as-alt-tests -destination 'platform=macOS' \
  CMUX_SKIP_ZIG_BUILD=1 \
  -only-testing:cmuxTests/StrictOptionAsAltRoutingTests \
  -only-testing:cmuxTests/GhosttyOptionAsAltModsTests test
```

Expected: both suites pass. The strict test observes text `e` and `n` with raw Alt while observing zero AppKit interpretations.

- [ ] **Step 6: Run neighboring IME and dead-key regressions**

```bash
xcodebuild -project cmux.xcodeproj -scheme cmux-unit -configuration Debug \
  -derivedDataPath build/strict-option-as-alt-tests -destination 'platform=macOS' \
  CMUX_SKIP_ZIG_BUILD=1 \
  -only-testing:cmuxTests/DeadKeyCompositionRegressionTests \
  -only-testing:cmuxTests/TraditionalChineseIMENumpadRegressionTests test
```

Expected: both neighboring suites pass, proving that Option retained for composition and non-Option CJK IME input are unchanged.

- [ ] **Step 7: Commit the direct routing implementation**

```bash
git add Sources/GhosttyTerminalView.swift cmuxTests/CJKIMEInputTests.swift
git commit -m "fix: route Option-as-Alt directly to Ghostty"
```

---

### Task 3: Final verification

**Files:**
- Verify: `Sources/GhosttyKeyModifiers.swift`
- Verify: `Sources/GhosttyTerminalView.swift`
- Verify: `cmuxTests/GhosttyOptionAsAltModsTests.swift`
- Verify: `cmuxTests/CJKIMEInputTests.swift`

**Interfaces:**
- Consumes: the completed direct routing implementation from Tasks 1 and 2.
- Produces: a clean build, whitespace validation, and a clean committed worktree.

- [ ] **Step 1: Run whitespace validation**

```bash
git diff --check HEAD~2..HEAD
```

Expected: no output and exit status 0.

- [ ] **Step 2: Build the macOS app target**

```bash
xcodebuild -project cmux.xcodeproj -scheme cmux -configuration Debug \
  -derivedDataPath build/strict-option-as-alt-app -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO build
```

Expected: `** BUILD SUCCEEDED **`.

- [ ] **Step 3: Confirm commit and worktree state**

```bash
git status --short --branch
git log -3 --oneline
```

Expected: no uncommitted source or test changes; the latest implementation commits describe strict Option-as-Alt routing.
