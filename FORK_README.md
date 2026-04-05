# Fork: Composite layer modifier emulation without explicit bindings

Upstream: https://github.com/rvaiya/keyd

## Problem

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

## Fix

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
