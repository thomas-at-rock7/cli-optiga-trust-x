cmake_minimum_required(VERSION "3.13.4")
project("cli-optiga-trust-x")

option(BUILD_FOR_RPI "BUILD_FOR_RPI" OFF) # Default
option(BUILD_FOR_ULTRA96 "BUILD_FOR_ULTRA96" OFF)
option(COLD_RESET "This is applicable if the host platform has GPIO option for RST and VDD." OFF) # Default
option(SOFT_RESET "This is applicable if the host platform doesn't have GPIO options for VDD and RST." OFF)
option(WARM_RESET "This is applicable if the host platform doesn't have GPIO option for VDD." OFF)
option(BUILD_LINUX_EXAMPLES "This option will build linux examples" OFF)


set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall")
add_definitions(-DENGINE_DYNAMIC_SUPPORT)

if (CMAKE_BUILD_TYPE)
    if (${CMAKE_BUILD_TYPE} STREQUAL "Release")
        message(STATUS "Build type Release")
    else()
        message(STATUS "Build type Debug")
        add_definitions(-DTRUSTX_ENGINE_DEBUG) # Enable TrustM debug messages
    endif()
else()
    message(STATUS "Build type not defined, defaulting to a Release build")
    set(CMAKE_BUILD_TYPE Release)
endif()

# Required libraries for linking
find_library(LIB_PTHREAD pthread)
find_library(LIB_RT rt)
find_library(LIB_SSL ssl)
find_library(LIB_CRYPTO crypto)

set (LIB_TRUSTX "trustx")
set (LIB_TRUSTX_ENGINE "trustx_engine")
set (TRUSTX "trustx_lib")
set (TRUSTX_HELPER "trustx_helper")
set (TRUSTX_UTIL "${TRUSTX}/optiga/util")
set (TRUSTX_CRYPT "${TRUSTX}/optiga/crypt")
set (TRUSTX_COMMS "${TRUSTX}/optiga/comms")
set (TRUSTX_COMMON "${TRUSTX}/optiga/common")
set (TRUSTX_CMD "${TRUSTX}/optiga/cmd")
set (PALDIR "${TRUSTX}/pal/linux")
set (ENGDIR "trustx_engine")
set (APPDIR "linux_example")
set (INCDIR $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${TRUSTX}/optiga/include>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${TRUSTX}/optiga/include/optiga>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${TRUSTX}/optiga/include/optiga/ifx_i2c>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${TRUSTX}/optiga/include/optiga/comms>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${TRUSTX}/optiga/include/optiga/common>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${TRUSTX}/optiga/include/optiga/cmd>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${TRUSTX}/optiga/include/optiga/pal>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${TRUSTX}/pal/linux>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/externals/mbedtls-2.12.0/include>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/trustx_helper/include>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/trustx_engine>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/examples/ecdsa_utils>
            $<INSTALL_INTERFACE:${TRUSTX}/optiga/include>
            $<INSTALL_INTERFACE:${TRUSTX}/optiga/include/optiga>
            $<INSTALL_INTERFACE:${TRUSTX}/optiga/include/optiga/ifx_i2c>
            $<INSTALL_INTERFACE:${TRUSTX}/optiga/include/optiga/comms>
            $<INSTALL_INTERFACE:${TRUSTX}/optiga/include/optiga/common>
            $<INSTALL_INTERFACE:${TRUSTX}/optiga/include/optiga/cmd>
            $<INSTALL_INTERFACE:${TRUSTX}/optiga/include/optiga/pal>
            $<INSTALL_INTERFACE:${TRUSTX}/externals/mbedtls-2.12.0/include>
            $<INSTALL_INTERFACE:${TRUSTX}/pal/linux>
            $<INSTALL_INTERFACE:trustx_helper/include>
            $<INSTALL_INTERFACE:trustx_engine>
            $<INSTALL_INTERFACE:/examples/ecdsa_utils>
)
set (ENGINE_INC $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${ENGDIR}>
            $<INSTALL_INTERFACE:${ENGDIR}>
)

# Set the reset type
if (COLD_RESET)
    message(STATUS "Cold reset defined")
    add_definitions(-DOPTIGA_COMMS_DEFAULT_RESET_TYPE=0)
elseif(SOFT_RESET)
    message(STATUS "Soft reset defined")
    add_definitions(-DOPTIGA_COMMS_DEFAULT_RESET_TYPE=1)
elseif(WARM_RESET)
    message(STATUS "Warm reset defined")
    add_definitions(-DOPTIGA_COMMS_DEFAULT_RESET_TYPE=2)
