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
3. **[Pause/resume control fifo](#3-pauseresume-control-fifo)** — lets a client
   that grabs the keyboard (e.g. rofi) suspend `app.conf` bindings for as long
   as it is up, fixing app overrides swallowing that client's own chords.

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

- Subscribes to `window::focus`, with the window class/title already parsed
  by i3.
- Subscribes to `window::title` (for the focused window) so title-based
  `app.conf` rules keep working.
- Subscribes to `workspace::focus` so switching to an *empty* workspace resets
  to the default (non-app) bindings. i3 emits no `window::focus` when a
  workspace has no window to focus, so without this the previous window's
  `app.conf` bindings would stay active on the empty workspace.
- De-duplicates on the resolved `(class, title)`: a workspace switch fires
  `workspace::focus` and `window::focus` together, so switching between two
  workspaces showing the same app resolves the same key and is skipped — only
  genuine app changes re-apply bindings via keyd.
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

---

## 3. Pause/resume control fifo

### Problem

Clients like rofi take an X keyboard grab **without** ever taking input focus,
and map an override-redirect window. Nothing reports them as active:

- i3 never manages an override-redirect window, so it emits no `window::focus`
  and the window is absent from `get_tree()` — the `I3` adapter is blind to it.
- `get_input_focus()` still returns the window *underneath*, because a grab is
  not focus — so the generic `XMonitor` cannot see it either.

The window underneath therefore keeps its `app.conf` bindings applied over the
grab. Where a layer emulates a modifier (`[capslock_layer:C]`, so `capslock+m`
is how `Ctrl+M` is typed), any `capslock_layer.<key>` override swallows that
chord inside rofi:

```
[cursor]
capslock_layer.m = C-S-f13     # rofi never sees Ctrl+M while cursor is underneath
```

Before the i3 adapter this was masked by accident: `XMonitor.get_floating_window()`
scans for `_NET_WM_STATE_ABOVE`, which rofi sets, so rofi resolved to class
`Rofi` and matched no section. That scan is exactly what change (2) removed to
fix the always-on-top freeze, so the two bugs share one mechanism.

### Fix

`keyd-application-mapper` serves a control fifo at
`/tmp/rlocal/keyd-application-mapper/fifo`, accepting one command per line:

```
pause <pid>     # suspend app.conf bindings; keyd is reset to config defaults
resume <pid>    # drop the pause and re-apply the focused window's bindings
```

This is resolved centrally in `apply_bindings()`, not in any one monitor, so
every adapter (i3, X, Wlroots, KDE, Gnome) gets it.

- **Window events cannot clobber a pause.** A `window::title` event from the
  window underneath (a terminal updating its title mid-grab) re-resolves to the
  paused state instead of re-applying that window's bindings.
- **A dead pauser cannot pin bindings.** `<pid>` liveness is rechecked on every
  re-resolve, so a client killed before it can send `resume` is pruned rather
  than disabling `app.conf` until the daemon restarts. Holding pausers as a set
  also refcounts nesting (a rofi menu whose action opens a second rofi).
- **Redundant transitions cost nothing.** Pausing a window that had no app
  bindings resolves to the reset that is already live, so keyd is not called.

### Usage

Clients should open the fifo read-write, so that a write cannot block when the
mapper is down and is discarded rather than replayed on its next start:

```sh
[ -p "$fifo" ] || return 0
exec 3<>"$fifo"
printf 'pause %s\n' "$$" >&3
exec 3>&-
```

See `rlocal/bin/rofi` in reed's dotfiles for the full wrapper.
