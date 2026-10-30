# Changelog

All notable changes to TrollVNC are documented here.

## [Unreleased]

### Added
- **Image Quality (0–9)**: Image quality is now an independent option instead of being tied to the output scale. A new `Quality` preference (`-q auto|0..9` on the CLI) forces the server-side Tight/JPEG quality level and overrides whatever the viewer negotiates in `SetEncodings`. The legacy behaviour is preserved by the default `auto` (`-1`): when the preference is unset or `-q auto` is used, the viewer’s negotiated quality is kept verbatim. Encoders without a quality parameter (ZRLE, Raw, Hextile) are unaffected.
- **Managed Configuration**: `Quality` is now a supported numeric key (0..9, or `-1` to follow the viewer).
- The *Image Quality* slider has an explicit **Auto** slot as its leftmost step (`-1`), meaning the server follows the quality negotiated by the viewer; `0..9` force that level.

### Changed
- `-s` / `Scale` now unambiguously controls output resolution only; changing the scale no longer implies a change in encoded image quality. Use `-q` / `Quality` for quality.
- The preferences pane gained an *Image Quality* slider with 11 stops (`-1` … `9`), and the *Output Scale* footer text no longer conflates scale with quality.

## [3.2] – 2026-04-19

### Added
- **Bind Address**: The VNC server can now be bound to a specific local IP address. Leave empty to listen on all interfaces. Validation is performed in the app UI before applying.
- **Orientation Correction** (formerly *Apply Orientation Fix*): Replaced the on/off toggle with a four-option selector — 0° (Default), 90° Clockwise, 180°, and Counterclockwise 90°. Useful for iPad models with non-standard default orientations.
- **Subscription Reconnection**: The client list now automatically reconnects to the VNC server daemon with exponential backoff (1s → 30s cap) and TCP keep-alive, recovering gracefully from server restarts.
- **Simulator Sandbox Support**: `sim-spawn.sh` now forwards the simulator sandbox path as an environment variable so the server reads preferences from the correct location during development. Logs are tee’d to both the terminal and the sandbox tmp directory.

### Changed
- Xcode app target now reads and writes preferences via the daemon’s sandbox path when running in the simulator.

### Fixed
- Version check now falls back gracefully on older iOS versions where the system API is unavailable.
- Fixed a short-read bug in `TVNCReadAll`: data is now read until EOF or timeout instead of stopping early on a partial buffer.
- Fixed subscription handshake validation: the server response is now checked to contain “OK” before the client list is considered live; an invalid response triggers an immediate reconnect.
- Fixed `TVNCSliderCell` reuse bugs: `prepareForReuse` resets the slider to `minimumValue`, and `refreshCellContentsWithSpecifier:` fully re-applies `min`/`max`/`format`/`isContinuous` from the incoming specifier.
