cmake_minimum_required(VERSION 3.11.0 FATAL_ERROR)
project(cppcoro LANGUAGES CXX)

option(CPPCORO_BUILD_TESTS "Build tests" ${BUILD_TESTS})
option(CPPCORO_EXTRA_WARNINGS "Build with additional warnings" ${EXTRA_WARNINGS})

if(MSVC)
  add_definitions(-D_SILENCE_ALL_CXX17_DEPRECATION_WARNINGS)
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
  option(LLVM_INSTALL_PREFIX "LLVM install prefix" OFF)
  if(LLVM_INSTALL_PREFIX)
    add_compile_options(-nostdinc++)
    include_directories(${LLVM_INSTALL_PREFIX}/include/c++/v1)
    link_directories(${LLVM_INSTALL_PREFIX}/lib)
    link_libraries(c c++)
  endif()
endif()

if(CPPCORO_EXTRA_WARNINGS)
  if(MSVC)
    string(REGEX REPLACE "[/-]W[0-9]" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4 /permissive-")
  else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -Wpedantic -Wconversion -Wno-unused-parameter")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fdiagnostics-absolute-paths -fcolor-diagnostics")
  endif()
endif()

set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set_property(GLOBAL PROPERTY PREDEFINED_TARGETS_FOLDER build)

file(GLOB_RECURSE headers include/cppcoro/*.hpp)
source_group(TREE ${CMAKE_CURRENT_SOURCE_DIR}/include/cppcoro PREFIX cppcoro FILES ${headers})

file(GLOB sources
  lib/async_auto_reset_event.cpp
  lib/async_manual_reset_event.cpp
  lib/async_mutex.cpp
  lib/cancellation_state.cpp
  lib/cancellation_token.cpp
  lib/cancellation_source.cpp
  lib/cancellation_registration.cpp
  lib/lightweight_manual_reset_event.cpp
  lib/ip_address.cpp
  lib/ip_endpoint.cpp
  lib/ipv4_address.cpp
  lib/ipv4_endpoint.cpp
  lib/ipv6_address.cpp
  lib/ipv6_endpoint.cpp)
if(WIN32)
  file(GLOB win32_sources
    lib/win32_overlapped_operation.hpp
    lib/win32.cpp
    lib/io_service.cpp
    lib/file.cpp
    lib/readable_file.cpp
    lib/writable_file.cpp
    lib/read_only_file.cpp
    lib/write_only_file.cpp
    lib/read_write_file.cpp
    lib/file_read_operation.cpp
    lib/file_write_operation.cpp)
  list(APPEND sources ${win32_sources})
endif()
source_group(TREE ${CMAKE_CURRENT_SOURCE_DIR}/lib PREFIX lib FILES ${sources})

add_library(cppcoro STATIC ${headers} ${sources})
set_target_properties(cppcoro PROPERTIES CXX_EXTENSIONS OFF)
target_compile_features(cppcoro PUBLIC cxx_std_17)

target_include_directories(cppcoro PRIVATE lib PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>)

if(WIN32)
  target_compile_options(cppcoro PUBLIC /await)
  target_link_libraries(cppcoro PUBLIC ws2_32 mswsock synchronization)
else()
  find_package(Threads REQUIRED)
  target_link_libraries(cppcoro PUBLIC Threads::Threads)
  target_compile_options(cppcoro PUBLIC -fcoroutines-ts)
endif()

install(TARGETS cppcoro EXPORT cppcoro LIBRARY DESTINATION lib ARCHIVE DESTINATION lib)

if(NOT EXISTS ${CMAKE_CURRENT_BINARY_DIR}/cppcoro-config.cmake.in)
  file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/cppcoro-config.cmake.in
    [[@PACKAGE_INIT@]] "\n"
    [[include("${CMAKE_CURRENT_LIST_DIR}/cppcoro-targets.cmake")]])
endif()

include(CMakePackageConfigHelpers)

configure_package_config_file(${CMAKE_CURRENT_BINARY_DIR}/cppcoro-config.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/cppcoro-config.cmake INSTALL_DESTINATION lib/cmake/cppcoro)

install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/ DESTINATION include FILES_MATCHING PATTERN "*.hpp")
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/cppcoro-config.cmake DESTINATION lib/cmake/cppcoro)
install(EXPORT cppcoro FILE cppcoro-targets.cmake NAMESPACE cppcoro:: DESTINATION lib/cmake/cppcoro)

if(CPPCORO_BUILD_TESTS)
  file(GLOB tests_sources
    test/main.cpp
    test/counted.cpp
    test/generator_tests.cpp
    test/recursive_generator_tests.cpp
    test/async_generator_tests.cpp
    test/async_auto_reset_event_tests.cpp
    test/async_manual_reset_event_tests.cpp
    test/async_mutex_tests.cpp
    test/async_latch_tests.cpp
    test/cancellation_token_tests.cpp
    test/task_tests.cpp
    test/shared_task_tests.cpp
    test/sync_wait_tests.cpp
    test/single_consumer_async_auto_reset_event_tests.cpp
    test/when_all_tests.cpp
    test/when_all_ready_tests.cpp
    test/ip_address_tests.cpp
    test/ip_endpoint_tests.cpp
    test/ipv4_address_tests.cpp
    test/ipv4_endpoint_tests.cpp
    test/ipv6_address_tests.cpp
    test/ipv6_endpoint_tests.cpp)
  if(WIN32)
    file(GLOB win32_tests_sources
      test/scheduling_operator_tests.cpp
      test/io_service_tests.cpp
      test/file_tests.cpp)
    list(APPEND tests_sources ${win32_tests_sources})
  endif()
  source_group(TREE ${CMAKE_CURRENT_SOURCE_DIR}/test PREFIX test FILES ${tests_sources})
  add_executable(tests ${tests_sources})
  target_link_libraries(tests PRIVATE cppcoro)
  set_target_properties(tests PROPERTIES VS_DEBUGGER_WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
  set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT tests)
  add_custom_target(run_tests COMMAND $<TARGET_FILE:tests> DEPENDS tests
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} USES_TERMINAL)
endif()
