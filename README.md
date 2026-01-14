# VibeDroid

VibeDroid is a fork of [ConnectBot](https://github.com/connectbot/connectbot), a powerful open-source SSH client for Android.

## About This Fork

VibeDroid builds upon ConnectBot's excellent foundation, adding enhancements focused on modern mobile workflows and "vibe coding" - AI-assisted development where you need a great terminal experience on the go.

### New Features

- **Kitty Keyboard Protocol**: Full support for the [Kitty keyboard protocol](https://sw.kovidgoyal.net/kitty/keyboard-protocol/) enabling Shift+Enter, Ctrl+Enter, and other modifier combinations in modern CLI tools like Claude Code
- **Virtual Terminal Width**: Render the terminal wider than the physical screen (e.g., 120 columns on a narrow phone) with single-finger horizontal panning
- **Improved Keyboard Handling**: Better detection of connected vs. paired Bluetooth keyboards
- **Force Software Keyboard Option**: Show soft keyboard even when hardware keyboard is detected
- **Toggle Keyboard/Title Bar**: Tap the terminal to toggle UI visibility
- **Long-press Text Selection**: Works correctly with horizontal pan offset
- **Per-orientation Font Sizes**: Remember different font sizes for portrait and landscape

See [NEW_FEATURES.md](NEW_FEATURES.md) for full details and implementation notes.

## Credits

VibeDroid is built on top of **ConnectBot**, created by Kenny Root and the ConnectBot contributors. We are grateful for their years of work creating such a solid SSH client.

- **Original Project**: [ConnectBot](https://github.com/connectbot/connectbot)
- **License**: Apache License 2.0

## Building

```sh
# Build the app
./gradlew build

# Build debug APK
./gradlew assembleGoogleDebug
```

### Product Flavors

- **google**: Uses Google Play Services for crypto provider updates
- **oss**: Uses bundled Conscrypt library, fully open-source

## Original ConnectBot

For the original ConnectBot app:
- [Google Play Store](https://play.google.com/store/apps/details?id=org.connectbot)
- [GitHub Releases](https://github.com/connectbot/connectbot/releases)
- [Translations](https://translations.launchpad.net/connectbot/trunk/+pots/fortune)
