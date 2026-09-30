# Generated Cmake Pico project file

cmake_minimum_required(VERSION 3.13)

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Initialise pico_sdk from installed location
# (note this can come from environment, CMake cache etc)

# == DO NOT EDIT THE FOLLOWING LINES for the Raspberry Pi Pico VS Code Extension to work ==
if(WIN32)
    set(USERHOME $ENV{USERPROFILE})
else()
    set(USERHOME $ENV{HOME})
endif()

set(sdkVersion 2.3.1)
set(toolchainVersion 15_2_Rel1)
set(picotoolVersion 2.3.1)

if(NOT DEFINED PICO_VS_CODE)
    set(PICO_VS_CODE "${USERHOME}/.pico-sdk/cmake/pico-vscode.cmake")
endif()

include(${PICO_VS_CODE})

# ====================================================================================
set(PICO_BOARD pico_w CACHE STRING "Board type")

# Pull in Raspberry Pi Pico SDK (must be before project)
include(pico_sdk_import.cmake)

project(logic-analyzer C CXX ASM)

# Initialise the Raspberry Pi Pico SDK
pico_sdk_init()

# Add executable. Default name is the project name, version 0.1

add_executable(logic-analyzer logic-analyzer.cpp)

pico_set_program_name(logic-analyzer "logic-analyzer")
pico_set_program_version(logic-analyzer "0.1")

# no_flash means the target is to run from RAM
pico_set_binary_type(logic-analyzer no_flash)

# Generate PIO header
pico_generate_pio_header(logic-analyzer ${CMAKE_CURRENT_LIST_DIR}/capture.pio)

# Modify the below lines to enable/disable output over UART/USB
pico_enable_stdio_uart(logic-analyzer 1)
pico_enable_stdio_usb(logic-analyzer 0)

# Add the standard library to the build
target_link_libraries(
    logic-analyzer
    pico_stdlib
)

# Add the standard include files to the build
target_include_directories(
    logic-analyzer PRIVATE
    ${CMAKE_CURRENT_LIST_DIR}
)

# Add any user requested libraries
target_link_libraries(
    logic-analyzer
    hardware_dma
    hardware_pio
    hardware_clocks
)

pico_add_extra_outputs(logic-analyzer)
