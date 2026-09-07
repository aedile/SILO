# Platform-independent Missile Command emulation core (shared with the host harness in ../../host)
set(CORE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../../core)
idf_component_register(
    SRCS "${CORE_DIR}/missile.c" "${CORE_DIR}/pokey.c"
    INCLUDE_DIRS "${CORE_DIR}"
    LDFRAGMENTS "linker.lf"
)
target_compile_options(${COMPONENT_LIB} PRIVATE -O3 -fjump-tables -ftree-switch-conversion)
