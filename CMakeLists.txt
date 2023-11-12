# Set minimum required version of CMake
cmake_minimum_required(VERSION 3.12)

# Include build functions from Pico SDK
include($ENV{PICO_SDK_PATH}/external/pico_sdk_import.cmake)

# Set name of project (as PROJECT_NAME) and C/C   standards
project(Rotary_Dimmer C CXX ASM)
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)

# Creates a pico-sdk subdirectory in our project for the libraries
pico_sdk_init()

add_compile_options(-Wall
        -Wno-format          # int != int32_t as far as the compiler is concerned because gcc has int32_t as long int
        -Wno-unused-function # we have some for the docs that aren't called
        -Wno-maybe-uninitialized
)

# Tell CMake where to find the executable source file
add_executable(Rotary_Dimmer
        main.c
)

# Create map/bin/hex/uf2 files
pico_add_extra_outputs(Rotary_Dimmer)

# Link to pico_stdlib (gpio, time, etc. functions)
target_link_libraries(Rotary_Dimmer
        pico_stdlib
        hardware_pwm
)

# Enable usb output, disable uart output
pico_enable_stdio_usb(Rotary_Dimmer 1)
pico_enable_stdio_uart(Rotary_Dimmer 0)

add_custom_command(TARGET Rotary_Dimmer
        POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/Rotary_Dimmer.uf2 E:)
# change E: to the drive that computer assigned to pi pico