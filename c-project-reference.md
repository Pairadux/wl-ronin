# C Development Reference

## Dependencies & Package Management

C has no `go.mod`, lockfile, or project-local dependency system. Dependencies are handled through:

- **System libraries** — installed via `pacman`, found at build time via `pkg-config`. Headers land in `/usr/include/`, shared libs in `/usr/lib/`. This is the standard, idiomatic approach for the entire Linux ecosystem.
- **Vendored single-header libraries** — drop `.h` (and sometimes `.c`) files directly into your project tree under `third_party/`. No install needed. Examples: `stb_image.h`, `inih`.
- **Version pinning** — done in your `meson.build` with `dependency('wayland-client', version : '>=1.20.0')`. Not a lockfile, but Meson will refuse to build if the version is too old.

For reproducible environments across machines, Nix (`flake.nix`) is the closest thing to `go.sum`. Optional but valuable for DevOps resume.

### Linking

- **Dynamic (default):** binary references shared libs at runtime. User needs libs installed. Check with `ldd ./build/binary`.
- **Static:** library code baked into binary. Self-contained, zero runtime deps (like Go binaries). Enable with `dependency('x', static : true)` in Meson. Bigger binary.

For Hyprland-specific tools, dynamic linking is fine — users already have the Wayland libs.

---

## Project Structure

```
projectname/
├── meson.build               # root build file
├── meson.options              # user-configurable build options
├── README.md
├── LICENSE
├── .clang-format              # auto-formatter config
│
├── protocol/                  # wayland-scanner generated code
│   └── meson.build
│
├── src/
│   ├── main.c                 # entry point, arg parsing, event loop
│   ├── module.c               # implementation
│   └── module.h               # public declarations
│
├── assets/                    # sprites, icons, etc.
│
└── third_party/               # vendored single-header libs
    └── stb_image.h
```

### How C Modules Work

Each `.c` file compiles independently into an object file. Headers (`.h`) declare the public interface. The linker combines everything.

```c
// thing.h
#ifndef THING_H
#define THING_H       // include guard — put in every .h file

int thing_init(void);
void thing_cleanup(void);

#endif
```

```c
// thing.c
#include "thing.h"

int thing_init(void) { /* ... */ }
void thing_cleanup(void) { /* ... */ }
```

```c
// main.c
#include "thing.h"

int main(void) {
    thing_init();
    thing_cleanup();
    return 0;
}
```

---

## Build System: Meson + Ninja

Install: `pacman -S meson ninja`

### Daily Workflow

```bash
# first time setup
meson setup build

# compile
ninja -C build

# run
./build/projectname

# symlink compile_commands.json for clangd (do once)
ln -s build/compile_commands.json .

# reconfigure after changing meson.build
meson setup build --reconfigure

# clean
ninja -C build clean
```

### Minimal meson.build

```meson
project('projectname', 'c',
  version : '0.1.0',
  default_options : [
    'c_std=c11',
    'warning_level=3',
  ],
)

wayland_client = dependency('wayland-client')

src = files(
  'src/main.c',
)

executable('projectname',
  sources : src,
  dependencies : [wayland_client],
  include_directories : include_directories('src', 'third_party'),
  install : true,
)
```

### Enabling Sanitizers (use during development)

```bash
meson setup build -Db_sanitize=address,undefined
```

---

## CLI Argument Parsing

Use `getopt_long` from `<getopt.h>` (part of glibc, no install needed).

```c
#include <getopt.h>

static struct option long_options[] = {
    {"config",  required_argument, 0, 'c'},
    {"speed",   required_argument, 0, 's'},
    {"help",    no_argument,       0, 'h'},
    {0, 0, 0, 0}
};

int opt;
while ((opt = getopt_long(argc, argv, "c:s:h", long_options, NULL)) != -1) {
    switch (opt) {
        case 'c': config_path = optarg; break;
        case 's': speed = atoi(optarg); break;
        case 'h': print_usage(); exit(0);
    }
}
```

## Config Files

Use `inih` (vendor `ini.c` + `ini.h` into `third_party/`). Reads INI format. Use `$XDG_CONFIG_HOME/projectname/config.ini` as the default path.

## Environment Variables

`getenv("VAR_NAME")` from `<stdlib.h>`. No library needed.

---

## Recommended Tools

### Essential

