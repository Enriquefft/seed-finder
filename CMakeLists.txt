cmake_minimum_required(VERSION 3.16)

set(LANGUAGES C CXX)
project(
  SEED_FINDER
  VERSION 1.0
  DESCRIPTION "A minecraft seed finder"
  LANGUAGES ${LANGUAGES})

set(CMAKE_EXPORT_COMPILE_COMMANDS on)

if(${CMAKE_VERSION} VERSION_LESS "3.25.2")
  message(
    STATUS
      "CMake version is less than 3.25.2, upgrade to silence unnecessary warnings"
  )
endif()

# ##############################################################################
# Dependencies fetching
include(FetchContent)

# Ctest
FetchContent_Declare(
  googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG f8d7d77c06936315286eb55f8de22cd23c188571)
FetchContent_MakeAvailable(googletest)

# ##############################################################################
# Targets

set(PROJECT_APP ${PROJECT_NAME}_app)

add_executable(${PROJECT_APP} src/find_biome.c)

target_include_directories(${PROJECT_APP} PUBLIC cubiomes/ include/)

# ##############################################################################

add_library(common INTERFACE)
target_compile_features(common INTERFACE c_std_23 cxx_std_23)

# Optional IPO. Do not use IPO if it's not supported by compiler.
include(CheckIPOSupported)
check_ipo_supported(
  RESULT result
  OUTPUT output
  LANGUAGES ${LANGUAGES})
if(result)
  message(STATUS "IPO is supported")
  # TODO: verify if IPO should be enabled set_target_properties(${PROJECT_LIB}
  # PROPERTIES INTERPROCEDURAL_OPTIMIZATION TRUE)
else()
  message(WARNING "IPO is not supported")
endif()

target_compile_options(
  common
  INTERFACE
    # Preferred warnings
    $<$<CXX_COMPILER_ID:MSVC>:
    /Wall
    >
    $<$<CXX_COMPILER_ID:Clang>:-Weverything>
    $<$<CXX_COMPILER_ID:GCC>:-fconcepts-diagnostics-depth=3>
    $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:
    # $<$<COMPILE_LANGUAGE:CXX>:-Werror>
    -Wall
    -Wextra
    -Wpedantic
    -Wformat=2
    -Wshadow
    -Wwrite-strings
    -Wcast-qual
    -Wcast-align
    -Wno-switch
    -Wconversion
    -Wno-ignored-optimization-argument
    >
    # Disable some warnings when using clang's Weverything
    $<$<CXX_COMPILER_ID:Clang>:
    -Wno-c++98-compat
    -Wno-c++98-compat-pedantic
    -Wno-c++20-compat
    -Wno-c++20-extensions
    -Wno-c99-extensions
    -Wno-zero-as-null-pointer-constant
    -Wno-padded
    -Wno-global-constructors
    -Wno-exit-time-destructors # Temporarily removed due to bug
                               # (https://github.com/llvm/llvm-project/issues/59572)
    -Wno-unused-macros
    >
    $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-fsanitize=address,undefined> # #
    $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-fsanitize-address-use-after-scope>
    $<$<CONFIG:DEBUG>:
    $<$<OR:$<CXX_COMPILER_ID:Clang>,$<CXX_COMPILER_ID:GNU>>:
    -ggdb3
    -O0
    >
    $<$<CXX_COMPILER_ID:MSVC>:
    /GL
    /LTCG
    /O3>
    >
    $<$<CONFIG:RELEASE>:
    $<$<OR:$<CXX_COMPILER_ID:Clang>,$<CXX_COMPILER_ID:GNU>>:
    -O3
    >
    >)

target_link_options(
  common INTERFACE
  $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-fsanitize=address,undefined>)

# Threads needed in gcc
set(THREADS_HAVE_PTHREAD_FLAG ON)
find_package(Threads REQUIRED QUIET)
target_link_libraries(${PROJECT_APP} PUBLIC Threads::Threads)

# config
target_link_libraries(${PROJECT_APP} PUBLIC common)

# ##############################################################################
# libraries linking

# SpdLog
find_package(spdlog QUIET)
if(spdlog_FOUND)
    target_link_libraries(${PROJECT_APP} PUBLIC spdlog)
else()
  # Only use this if spdlog is not installed on your system
  message(STATUS "SpdLog not found, using bundled version")
    target_include_directories(${PROJECT_APP} SYSTEM
                             PUBLIC ${CMAKE_SOURCE_DIR}/libs/spdlog/)
endif()
target_compile_definitions(${PROJECT_APP} PUBLIC SPDLOG_USE_STD_FORMAT)
#
# link fetched libraries
target_link_libraries(${PROJECT_APP} PUBLIC common)

  add_subdirectory(cubiomes)
target_link_libraries(${PROJECT_APP} PUBLIC cubiomes)


# ##############################################################################
# GTEST setup

# For Windows: Prevent overriding the parent project's compiler/linker settings
set(gtest_force_shared_crt
    ON
    CACHE BOOL "" FORCE)

if(${PROJECT_NAME} STREQUAL ${CMAKE_PROJECT_NAME})
  option(PACKAGE_TESTS "Build
the tests" ON)
endif()

if(${PACKAGE_TESTS})
  enable_testing()
  include(GoogleTest)
  add_subdirectory(tests)
endif()
