# CLAUDE.md — wl-ronin

## Project Overview

wl-ronin is a Wayland desktop pet application for Hyprland. It renders an animated sprite (a wandering ronin) on a transparent overlay surface that sits on top of all windows. The pet wanders the screen, idles, and optionally reacts to Hyprland IPC events (window opens, workspace switches, etc.). The name follows Wayland `wl-` naming convention — a ronin is a masterless wandering samurai, which describes what the pet does.

This is a learning project. The developer (Austin) is an experienced Go programmer learning C through practical Wayland client development. The project should remain idiomatic C — no C++, no unnecessary abstractions, no over-engineering.

## Tech Stack

- **Language:** C11
- **Build system:** Meson + Ninja
- **Core dependencies:** libwayland-client, wlr-layer-shell-unstable-v1 protocol, wayland-scanner
- **Vendored:** stb_image.h (sprite loading), inih (config parsing, added later)
- **Target compositor:** Hyprland (wlroots-based, supports wlr-layer-shell-v1)
- **Rendering:** SHM (shared memory) pixel buffers — raw ARGB8888 framebuffer blitting. May migrate to Cairo later if needed.
- **Platform:** Linux only. Arch Linux is the primary dev environment.

## Architecture

The project is structured as independent modules with clear responsibilities:

```
src/
├── main.c          — entry point, CLI arg parsing, main event loop (poll)
├── wayland.c/.h    — Wayland display connection, layer-shell surface, SHM buffer management
├── render.c/.h     — sprite sheet loading (stb_image), frame slicing, blitting to buffer
├── pet.c/.h        — pet state machine (idle, walk, sit), position, animation logic
├── config.c/.h     — config file parsing + CLI overrides (deferred, not in v1)
└── hyprland.c/.h   — Hyprland IPC socket connection, event parsing (stretch goal)
```

**Data flow:** `main.c` runs the event loop → calls `pet_update()` each tick → gets buffer from `wayland.c` → passes buffer + pet position/frame to `render.c` → commits frame via `wayland.c`.

**Key Wayland concepts in this project:**
- `wlr-layer-shell-v1` protocol creates the overlay surface
- Surface is on the `overlay` layer with `exclusive_zone = -1` (doesn't push windows)
- Input region is empty (all clicks pass through)
- SHM buffers are allocated via shared memory (`shm_open` / `mmap`), attached to `wl_buffer` objects
- Frame timing uses `poll()` on the Wayland display fd + a `timerfd` for animation ticks
- Protocol bindings are generated from XML by `wayland-scanner` at build time (in `protocol/meson.build`)

**Hyprland IPC (stretch goal):**
- Socket at `$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket2.sock`
- Streams newline-delimited event strings
- Connect with standard POSIX sockets, add fd to the `poll()` set in main loop
- Pet reacts to events (e.g., looks toward new window, runs to follow workspace switch)

## Development Milestones

These are in order. Each milestone builds on the previous one:

1. **Hello Wayland** — connect to display, print success, disconnect
2. **Colored rectangle** — create wlr-layer-shell surface, draw solid color on screen
3. **Transparency + passthrough** — full-screen transparent surface, empty input region, clicks pass through
4. **Static sprite** — load PNG with stb_image, blit at fixed position on the overlay
5. **Animation loop** — timerfd + poll, move sprite each tick
6. **State machine** — idle, walk left, walk right, sit states with random transitions and animation frames
7. **Config** — CLI flags, then config file for sprite path, speed, monitor selection
8. **Hyprland IPC** — pet reacts to compositor events

## Coding Guidelines

### Style

- C11 standard (`-std=c11`)
- Warning flags: `-Wall -Wextra -Wpedantic` (Meson `warning_level=3`)
- Use Address Sanitizer during development (`-Db_sanitize=address,undefined`)
- Format with `clang-format` (project `.clang-format` file is source of truth)
- Include guards in every header (`#ifndef HEADER_H` / `#define HEADER_H` / `#endif`)
- `static` for file-private functions and variables
- Meaningful names, no Hungarian notation
- Comments explain *why*, not *what*

### Patterns

- Each module exposes an `_init()` and `_cleanup()` function pair
- Modules do not call each other's init/cleanup — `main.c` orchestrates lifecycle
- Error handling: check return values, print to stderr with `fprintf(stderr, ...)`, return error codes or `-1`. No exceptions (C doesn't have them), no `exit()` from modules — only `main.c` decides when to exit.
- Memory: track all allocations, free everything on cleanup. Valgrind should report zero leaks on clean shutdown.
- No global mutable state if avoidable — pass context structs between functions. If a module needs persistent state, use a file-scoped `static` struct.

### What NOT to do

- Don't use C++ features or compile as C++
- Don't add dependencies that aren't necessary — this project should build with minimal system packages
- Don't abstract prematurely — write the concrete thing first, generalize only when there's a real second use case
- Don't use `malloc` where stack allocation or a fixed buffer works fine
- Don't write "clever" code — straightforward and readable wins

## How to Help

The developer wants to learn C by doing this project. When helping:

- **Explain, don't just give code.** If asked a question, explain the concept and let Austin write the implementation. Providing snippets or examples is fine, but don't write entire files or modules unless explicitly asked.
- **Point to references.** When relevant, point to specific source files in projects like `swaybg`, `wlr-randr`, or `wlroots` examples that demonstrate the concept in question. Real-world code is more valuable than contrived examples.
- **Catch C footguns.** If you see potential memory leaks, buffer overflows, use-after-free, missing null checks, or undefined behavior — flag them clearly. These are the hardest bugs to learn from because they don't always crash immediately.
- **Stay idiomatic.** Suggest patterns that experienced C developers would use, not patterns translated from Go or other languages. If Austin's approach is "Go-brained" in a way that doesn't fit C, say so.
- **Wayland-specific help is expected.** Wayland protocol interaction is complex and poorly documented outside of reading source code. Detailed help with protocol binding, surface configuration, buffer management, and event handling is appropriate — this isn't "doing the project" for him, it's navigating an ecosystem with a steep learning curve.
- **Don't generate boilerplate unprompted.** If asked "how do I do X?", explain the approach. If asked "can you write the boilerplate for X?", then write it.
- **Debug collaboratively.** When Austin hits a bug, ask what he's seeing (error messages, WAYLAND_DEBUG output, valgrind output) before suggesting fixes. Help him build debugging instincts.

## Build & Run

```bash
# setup
meson setup build
ln -s build/compile_commands.json .

# build
ninja -C build

# run
./build/wl-ronin

# run with wayland debug output
WAYLAND_DEBUG=1 ./build/wl-ronin

# run under valgrind
valgrind ./build/wl-ronin

# format code
ninja -C build clang-format
```

## Key References

- `swaybg` source — minimal wlr-layer-shell client, best structural reference for this project
- `wlr-protocols` (GitHub) — protocol XML files including layer-shell
- wayland-book.com — the Wayland protocol explained from fundamentals
- `wlroots` examples directory — small example Wayland clients
- Wayland protocol XML files in `/usr/share/wayland-protocols/` and wherever wlr-protocols installs to on the system
