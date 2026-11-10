file(GLOB SRC_FILES "src/*.c")

idf_component_register(
    SRCS ${SRC_FILES}
    INCLUDE_DIRS "include"
    PRIV_INCLUDE_DIRS "src"
)

# JARVIS PATCH 20261109 (fix 7): enable the component's PSRAM-first alloc paths.
target_compile_definitions(${COMPONENT_LIB} PRIVATE USE_ESP32)
