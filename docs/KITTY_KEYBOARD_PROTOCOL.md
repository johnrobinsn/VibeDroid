# Extended Keyboard Protocol Implementation

## Overview

VibeTTY implements two extended keyboard protocols that allow terminal applications to receive full modifier information for keys that traditionally don't report modifiers (Enter, Tab, Backspace, Escape):

1. **Kitty Keyboard Protocol** (CSI u format) - Modern protocol used by kitty, WezTerm, and modern CLI apps
2. **xterm modifyOtherKeys** (CSI 27 format) - Older protocol used by tmux and xterm

**Primary Use Case**: Applications like Claude Code can distinguish between Enter (submit) and Shift+Enter (newline) for multi-line input.

## Protocol Comparison

| Feature | Kitty Protocol | modifyOtherKeys |
|---------|---------------|-----------------|
| Format | `CSI keycode ; mod u` | `CSI 27 ; mod ; keycode ~` |
| Example (Shift+Enter) | `ESC[13;2u` | `ESC[27;2;13~` |
| Query support | Yes (`CSI ? u`) | No |
| Mode stack | Yes | No (single mode) |
| Used by | kitty, WezTerm, modern apps | tmux, xterm |
| User preference required | Yes | No (auto-enabled) |

## Kitty Keyboard Protocol

### Key Encoding Format

```
CSI unicode-key-code ; modifiers u
```

Where:
- `CSI` = `ESC [` (0x1B 0x5B)
- `unicode-key-code` = Unicode codepoint of the key
- `modifiers` = modifier bits + 1
- `u` = final character indicating CSI u format

### Control Sequences

| Sequence | Direction | Purpose |
|----------|-----------|---------|
| `CSI > flags u` | App → Terminal | Push/enable mode with flags |
| `CSI < u` | App → Terminal | Pop/disable mode |
| `CSI < count u` | App → Terminal | Pop multiple modes |
| `CSI ? u` | App → Terminal | Query current mode |
| `CSI ? flags u` | Terminal → App | Response to query |

### Progressive Enhancement Flags

| Flag | Value | Description |
|------|-------|-------------|
| FLAG_DISAMBIGUATE | 1 | Disambiguate escape codes |
| FLAG_REPORT_EVENT_TYPES | 2 | Report key press/repeat/release |
| FLAG_REPORT_ALTERNATE_KEYS | 4 | Report shifted/base layout keys |
| FLAG_REPORT_ALL_KEYS | 8 | Report all keys as escape codes |
| FLAG_REPORT_TEXT | 16 | Report associated text |

### Activation

The Kitty protocol requires **two conditions** to be active:

1. **User preference enabled**: Settings → Keyboard → "Enhanced keyboard protocol"
2. **Application requests it**: App sends `CSI > flags u`

## xterm modifyOtherKeys Protocol

### Key Encoding Format

```
CSI 27 ; modifiers ; keycode ~
```

Where:
- `27` = fixed marker for modifyOtherKeys
- `modifiers` = modifier bits + 1
- `keycode` = ASCII/Unicode value of the key

### Control Sequence

| Sequence | Purpose |
|----------|---------|
| `CSI > 4 ; 0 m` | Disable modifyOtherKeys |
| `CSI > 4 ; 1 m` | Enable mode 1 (some keys) |
| `CSI > 4 ; 2 m` | Enable mode 2 (all keys) |
| `CSI > 4 m` | Disable (mode defaults to 0) |

### Activation

modifyOtherKeys is automatically enabled when an application sends `CSI > 4 ; mode m`. No user preference is required.

## Modifier Encoding (Both Protocols)

| Modifier | Bit Value | Transmitted Value |
|----------|-----------|-------------------|
| Shift    | 1         | 2 (1+1)           |
| Alt      | 2         | 3 (2+1)           |
| Ctrl     | 4         | 5 (4+1)           |
| Shift+Ctrl | 5       | 6 (5+1)           |
| Shift+Alt | 3        | 4 (3+1)           |

## Examples

| Key Combination | Kitty Format | modifyOtherKeys Format |
|-----------------|--------------|------------------------|
| Shift+Enter     | `ESC[13;2u`  | `ESC[27;2;13~`        |
| Ctrl+Enter      | `ESC[13;5u`  | `ESC[27;5;13~`        |
| Shift+Tab       | `ESC[9;2u`   | `ESC[27;2;9~`         |
| Alt+Backspace   | `ESC[127;3u` | `ESC[27;3;127~`       |

