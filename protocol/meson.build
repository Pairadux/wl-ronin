wayland_scanner = find_program('wayland-scanner')

wayland_scanner_code = generator(
    wayland_scanner,
    output: '@BASENAME@-protocol.c',
    arguments: ['private-code', '@INPUT@', '@OUTPUT@'],
)

wayland_scanner_client = generator(
    wayland_scanner,
    output: '@BASENAME@-client-protocol.h',
    arguments: ['client-header', '@INPUT@', '@OUTPUT@'],
)

layer_shell_xml = files('wlr-layer-shell-unstable-v1.xml')

proto_sources = wayland_scanner_code.process(layer_shell_xml)
proto_headers = wayland_scanner_client.process(layer_shell_xml)

lib_protos = static_library('protos',
    sources: [proto_sources, proto_headers],
    dependencies: dependency('wayland-client'),
)

# wl_proto_dep is used by the root meson.build after subdir('protocol')
wl_proto_dep = declare_dependency(
    link_with: lib_protos,
    sources: proto_headers,
)