| Tool | Purpose | Install |
|------|---------|---------|
| `meson` + `ninja` | Build system | `pacman -S meson ninja` |
| `clangd` | LSP for Neovim (needs `compile_commands.json`) | `pacman -S clang` |
| `clang-format` | Auto-formatter | comes with `clang` |
| `valgrind` | Memory error detection | `pacman -S valgrind` |
| `gdb` | Debugger (use `gdb -tui` for curses UI) | `pacman -S gdb` |
| `pkg-config` | Finds library compiler flags | `pacman -S pkgconf` |

### Wayland-Specific

| Tool | Purpose |
|------|---------|
| `wayland-scanner` | Generates C code from protocol XML (comes with `wayland` package) |
| `wayland-info` | Lists protocols your compositor supports |
| `WAYLAND_DEBUG=1` | Env var — dumps all Wayland protocol messages to stderr |
| `strace` | Trace system calls, useful for socket/fd debugging |

### Compile Flags for Development

```
-Wall -Wextra -Wpedantic -fsanitize=address,undefined
```

Meson handles `-Wall -Wextra` via `warning_level=3`. Add sanitizers with `-Db_sanitize=address,undefined`.

---

## Testing

No built-in `go test` equivalent. Options:

- **Assert-based:** write your own with `assert()` from `<assert.h>`.
- **greatest.h** — single-header test framework, vendor it.
- **Meson test runner** — add `test('name', executable(...))` in `meson.build`.

For Wayland GUI projects, most testing is manual (run it, see if it works). Unit test state machine logic and parsing.

---

## First C Program Sanity Check

```c
#include <stdio.h>
#include <wayland-client.h>

int main(void) {
    struct wl_display *display = wl_display_connect(NULL);
    if (!display) {
        fprintf(stderr, "Failed to connect to Wayland display\n");
        return 1;
    }
    printf("Connected to Wayland!\n");
    wl_display_disconnect(display);
    return 0;
}
```

If this compiles and prints the message, your toolchain works.

---

---

# wl-ronin (Desktop Pet)

## Overview

A transparent Wayland overlay with an animated sprite that wanders the screen and optionally reacts to Hyprland events via IPC.

## Dependencies

### System (pacman)

```bash
pacman -S wayland wayland-protocols wlr-protocols meson ninja
```

- `libwayland-client` — base Wayland client library (`<wayland-client.h>`)
- `wlr-layer-shell-unstable-v1` protocol — creates overlay surfaces (from `wlr-protocols`)
- `wayland-scanner` — codegen tool, comes with `wayland` package

### Optional System

- `cairo` (`pacman -S cairo`) — nicer 2D drawing, PNG loading. Alternative to raw SHM pixel buffers.

### Vendored (third_party/)

- `stb_image.h` — single-header PNG/JPEG loader. `curl -o third_party/stb_image.h https://raw.githubusercontent.com/nothings/stb/master/stb_image.h`
- `inih` — INI config parser (ini.c + ini.h)

## Project Structure

```
wl-ronin/
├── meson.build                # root build file
├── meson.options              # build options (sanitizers, etc.)
├── .clang-format              # code formatter config
├── .gitignore                 # build/, *.o, compile_commands.json
├── README.md
├── LICENSE
│
├── protocol/
│   └── meson.build            # wayland-scanner codegen rules
│
├── src/
│   ├── main.c
│   ├── wayland.c
│   ├── wayland.h
│   ├── render.c
│   ├── render.h
│   ├── pet.c
│   ├── pet.h
│   ├── config.c
│   ├── config.h
│   ├── hyprland.c             # stretch goal
│   └── hyprland.h             # stretch goal
│
├── assets/
│   └── sprite.png             # pet sprite sheet
│
└── third_party/
    ├── stb_image.h            # vendored — PNG/JPEG loader
    └── inih/
        ├── ini.c              # vendored — INI parser
        └── ini.h
```

### File Responsibilities

**Build files (root):**

