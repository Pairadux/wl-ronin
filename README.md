# wl-ronin

A Wayland desktop pet for Hyprland. An animated ronin wanders your screen on a transparent overlay, idling, walking, and reacting to compositor events. Clicks pass through so it stays out of your way.

The name follows the `wl-` Wayland convention. A ronin is a masterless wandering samurai.

## Requirements

- Hyprland (or any wlroots compositor with `wlr-layer-shell-v1` support)
- `libwayland-client`

## Build & Run

```sh
meson setup build
ninja -C build
./build/wl-ronin
```

## License

MIT
