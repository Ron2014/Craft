cmake_minimum_required(VERSION 2.8)

project(craft)

FILE(GLOB SOURCE_FILES src/*.c)

add_executable(
    craft
    ${SOURCE_FILES}
    deps/glew/src/glew.c
    deps/lodepng/lodepng.c
    deps/noise/noise.c
    deps/sqlite/sqlite3.c
    deps/tinycthread/tinycthread.c)

add_definitions(-std=c99 -O3)

add_subdirectory(deps/glfw)
include_directories(deps/glew/include)
include_directories(deps/glfw/include)
include_directories(deps/lodepng)
include_directories(deps/noise)
include_directories(deps/sqlite)
include_directories(deps/tinycthread)

if(MINGW)
    set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH}
        "C:/Program Files/CURL/bib" "C:/Program Files (x86)/CURL/lib")
    set(CMAKE_INCLUDE_PATH ${CMAKE_INCLUDE_PATH}
        "C:/Program Files/CURL/include" "C:/Program Files (x86)/CURL/include")
#    set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH}
#        "E:/GitHub/MineCraft/deps/CURL/lib")
#    set(CMAKE_INCLUDE_PATH ${CMAKE_INCLUDE_PATH}
#        "E:/GitHub/MineCraft/deps/CURL/include")
endif()

find_package(CURL REQUIRED)
include_directories(${CURL_INCLUDE_DIR})

if(NOT GLFW_LIBRARIES)
message("====== GLFW NOT FOUND")
endif()

if(NOT CURL_LIBRARIES)
message("====== CURLS NOT FOUND")
endif()

message("====== ${CURL_INCLUDE_DIR}")
message("====== ${CURL_LIBRARIES}")
message("====== ${GLFW_LIBRARIES}")

if(APPLE)
    target_link_libraries(craft glfw
        ${GLFW_LIBRARIES} ${CURL_LIBRARIES})
endif()

if(UNIX)
    target_link_libraries(craft dl glfw
        ${GLFW_LIBRARIES} ${CURL_LIBRARIES})
endif()

if(MINGW)
    target_link_libraries(craft ws2_32.lib glfw
        ${GLFW_LIBRARIES} ${CURL_LIBRARIES})
endif()
