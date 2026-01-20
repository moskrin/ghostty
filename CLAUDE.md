# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

Building from Git checkout on Linux requires `blueprint-compiler` (0.16.0+). Use `nix develop` for a complete dev environment, or install it separately.

```bash
zig build                              # Build (debug mode)
zig build run                          # Build and run
zig build test                         # Run all tests
zig build test -Dtest-filter=<name>    # Run specific test
zig fmt .                              # Format Zig code
prettier -w .                          # Format docs/resources
```

### libghostty-vt (VT parsing library)

```bash
zig build lib-vt                           # Build library
zig build lib-vt -Dtarget=wasm32-freestanding  # Build Wasm
zig build test-lib-vt                      # Run tests
zig build test-lib-vt -Dtest-filter=<name> # Run specific test
```

When working on libghostty-vt, don't build the full app. For C-only changes, build examples instead of running Zig tests.

### macOS App

Do NOT use `xcodebuild`. Use `zig build` for everything including running Xcode tests. Main branch requires Xcode 26 and macOS 26 SDK (can build on macOS 15).

### Memory Leak Detection (Linux)

```bash
zig build run-valgrind    # Run under Valgrind with proper suppression flags
```

## Directory Structure

- `src/` - Shared Zig core
- `include/` - C API headers
- `macos/` - macOS app (Swift/Objective-C)
- `src/apprt/gtk/` - GTK app (Linux/FreeBSD)
- `src/terminal/` - VT emulation core
- `src/renderer/` - GPU rendering (Metal/OpenGL/WebGL)
- `src/font/` - Font rendering and shaping
- `src/input/` - Keyboard/mouse handling
- `src/config/` - Configuration system

## Architecture Overview

### Application Layers

```
App (src/App.zig) - Manages surfaces, font cache, config
  └── Surface (src/Surface.zig) - Single terminal instance
        ├── Terminal - VT emulation state machine
        ├── IO Thread - PTY read/write
        └── Renderer Thread - Async GPU rendering
```

### Terminal Emulation (src/terminal/)

- `Terminal.zig` - Main state machine
- `Parser.zig` - VT sequence parser (DEC ANSI state machine)
- `Screen.zig` / `PageList.zig` - Grid state and scrollback
- `csi.zig`, `osc.zig`, `dcs.zig` - Sequence handlers

### Data Flow

```
PTY → IO Thread → Parser → Terminal → Screen → Renderer Thread → GPU
User Input → Input Handler → (keybinding or PTY write)
```

### Platform Abstraction (apprt)

Compile-time selection via `src/apprt.zig`:
- `gtk` - Linux/FreeBSD
- Native macOS via Swift (bridged through `macos/`)
- `embedded` - C API without native UI
- `browser` - WASM/WebGL

### Threading Model

Each Surface spawns 3 threads:
1. **IO Thread** - PTY reads/writes, child process monitoring
2. **Renderer Thread** - Glyph shaping, GPU buffer updates, frame rendering
3. **Main Thread** - User input, config, IPC

## AI Contribution Policy

**All AI assistance must be disclosed in PRs.** Example: "This PR was written primarily by Claude Code."

Requirements:
- Must test on all impacted platforms (don't write GTK code on macOS with AI alone, or vice versa)
- Must understand and be able to answer questions about AI-generated code
- PRs with no visible human accountability may be closed without hesitation
- No AI-generated media (artwork, icons, assets)
- Community interactions (issue comments, PR descriptions) must be human-composed

## Testing

### Input Stack Changes

If modifying the input stack (key events → PTY text), manually test this matrix on Linux:

| Windowing | IME | Input Types |
|-----------|-----|-------------|
| Wayland, X11 | ibus, fcitx, none | Dead keys, CJK, Emoji, Unicode Hex |

Also test ibus versions 1.5.29, 1.5.30, 1.5.31 (different behaviors).

## Logging

```bash
# Linux (systemd)
journalctl --user --unit app-com.mitchellh.ghostty.service

# macOS
sudo log stream --level debug --predicate 'subsystem=="com.mitchellh.ghostty"'
```

Control via `GHOSTTY_LOG` env var: `stderr`, `macos`, `true`, `false`, or combine with commas. Prefix with `no-` to disable (e.g., `stderr,no-macos`).

## Linting

CI enforces:
- `zig fmt` for Zig code
- `prettier` for docs/resources (version must match devShell.nix)
- `alejandra` for Nix files (version must match devShell.nix)
- `shellcheck --check-sourced --severity=warning` for bash scripts