## Architecture

### Key Files

#### termlib

| File | Purpose |
|------|---------|
| `KittyKeyboardProtocol.kt` | Kitty protocol state, mode stack, key encoding |
| `ModifyOtherKeysProtocol.kt` | modifyOtherKeys state and key encoding |
| `TerminalEmulator.kt` | CSI sequence handling, protocol integration |
| `Terminal.cpp` | Native CSI fallback handler |

#### app

| File | Purpose |
|------|---------|
| `TerminalKeyListener.kt` | Key event handling, passes modifiers |
| `TerminalBridge.kt` | Transport operations |
| `PreferenceConstants.kt` | `KITTY_KEYBOARD_PROTOCOL` setting |
| `SettingsScreen.kt` | UI toggle for Kitty protocol |

### Key Dispatch Flow

```kotlin
override fun dispatchKey(modifiers: Int, key: Int) {
    // Check Kitty protocol first (requires user preference + app request)
    if (kittyProtocol.shouldEncode(key, modifiers)) {
        val encoded = kittyProtocol.encodeKey(key, modifiers)
        if (encoded != null) {
            handler.post { onKeyboardInput.invoke(encoded) }
            return
        }
    }

    // Check modifyOtherKeys (auto-enabled by apps like tmux)
    if (modifyOtherKeysProtocol.shouldEncode(key, modifiers)) {
        val encoded = modifyOtherKeysProtocol.encodeKey(key, modifiers)
        if (encoded != null) {
            handler.post { onKeyboardInput.invoke(encoded) }
            return
        }
    }

    // Default: let native terminal handle it
    terminalNative.dispatchKey(modifiers, key)
}
```

## tmux Configuration

tmux uses modifyOtherKeys to communicate with the outer terminal. For Shift+Enter and other extended keys to work inside tmux:

### Required tmux.conf Settings

```bash
# Enable extended keys (required)
set -g extended-keys always
```

### How It Works

1. tmux sends `CSI > 4 ; 1 m` to VibeTTY to request modifyOtherKeys
2. VibeTTY enables modifyOtherKeys mode
3. User presses Shift+Enter
4. VibeTTY sends `ESC[27;2;13~` to tmux
5. tmux decodes it and forwards to the pane (in CSI u format: `ESC[13;2u`)

### tmux Version Notes

- **tmux 3.4**: Use `extended-keys always` for best compatibility
- **tmux 3.5+**: Better native support, `extended-keys on` may suffice

## Testing

### Direct Connection (No tmux)

1. Enable setting: Settings → Keyboard → "Enhanced keyboard protocol"
2. Connect to any host
3. Run `cat -v`
4. Press Shift+Enter
5. Expected: `^[[13;2u` (Kitty format)

### With tmux

1. Ensure `set -g extended-keys always` in `~/.tmux.conf`
2. Restart tmux: `tmux kill-server && tmux`
3. Run `cat -v` inside tmux
4. Press Shift+Enter
5. Expected: `^[[13;2u` (tmux converts to CSI u format)

### Protocol Negotiation Test (Kitty)

```bash
# Query current mode
printf '\e[?u'

# Push mode with disambiguate flag
printf '\e[>1u'

# Query again
printf '\e[?u'

# Pop mode
printf '\e[<u'
```

### Protocol Test (modifyOtherKeys)

```bash
# Enable mode 2
printf '\e[>4;2m'

# Run cat -v and press Shift+Enter
cat -v
# Should show: ^[[27;2;13~

# Disable
printf '\e[>4;0m'
```

## Keys Supported

| Key | Unicode Codepoint |
|-----|-------------------|
| Enter | 13 |
| Tab | 9 |
| Backspace | 127 |
| Escape | 27 |

## Compatibility

### Applications Using Kitty Protocol
- Kitty terminal
- WezTerm
- foot
- Claude Code
- Neovim (with configuration)

### Applications Using modifyOtherKeys
- tmux
- xterm
- GNU Emacs

## References

- [Kitty Keyboard Protocol Specification](https://sw.kovidgoyal.net/kitty/keyboard-protocol/)
- [xterm modifyOtherKeys](https://invisible-island.net/xterm/manpage/xterm.html#VT100-Widget-Resources:modifyOtherKeys)
- [tmux extended-keys](https://github.com/tmux/tmux/wiki/Modifier-Keys)
