# keyd — reed's fork

Upstream: https://github.com/rvaiya/keyd

This is a fork of `keyd` consisting of the following changes:

1. **[Composite layer modifier emulation without explicit bindings](#1-composite-layer-modifier-emulation-without-explicit-bindings)** —
   lets a composite layer (e.g. `[capslock_layer+meta]`) claim keys via inherited
   modifier emulation even with no explicit bindings, so app-specific overrides
   no longer hijack additional modifier combinations.
2. **[Native i3 adapter for keyd-application-mapper](#2-native-i3-adapter-for-keyd-application-mapper)** —
   resolves the active window over i3's IPC socket instead of guessing from X
   state, fixing the case where an always-on-top window (e.g. Zoom) freezes
   `app.conf` bindings.

---

## 1. Composite layer modifier emulation without explicit bindings

### Problem

When using `keyd-application-mapper` with app-specific bindings (app.conf),
overriding a key in a layer (e.g. `capslock_layer.w = macro(C-backspace)`)
hijacks ALL uses of that key in the layer, even when additional modifier
layers like Meta/Super are active.

For example, with this setup:

```
# Root config (default.conf)
[capslock_layer:C]
# (emulates Ctrl for unmapped keys)

# App config (app.conf)
[telegramdesktop]
capslock_layer.w = macro(C-backspace)
```

- `CapsLock+w` in Telegram correctly sends `Ctrl+Backspace` (app override)
- `Super+CapsLock+w` ALSO fires the macro instead of sending `Ctrl+Super+w`
  to the window manager (i3), breaking global shortcuts like launching WhatsApp

There is no way to fix this with upstream keyd because:

1. Composite layers (`[capslock_layer+meta]`) are only considered during lookup
   when they have explicit key bindings (`op != 0`)
2. There is no transparent/passthrough binding type
3. The only workaround is hardcoding every key in the composite layer

### Fix

Modified `lookup_descriptor` in `src/keyboard.c` so that composite layers with
all constituents active can claim keys via inherited modifier emulation even
without explicit bindings.

When a composite layer matches (all constituents active) but has no explicit
binding for the pressed key, a `OP_KEYSEQUENCE` is synthesized with the
combined constituent modifiers. This makes the composite layer take precedence
over individual layer bindings (including app overrides).

### Usage

Add an empty composite layer to your root config:

```
[capslock_layer+meta]
```

Now `Super+CapsLock+w` sends `Ctrl+Super+w` (modifier emulation from
capslock_layer's `:C` + meta's `:M`) regardless of app-specific overrides
on `capslock_layer.w`.

---

## 2. Native i3 adapter for keyd-application-mapper

### Problem

On i3 (X11), the generic `XMonitor` in `keyd-application-mapper` resolves the
active window by scanning every window for the `_NET_WM_STATE_ABOVE`
(always-on-top) flag. Apps that keep an always-on-top window alive — notably
Zoom's floating meeting-controls window — are matched by this scan and reported
as the active window indefinitely.

The result: once Zoom is running, every focus change reports `zoom`. Because
`on_window_change` only re-applies bindings when the class/title actually
changes, it then goes silent and `app.conf` bindings freeze on whatever Zoom
matched.

### Fix

Added a native `I3` adapter (`scripts/keyd-application-mapper`) that talks to
i3's IPC socket via the `i3ipc` library instead of guessing from X state:

- Subscribes to `window::focus` and re-applies bindings only when focus
  actually changes, with the window class/title already parsed by i3.
- Subscribes to `window::title` (for the focused window) so title-based
  `app.conf` rules keep working.
- Emits the currently focused window once at startup.

It is registered ahead of the generic `X` monitor and is auto-detected:
`i3ipc.Connection()` only succeeds under i3/sway, so other sessions fall
through to the existing monitors. This sidesteps the always-on-top bug class
entirely — i3 reports the exact focused container, so Zoom's floating window
can never masquerade as active.

### Usage

Install the `i3ipc` python library (`python-i3ipc` on Arch), then run
`keyd-application-mapper` as usual — it will print `i3 detected` and react to
real focus changes.
