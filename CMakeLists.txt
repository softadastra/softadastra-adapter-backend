cmake_minimum_required(VERSION 3.20)
project(studio LANGUAGES CXX)

# ===== C++ standard =====
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(VIX_ENABLE_SANITIZERS "Enable ASan/UBSan (dev only)" OFF)

# ===== Prefixes (adjust if your install lives elsewhere) =====
list(APPEND CMAKE_PREFIX_PATH 
  "/usr/local"
  "/usr/local/lib/cmake"
  "/usr/local/lib/cmake/vix"  # new lowercase package dir
  "/usr/local/lib/cmake/Vix"  # legacy package dir (exports fmt::fmt)
)

# ❗ Disable ORM completely in generated apps (core-only)
set(CMAKE_DISABLE_FIND_PACKAGE_VixOrm ON)

# ===== System deps first =====
find_package(Threads REQUIRED)
find_package(OpenSSL QUIET)
find_package(Boost REQUIRED COMPONENTS system thread filesystem)

# Optional: create a small shim if Boost::filesystem isn't exported
if (NOT TARGET Boost::filesystem)
  if (Boost_FOUND AND Boost_FILESYSTEM_LIBRARY)
    add_library(Boost::filesystem UNKNOWN IMPORTED)
    set_target_properties(Boost::filesystem PROPERTIES
      IMPORTED_LOCATION "${Boost_FILESYSTEM_LIBRARY}"
      INTERFACE_INCLUDE_DIRECTORIES "${Boost_INCLUDE_DIRS}")
    message(STATUS "Shim: Using Boost::filesystem from ${Boost_FILESYSTEM_LIBRARY}")
  else()
    message(FATAL_ERROR "Boost::filesystem not found and no library path provided.")
  endif()
endif()

# ===== fmt shim (must be BEFORE find_package(Vix/vix)) =====
# Some legacy Vix packages export vix::utils with fmt::fmt in interface.
# To avoid configure errors, ensure fmt::fmt exists (alias header-only or dummy).
find_package(fmt QUIET)
if (TARGET fmt::fmt-header-only AND NOT TARGET fmt::fmt)
  add_library(fmt::fmt ALIAS fmt::fmt-header-only)
endif()
if (NOT TARGET fmt::fmt)
  add_library(fmt::fmt INTERFACE IMPORTED)
  target_include_directories(fmt::fmt INTERFACE "/usr/include" "/usr/local/include")
endif()

# ===== Vix (core-only) — prefer new lowercase package, keep legacy fallback =====
set(vix_FOUND FALSE)
find_package(vix QUIET CONFIG)        # new
if (vix_FOUND)
  message(STATUS "Found vix (lowercase) package config")
else()
  find_package(Vix QUIET CONFIG)       # legacy
  if (Vix_FOUND)
    message(STATUS "Found Vix (legacy) package config")
    set(vix_FOUND TRUE)
  endif()
endif()

if (NOT vix_FOUND)
  message(FATAL_ERROR "Could not find Vix/vix package config. Set CMAKE_PREFIX_PATH to your install prefix.")
endif()

# Decide main target name (new vs legacy namespace)
if (TARGET vix::vix)
  set(VIX_MAIN_TARGET vix::vix)
elseif (TARGET Vix::vix)
  set(VIX_MAIN_TARGET Vix::vix)
else()
  message(FATAL_ERROR "Neither vix::vix nor Vix::vix target exists. Check your Vix/vix installation.")
endif()

# ===== App target =====
add_executable(studio src/main.cpp)

target_link_libraries(studio PRIVATE
  ${VIX_MAIN_TARGET}
  Boost::system Boost::thread Boost::filesystem
  Threads::Threads
)

if (OpenSSL_FOUND)
  target_link_libraries(studio PRIVATE OpenSSL::SSL OpenSSL::Crypto)
endif()

if (UNIX AND NOT APPLE)
  target_link_libraries(studio PRIVATE dl)
endif()

# Warnings
if (MSVC)
  target_compile_options(studio PRIVATE /W4 /permissive-)
else()
  target_compile_options(studio PRIVATE -Wall -Wextra -Wpedantic)
endif()

# Sanitizers (optional)
if (VIX_ENABLE_SANITIZERS AND NOT MSVC)
  target_compile_options(studio PRIVATE -O1 -g -fno-omit-frame-pointer -fsanitize=address,undefined)
  target_link_options(studio PRIVATE -fsanitize=address,undefined)
endif()

# Run helper
add_custom_target(run
  COMMAND $<TARGET_FILE:studio>
  DEPENDS studio
  USES_TERMINAL
  WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
)

set_target_properties(studio PROPERTIES
  RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}"
  INSTALL_RPATH_USE_LINK_PATH ON
)
