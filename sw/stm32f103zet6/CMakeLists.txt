# ---------------------------------------------------------------------------- #
# ---------------------------------------------------------------------------- #
cmake_minimum_required(VERSION 3.11)
# ---------------------------------------------------------------------------- #

# ---------------------------------------------------------------------------- #
# ---------------------------------------------------------------------------- #
include(FetchContent)
# ---------------------------------------------------------------------------- #

# ---------------------------------------------------------------------------- #
# Resolve required names, URLs, paths and versions.                            #
# ---------------------------------------------------------------------------- #
set(TOOLCHAIN_URL https://github.com/voltirex/gcc-arm-none-eabi-7-2017-q4-major-linux)

set(CMAKE_UTILS_GIT_REPOSITORY https://github.com/voltirex/cmake-utils.git)
set(CMAKE_UTILS_GIT_TAG        master)

set(STM32CUBE_FW_F1_V16X_GIT_REPOSITORY https://github.com/voltirex/stm32cube-fw-f1-v16x.git)
set(STM32CUBE_FW_F1_V16X_GIT_TAG        master)

set(STM32CUBE_FW             "stm32cube-fw-f1-v16x")
set(MAIN_TARGET_NAME         vx-peripheral-stm32f103zet6)
set(LINKER_FILE              ${CMAKE_SOURCE_DIR}/src/STM32F103ZETx_FLASH.ld)
set(STARTUP_FILE             ${CMAKE_SOURCE_DIR}/src/startup_stm32f103xe.s)
# ---------------------------------------------------------------------------- #

# ---------------------------------------------------------------------------- #
# ---------------------------------------------------------------------------- #
FetchContent_Declare(
    cmake-utils
        GIT_REPOSITORY ${CMAKE_UTILS_GIT_REPOSITORY}
        GIT_TAG        ${CMAKE_UTILS_GIT_TAG}
)

FetchContent_GetProperties(cmake-utils)
if(NOT cmake-utils_POPULATED)
    FetchContent_Populate(cmake-utils)
    include(${cmake-utils_SOURCE_DIR}/cmake-utils.cmake)
endif()
# ---------------------------------------------------------------------------- #

# ---------------------------------------------------------------------------- #
# Prepare toolchain.                                                           #
# ---------------------------------------------------------------------------- #
set(TOOLCHAINS_DIR ${CMAKE_SOURCE_DIR}/toolchains)
set(TOOLCHAIN_DIR  ${TOOLCHAINS_DIR}/gcc-arm-none-eabi-7-2017-q4-major-linux/gcc-arm-none-eabi-7-2017-q4-major-linux)

if (NOT EXISTS ${TOOLCHAINS_DIR})
    file(MAKE_DIRECTORY ${TOOLCHAINS_DIR})
    execute_process(COMMAND git clone ${TOOLCHAIN_URL} WORKING_DIRECTORY ${TOOLCHAINS_DIR})
endif()
# ---------------------------------------------------------------------------- #

# ---------------------------------------------------------------------------- #
# The toolchain file contains information about the crosscompiler setup -      #
# crosscompiler and library locations. An expanded toolchain file contains all #
# paths from the toolchain file, with expanded paths inside. This is required  #
# because the expanded toolchain file is passed down to subprojects and        #
# external projects, which don't reside at the top level of the project.       #
# ---------------------------------------------------------------------------- #
set(CMAKE_TOOLCHAIN_FILE          ${cmake-utils_SOURCE_DIR}/toolchains/gcc-arm-none-eabi-7-2017-q4-major_toolchainfile.cmake)
set(CMAKE_TOOLCHAIN_FILE_EXPANDED ${CMAKE_BINARY_DIR}/gcc-arm-none-eabi-7-2017-q4-major_toolchainfile_expanded.cmake)
# ---------------------------------------------------------------------------- #

# ---------------------------------------------------------------------------- #
# ---------------------------------------------------------------------------- #
FetchContent_Declare(
    ${STM32CUBE_FW}
        GIT_REPOSITORY ${STM32CUBE_FW_F1_V16X_GIT_REPOSITORY}
        GIT_TAG        ${STM32CUBE_FW_F1_V16X_GIT_TAG}
)

FetchContent_GetProperties(${STM32CUBE_FW})
if(NOT ${STM32CUBE_FW}_POPULATED)
    FetchContent_Populate(${STM32CUBE_FW})
endif()
# ---------------------------------------------------------------------------- #

# ---------------------------------------------------------------------------- #
# ---------------------------------------------------------------------------- #
project(
    vx-peripheral-stm32f103zet6
        LANGUAGES
            C ASM
        VERSION
            0.1
)
# ---------------------------------------------------------------------------- #

# ---------------------------------------------------------------------------- #
# Create expanded toolchain file.                                              #
# ---------------------------------------------------------------------------- #
configure_file(${CMAKE_TOOLCHAIN_FILE} ${CMAKE_TOOLCHAIN_FILE_EXPANDED})
# ---------------------------------------------------------------------------- #

# ---------------------------------------------------------------------------- #
# ---------------------------------------------------------------------------- #
set(CMAKE_C_STANDARD 99)

set(
    CMAKE_C_FLAGS
        "${CMAKE_C_FLAGS} \
            -O0 \
            -mcpu=cortex-m3 \
            -mthumb \
            -Wall \
            -Wextra \
            -fdata-sections \
            -ffunction-sections \
            -g \
            -gdwarf-2"
)

add_definitions(-DUSE_FULL_LL_DRIVER)
add_definitions(-DUSE_HAL_DRIVER)
add_definitions(-DSTM32F103xE)
add_definitions(-DARM_MATH_CM3)
# ---------------------------------------------------------------------------- #

# ---------------------------------------------------------------------------- #
# ---------------------------------------------------------------------------- #
stm32cube_fw_cmsis_prepare(
    ${STM32CUBE_FW}
    ${${STM32CUBE_FW}_SOURCE_DIR}
    cmsis_SOURCE_FILES
    cmsis_INCLUDE_FILES
    cmsis_INCLUDE_DIRECTORIES
)

stm32cube_fw_stm32xxxx_hal_prepare(
    ${STM32CUBE_FW}
    ${${STM32CUBE_FW}_SOURCE_DIR}
    stm32xxxx_hal_SOURCE_FILES
    stm32xxxx_hal_INCLUDE_FILES
    stm32xxxx_hal_INCLUDE_DIRECTORIES
)

stm32cube_fw_usb_core_and_cdc_prepare(
    ${STM32CUBE_FW}
    ${${STM32CUBE_FW}_SOURCE_DIR}
    stm32xxxx_usb_core_and_cdc_SOURCE_FILES
    stm32xxxx_usb_core_and_cdc_INCLUDE_FILES
    stm32xxxx_usb_core_and_cdc_INCLUDE_DIRECTORIES
)

stm32cube_fw_freertos_prepare(
    ${STM32CUBE_FW}
    ${${STM32CUBE_FW}_SOURCE_DIR}
    freertos_SOURCE_FILES
    freertos_INCLUDE_FILES
    freertos_INCLUDE_DIRECTORIES
)
# ---------------------------------------------------------------------------- #

# ---------------------------------------------------------------------------- #
# ---------------------------------------------------------------------------- #
set(
    project_SOURCE_FILES
        ${STARTUP_FILE}
        src/freertos.c
        src/main.c
        src/stm32f1xx_hal_msp.c
        src/stm32f1xx_hal_timebase_TIM.c
        src/stm32f1xx_it.c
        src/system_stm32f1xx.c
        src/usbd_cdc_if.c
        src/usbd_conf.c
        src/usbd_desc.c
        src/usb_device.c
)

set(
    project_INCLUDE_FILES
        src/FreeRTOSConfig.h
        src/main.h
        src/stm32_assert.h
        src/stm32f1xx_hal_conf.h
        src/stm32f1xx_it.h
        src/usbd_cdc_if.h
        src/usbd_conf.h
        src/usbd_desc.h
        src/usb_device.h
)

set(
    project_INCLUDE_DIRECTORIES
        src
)
# ---------------------------------------------------------------------------- #

# ---------------------------------------------------------------------------- #
# ---------------------------------------------------------------------------- #
add_custom_target(
    ${MAIN_TARGET_NAME}
        ALL
)

add_dependencies(
    ${MAIN_TARGET_NAME}
        ${MAIN_TARGET_NAME}.elf
        ${MAIN_TARGET_NAME}.hex
        ${MAIN_TARGET_NAME}.bin
)
# ---------------------------------------------------------------------------- #

# ---------------------------------------------------------------------------- #
# ---------------------------------------------------------------------------- #
add_executable(
    ${MAIN_TARGET_NAME}.elf
        ${cmsis_SOURCE_FILES}
        ${cmsis_INCLUDE_FILES}
        ${stm32xxxx_hal_SOURCE_FILES}
        ${stm32xxxx_hal_INCLUDE_FILES}
        ${stm32xxxx_usb_core_and_cdc_SOURCE_FILES}
        ${stm32xxxx_usb_core_and_cdc_INCLUDE_FILES}
        ${freertos_SOURCE_FILES}
        ${freertos_INCLUDE_FILES}
        ${project_SOURCE_FILES}
        ${project_INCLUDE_FILES}
)

target_include_directories(
    ${MAIN_TARGET_NAME}.elf
        PUBLIC
            ${cmsis_INCLUDE_DIRECTORIES}
            ${stm32xxxx_hal_INCLUDE_DIRECTORIES}
            ${stm32xxxx_usb_core_and_cdc_INCLUDE_DIRECTORIES}
            ${freertos_INCLUDE_DIRECTORIES}
            ${project_INCLUDE_DIRECTORIES}
)

set_target_properties(
    ${MAIN_TARGET_NAME}.elf
        PROPERTIES
            LINK_FLAGS "-T ${LINKER_FILE} -specs=nano.specs -Wl,--gc-sections"
)

target_link_libraries(
    ${MAIN_TARGET_NAME}.elf
        c
        m
        nosys
)
# ---------------------------------------------------------------------------- #

# ---------------------------------------------------------------------------- #
# ---------------------------------------------------------------------------- #
add_custom_target(
    ${MAIN_TARGET_NAME}.hex
        ${CMAKE_OBJCOPY} -O ihex ${CMAKE_BINARY_DIR}/${MAIN_TARGET_NAME}.elf ${CMAKE_BINARY_DIR}/${MAIN_TARGET_NAME}.hex
)

add_dependencies(
    ${MAIN_TARGET_NAME}.hex
        ${MAIN_TARGET_NAME}.elf
)

add_custom_target(
    ${MAIN_TARGET_NAME}.bin
        ${CMAKE_OBJCOPY} -O binary -S ${CMAKE_BINARY_DIR}/${MAIN_TARGET_NAME}.elf ${CMAKE_BINARY_DIR}/${MAIN_TARGET_NAME}.bin
)

add_dependencies(
    ${MAIN_TARGET_NAME}.bin
        ${MAIN_TARGET_NAME}.elf
)
# ---------------------------------------------------------------------------- #
