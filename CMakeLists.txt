cmake_minimum_required(VERSION 3.13)

set(PICO_BOARD pico2_w CACHE STRING "Board type")
set(PICO_PLATFORM rp2350 CACHE STRING "the Cortex-M33s")

# The player drives the RP2350's PWM and DMA blocks through `sh.sysl.rp2350`'s register map, which is
# the Arm and RISC-V cores' common ground; only the Arm build is wired and verified here.
set(SYSL_TARGET thumb-freestanding-softfp)

include(pico_sdk_import.cmake)

project(musicbox_pico C CXX ASM)

pico_sdk_init()

set(SYSL_ARCHIVE ${CMAKE_CURRENT_BINARY_DIR}/libplayer.a)

# **`sysl build-c` is a program CMake knows nothing about**, so what it can see has to be handed to it:
# newlib's headers from the toolchain, the SDK's from the executable's resolved include list, and the
# macros `pico/cyw43_arch.h` wants. `pico2` carries C that includes them. solder-pico2's
# CMakeLists.txt carries the whole story of each line.
execute_process(
    COMMAND ${CMAKE_C_COMPILER} -print-sysroot
    OUTPUT_VARIABLE ARM_SYSROOT
    OUTPUT_STRIP_TRAILING_WHITESPACE)

set(SYSL_C_INCLUDES
    ${ARM_SYSROOT}/include
    $<TARGET_PROPERTY:musicbox_pico,INCLUDE_DIRECTORIES>)

set(SYSL_C_DEFINES $<TARGET_PROPERTY:musicbox_pico,COMPILE_DEFINITIONS>)

add_custom_command(
    OUTPUT ${SYSL_ARCHIVE} ${SYSL_ARCHIVE}.h
    COMMAND sysl build-c ${CMAKE_CURRENT_SOURCE_DIR}/player --target ${SYSL_TARGET}
            --include-path pico_sdk=${PICO_SDK_PATH}
            "$<$<BOOL:${SYSL_C_INCLUDES}>:--include-path;$<JOIN:${SYSL_C_INCLUDES},;--include-path;>>"
            "$<$<BOOL:${SYSL_C_DEFINES}>:-D;$<JOIN:${SYSL_C_DEFINES},;-D;>>"
            -o ${SYSL_ARCHIVE}
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/player/main.sysl
            ${CMAKE_CURRENT_SOURCE_DIR}/player/sysl.hocon
    COMMENT "Compiling the player and musicbox for ${SYSL_TARGET}"
    COMMAND_EXPAND_LISTS
    VERBATIM)

add_custom_target(sysl_archive DEPENDS ${SYSL_ARCHIVE})

# No C in this project: sysl exports `main` and the SDK's `crt0` branches into it.
add_executable(musicbox_pico ${SYSL_ARCHIVE})

set_target_properties(musicbox_pico PROPERTIES LINKER_LANGUAGE C)

add_dependencies(musicbox_pico sysl_archive)

# `pico2`'s `init` starts the wireless chip, hence the cyw43 library; `m` is for `pow`, which the
# notation parser reaches for a note's frequency.
target_link_libraries(musicbox_pico pico_stdlib pico_cyw43_arch_none ${SYSL_ARCHIVE} m)

# **A 16 KB main stack, carved from the top of RAM.** `boot` holds the synthesizer's construction
# result, a `Result[Synth, _]` of about 5.6 KB, in its own frame before boxing it, and the SDK's
# default core-0 stack is SCRATCH_Y's 4 KB — too small. Raising `PICO_STACK_SIZE` alone does not help:
# the stack lives in SCRATCH_Y, and the link fails once the reservation outgrows the region. So RAM
# gives up its top 16 KB (0x2007C000-0x20080000) and SCRATCH_Y moves there at that size; the
# reservation is then sized to the whole region, so the linker checks it fits. Core 1 is never
# started, so SCRATCH_X and the old SCRATCH_Y (0x20080000-0x20082000) go unused. README.md has the
# map and the measured frames.
pico_set_linker_script_var(musicbox_pico RAM_LENGTH 0x7C000)
pico_set_linker_script_var(musicbox_pico SCRATCH_Y_ORIGIN 0x2007C000)
pico_set_linker_script_var(musicbox_pico SCRATCH_Y_LENGTH 0x4000)
target_compile_definitions(musicbox_pico PRIVATE PICO_STACK_SIZE=0x4000)

pico_enable_stdio_usb(musicbox_pico 1)
pico_enable_stdio_uart(musicbox_pico 0)

pico_add_extra_outputs(musicbox_pico)