- `meson.build` — declares the project, finds dependencies via pkg-config, lists source files, defines the executable target, sets compiler flags. This is the only build file you edit regularly.
- `meson.options` — optional. Defines user-facing build toggles (e.g., `option('hyprland_ipc', type : 'boolean', value : true)`). Not needed for v1 but good practice.
- `.clang-format` — formatting rules. Run `clang-format -dump-config -style=llvm > .clang-format` to generate a sane default, then tweak to taste.
- `.gitignore` — at minimum: `build/`, `compile_commands.json` (it's a symlink to `build/`).

**Protocol codegen:**

- `protocol/meson.build` — contains the `wayland-scanner` invocation rules. Takes the `wlr-layer-shell-unstable-v1.xml` protocol file (installed system-wide by `wlr-protocols`), runs `wayland-scanner` to generate two files at build time: a `.h` (protocol function declarations) and a `.c` (glue code). You never edit the generated files. Copy this pattern from `swaybg`'s protocol meson.build — it's ~15 lines.

**Source files:**

- `main.c` — entry point. Parses CLI args with `getopt_long`, loads config, calls `wayland_init()`, enters the main event loop (`poll()` on the Wayland display fd + a timer fd), calls `pet_update()` and `render_frame()` each tick, and calls cleanup functions on exit. No header needed — nothing calls into main.

- `wayland.h` / `wayland.c` — all Wayland protocol interaction. Connects to the display, binds compositor globals (compositor, shm, layer-shell), creates the layer-shell surface, handles surface configure events, manages the SHM buffer pool (allocating shared memory, creating `wl_buffer` objects). Exposes functions like `wayland_init()`, `wayland_get_buffer()`, `wayland_commit_frame()`, `wayland_cleanup()`. This is the most complex file and where you'll spend the most time early on.

- `render.h` / `render.c` — drawing logic. Loads sprite sheet from PNG via `stb_image`, manages sprite frame data (pixel arrays, dimensions, frame count). Exposes functions like `render_init(sprite_path)`, `render_clear(buffer)` (fills buffer with transparent pixels), `render_draw_sprite(buffer, x, y, frame)` (blits a sprite frame into the SHM buffer at the given position), `render_cleanup()`. Keeps rendering concerns separate from Wayland protocol concerns.

- `pet.h` / `pet.c` — the pet's brain. Defines the state enum (`PET_IDLE`, `PET_WALK_LEFT`, `PET_WALK_RIGHT`, `PET_SIT`, etc.), holds current state, position, animation frame counter, and direction. Exposes `pet_init()`, `pet_update(delta_time)` (advances position, handles state transitions, picks random behaviors), `pet_get_x()`, `pet_get_y()`, `pet_get_frame()`. Pure logic — no Wayland or rendering code in here. This is the most fun file and the easiest to unit test.

- `config.h` / `config.c` — config file loading and CLI arg parsing. Defines a config struct with fields like `sprite_path`, `speed`, `monitor`, `cache_dir`. Exposes `config_load(argc, argv)` which reads the INI file from `$XDG_CONFIG_HOME/wl-ronin/config.ini`, then applies CLI overrides on top. Returns a populated config struct that other modules read from.

- `hyprland.h` / `hyprland.c` — (stretch goal) connects to Hyprland's IPC socket, parses event lines, exposes an fd for the main loop to `poll()` on, and provides parsed events to `pet.c` so the pet can react. Exposes `hyprland_init()`, `hyprland_get_fd()`, `hyprland_read_event()`, `hyprland_cleanup()`. Don't write this until everything else works.

**Third party:**

- `stb_image.h` — you include this in exactly one `.c` file (probably `render.c`) with `#define STB_IMAGE_IMPLEMENTATION` before the include. That define tells it to include the actual function bodies, not just declarations. Every other file that needs stb_image types just includes it without the define.
- `inih/ini.c` + `ini.h` — compiled as part of your project (listed in `meson.build` sources). Used by `config.c`.

**Assets:**

- `sprite.png` — your sprite sheet. Lay out animation frames in a horizontal strip (e.g., 8 frames of 32x32 pixels = 256x32 image). `render.c` loads it and slices it into frames based on a configured frame width.

### What main.c looks like at a high level

```c
#include "config.h"
#include "wayland.h"
#include "render.h"
#include "pet.h"

int main(int argc, char *argv[]) {
    Config cfg = config_load(argc, argv);

    wayland_init();
    render_init(cfg.sprite_path);
    pet_init(cfg.speed);

    // main loop: poll wayland fd + timer fd
    while (running) {
        // handle wayland events
        // on timer tick:
        pet_update(delta);
        uint32_t *buffer = wayland_get_buffer();
        render_clear(buffer);
        render_draw_sprite(buffer, pet_get_x(), pet_get_y(), pet_get_frame());
        wayland_commit_frame();
    }

    pet_cleanup();
    render_cleanup();
    wayland_cleanup();
    return 0;
}
```

This is pseudocode — the real version will have `poll()`, error handling, and signal handling for clean shutdown. But this is the shape of it.

## How It Works with Hyprland

- Hyprland supports `wlr-layer-shell-v1` (verify with `wayland-info | grep layer_shell`)
- Create a layer surface on the `overlay` layer
- Set `exclusive_zone = -1` (don't push windows around)
- Set input region to empty (clicks pass through to windows below)
- Surface is a transparent ARGB8888 buffer — clear to transparent each frame, draw sprite at current position
- Hyprland composites the surface over everything

### Hyprland IPC (stretch goal)

- Socket at `$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket2.sock`
- Streams events (window open, workspace switch, etc.)
- Standard POSIX `connect()` + `poll()` alongside the Wayland display fd
- No library needed

## Config & Environment

- `$WAYLAND_DISPLAY` — read automatically by `libwayland-client`
- `$HYPRLAND_INSTANCE_SIGNATURE` — needed for IPC socket path
- `$XDG_CONFIG_HOME/wl-ronin/config.ini` — sprite path, speed, monitor, IPC toggle
- CLI flags: `--config`, `--monitor`, `--sprite`, `--speed`, `--help`

## Rendering Options

- **Raw SHM buffers** — zero deps, manual pixel writing. Most educational. Allocate shared memory, get raw ARGB pixel pointer, memcpy sprite data into position.
- **Cairo** — nicer API, built-in PNG loading, anti-aliasing. Good middle ground. `dependency('cairo')` in Meson.

Recommendation: start with raw SHM for learning, switch to Cairo if it gets painful.

## Roadmap

1. **Hello Wayland** — connect to display, print success
2. **Colored rectangle** — create `wlr-layer-shell` surface, draw a solid color. This is the hard milestone — once you have a visible surface, everything else is iteration.
3. **Transparency + passthrough** — full-screen transparent surface with empty input region
4. **Static sprite** — load PNG with `stb_image`, blit at fixed position
5. **Animation loop** — `timerfd_create` or `poll()` with timeout, move sprite each tick
6. **State machine** — idle, walk left, walk right, sit. Random transitions. Sprite animation frames.
7. **Hyprland IPC** — pet reacts to events (window opened, workspace switch, etc.)

## Key References

- `swaybg` source — small, readable `wlr-layer-shell` client, good structural reference
- `wlr-protocols` GitHub — protocol XML files
- `wayland-book.com` — the Wayland protocol explained

---

---

# Clipboard Manager (Clipboard Daemon)

## Overview

A daemon that watches Wayland clipboard events, stores text/image history, and integrates with rofi/wofi for selection.

## Dependencies

### System (pacman)

```bash
pacman -S wayland wayland-protocols wlr-protocols meson ninja
```

- `libwayland-client` — same as wl-ronin
- `wlr-data-control-unstable-v1` protocol — allows non-compositor programs to watch/set clipboard (from `wlr-protocols`)
- `wayland-scanner` — same as wl-ronin

### Optional System

- `libpng` / `libjpeg-turbo` — for image clipboard storage/thumbnailing. Skip for v1.

### Vendored

- `inih` — config parsing

## Project Structure

```
cliphist/
├── meson.build
├── protocol/
│   └── meson.build
├── src/
│   ├── main.c                 # entry point, arg parsing, daemon/CLI mode
│   ├── wayland.c / .h         # display connection, data-control binding
│   ├── clipboard.c / .h       # ring buffer, read fd, store entries
│   ├── cache.c / .h           # persistence to disk
│   └── config.c / .h
└── third_party/
    └── inih/
```

## Architecture

- Daemon mode: connects to Wayland, binds `zwlr_data_control_manager_v1`, listens for `selection` events. Each clipboard change delivers a file descriptor — read data from it, store in ring buffer. Persist to `$XDG_CACHE_HOME/cliphist/`.
- CLI mode (`cliphist list`): reads cache, dumps entries to stdout (one per line).
- Rofi integration: keybind runs `cliphist list | rofi -dmenu | cliphist paste`. No socket server needed for v1.

## Config & Environment

- `$XDG_CACHE_HOME` — cache location
- `$XDG_CONFIG_HOME/cliphist/config.ini` — max history size, store images toggle, ignore patterns
- CLI flags: `--max-items`, `--cache-dir`, `--no-images`

## Roadmap

1. Generate `wlr-data-control` protocol code with `wayland-scanner`
2. Connect to Wayland, bind `zwlr_data_control_manager_v1` global
3. Create `data_control_device`, listen for `selection` events
4. Read clipboard content from received fd, print to stdout (working clipboard watcher)
5. Ring buffer storage + disk cache
6. `list` and `paste` subcommands
7. Image support, ignore patterns

## Key Protocols

- `zwlr_data_control_manager_v1` — get a data control device
- `zwlr_data_control_device_v1` — `selection` event fires on clipboard change, gives you a `data_offer`
- `zwlr_data_control_offer_v1` — call `receive` with a MIME type and fd, read clipboard data from the fd
