cmake_minimum_required(VERSION 3.10)
project(nam-binary-loader VERSION 0.1.0)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED OFF)
set(CMAKE_CXX_EXTENSIONS OFF)

# Use libc++ on macOS, system default on Linux
if (CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
elseif (CMAKE_SYSTEM_NAME STREQUAL "Linux")
  link_libraries(stdc++fs)
endif()

# Path to NeuralAmpModelerCore (sibling directory by default)
set(NAM_CORE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/../NeuralAmpModelerCore" CACHE PATH
    "Path to NeuralAmpModelerCore source directory")

if (NOT EXISTS "${NAM_CORE_PATH}/NAM/dsp.h")
  message(FATAL_ERROR "NeuralAmpModelerCore not found at ${NAM_CORE_PATH}. "
          "Set NAM_CORE_PATH to the NeuralAmpModelerCore source directory.")
endif()

set(NAM_DEPS_PATH "${NAM_CORE_PATH}/Dependencies")

# Include paths
include_directories(SYSTEM "${NAM_DEPS_PATH}/eigen")
include_directories("${NAM_DEPS_PATH}/nlohmann")  # for json.hpp
include_directories("${NAM_CORE_PATH}")            # for <NAM/...> includes
include_directories("${CMAKE_CURRENT_SOURCE_DIR}/include")  # for <namb/...> includes

# NAM core sources
file(GLOB_RECURSE NAM_SOURCES "${NAM_CORE_PATH}/NAM/*.cpp" "${NAM_CORE_PATH}/NAM/*.c")

# NAMB loader source
set(NAMB_SOURCES
    "${CMAKE_CURRENT_SOURCE_DIR}/include/namb/get_dsp_namb.cpp"
)

# --- nam2namb converter tool ---
add_executable(nam2namb tools/nam2namb.cpp ${NAM_SOURCES})

# --- loadmodel tool (supports both .nam and .namb) ---
add_executable(loadmodel tools/loadmodel.cpp ${NAM_SOURCES} ${NAMB_SOURCES})

# --- test runner ---
add_executable(test_namb test/test_namb.cpp ${NAM_SOURCES} ${NAMB_SOURCES})
# Ensure assertions are enabled
set_target_properties(test_namb PROPERTIES COMPILE_OPTIONS "-O0")
target_compile_options(test_namb PRIVATE
  $<$<OR:$<CONFIG:Release>,$<CONFIG:RelWithDebInfo>,$<CONFIG:MinSizeRel>>:-UNDEBUG>
)

# Compiler warnings
if (MSVC)
  set(WARN_FLAGS /W4)
else()
  set(WARN_FLAGS -Wall -Wextra -Wpedantic -Wno-unused-parameter)
endif()

target_compile_options(nam2namb PRIVATE ${WARN_FLAGS})
target_compile_options(loadmodel PRIVATE ${WARN_FLAGS})
target_compile_options(test_namb PRIVATE ${WARN_FLAGS})

# Suppress Eigen warnings in NAM sources
if (NOT MSVC)
  set_source_files_properties(${NAM_SOURCES} PROPERTIES COMPILE_FLAGS "-Wno-error")
endif()
