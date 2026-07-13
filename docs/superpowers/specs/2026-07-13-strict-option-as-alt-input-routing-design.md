# Strict Option-as-Alt Input Routing

## Problem

cmux currently sends printable Option key events through AppKit text interpretation before forwarding them to embedded Ghostty. AppKit treats combinations such as Option-E and Option-N as dead-key input, so `macos-option-as-alt = true` can produce `´` or begin composition instead of sending an Alt/Meta key to Ghostty.

## Desired behavior

When a terminal has focus and Ghostty's effective `macos-option-as-alt` setting treats the pressed physical Option key as Alt, cmux must send that key directly to Ghostty without passing it through AppKit text interpretation.

- `true` applies to both Option keys.
- `left` and `right` apply only to the configured physical side.
- Option-E is available to an `alt+e` binding or Ghostty's Alt/Meta fallback encoding.
- Option-N is sent as Alt/Meta-N and does not begin macOS dead-key composition.
- Option keys that Ghostty does not treat as Alt retain the existing AppKit composition behavior.
- Non-Option IME and text input retain the existing behavior.
- Application-level shortcuts consumed before the event reaches the terminal are unchanged.

## Design

In `GhosttyNSView.keyDown(with:)`, compute Ghostty's translated modifiers before calling `interpretKeyEvents`. A direct Option-as-Alt path is selected when the original event contains Option and Ghostty's translated modifiers remove Alt for that physical Option side.

For that path, cmux builds the normal `ghostty_input_key_s` using:

- the original physical keycode and raw modifiers, so Ghostty can match Alt bindings;
- Ghostty's translated modifier result for consumed modifiers;
- text translated without Option, so Option-E is represented by `e` rather than `´`;
- the correct press or repeat action.

cmux then sends the event through the existing Ghostty key-send helper and returns immediately. It does not call `interpretKeyEvents`, does not accumulate AppKit text, and does not start or modify dead-key composition for that event.

All other events continue through the existing AppKit/IME path. The existing dead-key preservation helper remains available for Option sides not configured as Alt.

## Failure handling

Once an event qualifies for strict Option-as-Alt routing, cmux does not fall back to AppKit if Ghostty reports the event as unhandled. Falling back would reintroduce the unwanted dead-key character and violate the strict routing contract.

If the Ghostty surface is unavailable, the existing surface recovery behavior remains responsible for input recovery.

## Tests

Add regression coverage proving that:

1. An Option event whose Alt modifier is removed by Ghostty translation bypasses `interpretKeyEvents` and is sent directly to Ghostty.
2. Option-E reaches Ghostty with Alt and unmodified text `e`, not `´`.
3. Option-N does not start dead-key composition in strict Option-as-Alt mode.
4. An Option side retained by Ghostty translation still uses the existing AppKit composition path, preserving `left` and `right` behavior.
5. Existing CJK IME and Option-side modifier tests continue to pass.

## Out of scope

- Changing cmux application shortcut precedence.
- Disabling AppKit text input globally.
- Changing Ghostty's configuration format or Alt encoding behavior.
