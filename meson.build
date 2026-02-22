project('wl-ronin', 'c',
    version: '0.1.0',
    default_options: [
        'c_std=c11',
        'warning_level=3',
        'b_sanitize=address,undefined',
    ],
)

wayland_client = dependency('wayland-client')

subdir('protocol')

src = files(
    'src/main.c',
    'src/wayland.c',
    'src/render.c',
    'src/pet.c',
    'src/config.c',
    'src/hyprland.c',
)

executable('wl-ronin',
    sources: src,
    dependencies: [wayland_client, wl_proto_dep],
    include_directories: include_directories('src', 'vendor'),
)

clang_format = find_program('clang-format', required: false)
if clang_format.found()
    run_target('clang-format',
        command: [clang_format, '-i'] + src,
    )
endif
