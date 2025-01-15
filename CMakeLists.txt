
cmake_minimum_required(VERSION 3.10)
project(DeviceDashboard)


# Define source files
set(SOURCES main.c mongoose.c net.c packed_fs.c)

# Add executable target
add_executable(device-dashboard ${SOURCES})

# Custom command to bundle JS libraries
add_custom_command(
    OUTPUT ${CMAKE_SOURCE_DIR}/web_root/bundle.js
    COMMAND curl -s https://npm.reversehttp.com/preact,preact/hooks,htm/preact,preact-router -o ${CMAKE_SOURCE_DIR}/web_root/bundle.js
    COMMENT "Bundling JS libraries"
)

# Custom command to create optimized CSS
add_custom_command(
    OUTPUT ${CMAKE_SOURCE_DIR}/web_root/main.css
    DEPENDS ${CMAKE_SOURCE_DIR}/web_root/index.html
    COMMAND ${NPX} tailwindcss -o ${CMAKE_SOURCE_DIR}/web_root/main.css --minify
    COMMENT "Creating optimized CSS"
)

if (NOT DEFINED DEFAULT_WEB_ROOT)
    set(DEFAULT_WEB_ROOT ${CMAKE_SOURCE_DIR}/web_root)
endif()

add_compile_definitions(WEB_ROOT=${DEFAULT_WEB_ROOT})
target_compile_options(device-dashboard PUBLIC -Wformat -Wformat-security -Wconversion -Wsign-conversion -pedantic -Werror -DMG_ENABLE_PACKED_FS=1)

# Custom command to generate packed filesystem
add_custom_command(
    OUTPUT ${CMAKE_SOURCE_DIR}/packed_fs.c
    DEPENDS ${CMAKE_SOURCE_DIR}/web_root/main.css ${CMAKE_SOURCE_DIR}/web_root/bundle.js
    COMMAND ${GZIP} ${CMAKE_SOURCE_DIR}/web_root/*
    COMMAND ${CMAKE_C_COMPILER} ${CMAKE_SOURCE_DIR}/pack.c -o ${CMAKE_BINARY_DIR}/pack
    COMMAND ${CMAKE_BINARY_DIR}/pack ${CMAKE_SOURCE_DIR}/web_root/* ${CMAKE_SOURCE_DIR}/certs/* > ${CMAKE_SOURCE_DIR}/packed_fs.c
    COMMAND ${GZIP} -d ${CMAKE_SOURCE_DIR}/web_root/*
    COMMENT "Generating packed filesystem"
)

# Add custom target for generating packed filesystem
add_custom_target(generate_packed_fs ALL DEPENDS ${CMAKE_SOURCE_DIR}/packed_fs.c)

# Specify the install rules
install(TARGETS device-dashboard DESTINATION bin)
install(DIRECTORY ${CMAKE_SOURCE_DIR}/web_root/ DESTINATION share/device-dashboard/web_root)