else()
    message(STATUS "Reset hasn't been defined a Cold reset will be used")
    add_definitions(-DOPTIGA_COMMS_DEFAULT_RESET_TYPE=0)
endif()

# Choose the appropiate i2c file depending on target, the default is rasberry pi
if (BUILD_FOR_RPI)
    message(STATUS "Using RPI i2c config")
    set (I2C_CONFIG "${PALDIR}/target/rpi3/pal_ifx_i2c_config.c")
elseif (BUILD_FOR_ULTRA96)
    message(STATUS "Using Ultra96 i2c config")
    set (I2C_CONFIG "${PALDIR}/target/ultra96/pal_ifx_i2c_config.c")
else()
    message(STATUS "Using Default I2C config which is RPI i2c config")
    set (I2C_CONFIG "${PALDIR}/target/rpi3/pal_ifx_i2c_config.c")
endif()

# Find all sources and build Trust M Library
file(GLOB_RECURSE HELPER_SOURCES LIST_DIRECTORIES true "${TRUSTX_HELPER}/*.c")
file(GLOB_RECURSE UTIL_SOURCES LIST_DIRECTORIES true "${TRUSTX_UTIL}/*.c")
file(GLOB_RECURSE CRYPT_SOURCES LIST_DIRECTORIES true "${TRUSTX_CRYPT}/*.c")
file(GLOB_RECURSE COMMS_SOURCES LIST_DIRECTORIES true "${TRUSTX_COMMS}/*.c")
file(GLOB_RECURSE COMMON_SOURCES LIST_DIRECTORIES true "${TRUSTX_COMMON}/*.c")
file(GLOB_RECURSE CMD_SOURCES LIST_DIRECTORIES true "${TRUSTX_CMD}/*.c")
add_library(${LIB_TRUSTX} SHARED
            ${HELPER_SOURCES}
            ${UTIL_SOURCES}
            ${CRYPT_SOURCES}
            ${COMMS_SOURCES}
            ${COMMON_SOURCES}
            ${CMD_SOURCES}
            ${PALDIR}/pal.c
            ${PALDIR}/pal_gpio.c
            ${PALDIR}/pal_i2c.c
            ${PALDIR}/pal_os_event.c
            ${PALDIR}/pal_os_lock.c
            ${PALDIR}/pal_os_timer.c
            ${I2C_CONFIG}
)
target_include_directories (${LIB_TRUSTX} PUBLIC "${INCDIR}")

# Find sources for Trust M library and build it
file(GLOB_RECURSE ENGINE_SOURCES LIST_DIRECTORIES true "${ENGDIR}/*.c")
add_library(${LIB_TRUSTX_ENGINE} SHARED ${ENGINE_SOURCES})
target_link_libraries(${LIB_TRUSTX_ENGINE} PUBLIC
                                ${LIB_TRUSTX}
                                ${LIB_PTHREAD}
                                ${LIB_RT}
                                ${LIB_SSL}
                                ${LIB_CRYPTO}
)
target_include_directories (${LIB_TRUSTX_ENGINE} PUBLIC "${ENGINE_INC}")

if (BUILD_LINUX_EXAMPLES)
    # Build all Trust M applications but don't install them
    file(GLOB_RECURSE APP_SOURCES LIST_DIRECTORIES true "${APPDIR}/*.c")
    foreach(appsrc IN ITEMS ${APP_SOURCES})
        get_filename_component(appbin ${appsrc} NAME_WE)
        add_executable(${appbin} ${appsrc})
        target_link_libraries(${appbin} PUBLIC
                                ${LIB_TRUSTX}
                                ${LIB_PTHREAD}
                                ${LIB_RT}
                                ${LIB_SSL}
                                ${LIB_CRYPTO}
        )
        target_include_directories (${appbin} PUBLIC "${INCDIR}")
    endforeach()
endif()


# Installation, just install libraries
set(LIB_DEST lib)
set(ENGINE_LIB_DEST lib/engines-1.1)
install(TARGETS ${LIB_TRUSTX}
    EXPORT ${CMAKE_PROJECT_NAME}-targets
    ARCHIVE DESTINATION ${LIB_DEST}
    LIBRARY DESTINATION ${LIB_DEST}
)

install(TARGETS ${LIB_TRUSTX_ENGINE}
    EXPORT ${CMAKE_PROJECT_NAME}-targets
    ARCHIVE DESTINATION ${ENGINE_LIB_DEST}
    LIBRARY DESTINATION ${ENGINE_LIB_DEST}
)
