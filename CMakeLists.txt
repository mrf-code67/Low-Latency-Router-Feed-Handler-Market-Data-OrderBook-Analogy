cmake_minimum_required(VERSION 3.16)
project(router_feed_server_refactored CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type (Debug/Release/RelWithDebInfo/MinSizeRel)" FORCE)
endif()

option(RFS_ENABLE_LTO "Enable -flto / IPO in Release" ON)
option(RFS_NO_EXCEPTIONS_RTTI "Build with -fno-exceptions -fno-rtti (demo; may terminate on OOM)" OFF)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)

set(RFS_COMMON_WARNINGS -Wall -Wextra -Wpedantic)

set(RFS_RELEASE_FLAGS
    -O3
    -DNDEBUG
    -march=native
    -mtune=native
    -fomit-frame-pointer
    -funroll-loops
    -finline-functions
)

set(RFS_DEBUG_FLAGS
    -O0
    -g3
    -fno-omit-frame-pointer
)

add_executable(router_feed_server
    src/main.cc
    src/config_loader.cpp
    src/huge_page_region.cpp
    src/router_book.cpp
    src/time_util.cpp
    src/net/udp_server.cpp
    src/net/tcp_server.cpp
    src/net/tcp_client.cpp
    src/threads/thread_util.cpp
    src/threads/io_thread.cpp
    src/threads/parser_thread.cpp
    src/threads/gap_thread.cpp
)

target_compile_options(router_feed_server PRIVATE
    ${RFS_COMMON_WARNINGS}
    $<$<CONFIG:Debug>:${RFS_DEBUG_FLAGS}>
    $<$<CONFIG:Release>:${RFS_RELEASE_FLAGS}>
)

if(RFS_NO_EXCEPTIONS_RTTI)
  target_compile_options(router_feed_server PRIVATE -fno-exceptions -fno-rtti)
endif()

if(RFS_ENABLE_LTO)
  set_property(TARGET router_feed_server PROPERTY INTERPROCEDURAL_OPTIMIZATION_RELEASE TRUE)
endif()

target_link_libraries(router_feed_server PRIVATE pthread)

add_executable(router_sim
    tools/router_sim.cpp
)

target_compile_options(router_sim PRIVATE
    ${RFS_COMMON_WARNINGS}
    $<$<CONFIG:Debug>:${RFS_DEBUG_FLAGS}>
    $<$<CONFIG:Release>:${RFS_RELEASE_FLAGS}>
)

if(RFS_NO_EXCEPTIONS_RTTI)
  target_compile_options(router_sim PRIVATE -fno-exceptions -fno-rtti)
endif()

if(RFS_ENABLE_LTO)
  set_property(TARGET router_sim PROPERTY INTERPROCEDURAL_OPTIMIZATION_RELEASE TRUE)
endif()

target_link_libraries(router_sim PRIVATE pthread)
