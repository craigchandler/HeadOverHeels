cmake_minimum_required(VERSION 3.15)
project(HeadOverHeels VERSION 1.4.0 LANGUAGES C CXX)

set(HOH_PACKAGE_NAME "headoverheels")
set(HOH_PACKAGE_VERSION "1.4dev")
set(CMAKE_INSTALL_MESSAGE LAZY CACHE STRING "Install message verbosity")
set_property(CACHE CMAKE_INSTALL_MESSAGE PROPERTY STRINGS ALWAYS LAZY NEVER)

option(USE_SYSTEM_ZLIB "Use system zlib (prefer distro dev package)" ON)
option(USE_SYSTEM_LIBPNG "Use system libpng (prefer distro dev package)" ON)
option(USE_SYSTEM_OGG "Use system libogg (prefer distro dev package)" ON)
option(USE_SYSTEM_VORBIS "Use system libvorbis (prefer distro dev package)" ON)
option(USE_SYSTEM_TINYXML2 "Use system tinyxml2 (prefer distro dev package)" ON)
option(USE_BUNDLED_TINYXML2 "Build bundled tinyxml2 if system not available" OFF)
option(USE_BUNDLED_ALLEGRO "Use bundled static Allegro4 from external/ instead of system Allegro4" OFF)
option(USE_ALLEGRO5 "Build the Allegro5 backend instead of Allegro4" OFF)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(CheckIncludeFile)
include(FetchContent)
include(GNUInstallDirs)

if(POLICY CMP0135)
  cmake_policy(SET CMP0135 NEW)
endif()

check_include_file(execinfo.h HOH_HAVE_EXECINFO_H)

find_package(PkgConfig REQUIRED)
find_package(Threads REQUIRED)
find_package(Git QUIET)

set(HOH_GIT_HEAD_HASH "")
if(GIT_FOUND AND EXISTS "${CMAKE_SOURCE_DIR}/.git")
  execute_process(
    COMMAND "${GIT_EXECUTABLE}" rev-parse --short HEAD
    WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
    OUTPUT_VARIABLE HOH_GIT_HEAD_HASH
    OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_QUIET)
endif()

# Zlib
if(USE_SYSTEM_ZLIB)
  find_package(ZLIB QUIET)
  if(ZLIB_FOUND)
    message(STATUS "Found ZLIB: ${ZLIB_VERSION}")
  endif()
endif()

# PNG
if(USE_SYSTEM_LIBPNG)
  find_package(PNG QUIET)
  if(NOT PNG_FOUND)
    pkg_check_modules(PNG QUIET IMPORTED_TARGET libpng16)
  endif()
  if(NOT PNG_FOUND)
    pkg_check_modules(PNG QUIET IMPORTED_TARGET libpng)
  endif()
  if(PNG_FOUND)
    message(STATUS "Found PNG")
  endif()
endif()

# TinyXML2
if(USE_SYSTEM_TINYXML2)
  find_package(TinyXML2 QUIET)
endif()

if(NOT TinyXML2_FOUND AND USE_BUNDLED_TINYXML2)
  message(STATUS "TinyXML2 not found: will fetch bundled tinyxml2")
  FetchContent_Declare(
    tinyxml2
    URL https://github.com/leethomason/tinyxml2/archive/refs/tags/11.0.0.tar.gz
    URL_HASH SHA256=5556deb5081fb246ee92afae73efd943c889cef0cafea92b0b82422d6a18f289)
  FetchContent_GetProperties(tinyxml2)
  if(NOT tinyxml2_POPULATED)
    FetchContent_Populate(tinyxml2)
  endif()
  if(NOT TARGET tinyxml2)
    add_library(tinyxml2 STATIC "${tinyxml2_SOURCE_DIR}/tinyxml2.cpp")
    add_library(tinyxml2::tinyxml2 ALIAS tinyxml2)
    target_include_directories(tinyxml2 PUBLIC "${tinyxml2_SOURCE_DIR}")
  endif()
  set(TinyXML2_FOUND TRUE)
endif()

# Ogg / Vorbis via pkg-config
if(USE_SYSTEM_OGG)
  pkg_check_modules(OGG QUIET IMPORTED_TARGET ogg)
endif()
if(USE_SYSTEM_VORBIS)
  pkg_check_modules(VORBIS QUIET IMPORTED_TARGET vorbis)
  pkg_check_modules(VORBISFILE QUIET IMPORTED_TARGET vorbisfile)
  pkg_check_modules(VORBISENC QUIET IMPORTED_TARGET vorbisenc)
endif()

# Allegro4: try pkg-config first, otherwise allow building bundled copy.
if(NOT USE_ALLEGRO5 AND NOT USE_BUNDLED_ALLEGRO)
  pkg_check_modules(ALLEGRO4 QUIET IMPORTED_TARGET allegro-4)
  if(NOT ALLEGRO4_FOUND)
    pkg_check_modules(ALLEGRO4 QUIET IMPORTED_TARGET allegro)
  endif()
endif()

if(NOT USE_ALLEGRO5 AND USE_BUNDLED_ALLEGRO)
  message(STATUS "Using bundled static Allegro4")
  include(ExternalProject)

  set(ALLEGRO4_TARBALL "${CMAKE_SOURCE_DIR}/external/allegro/allegro-4.4.3.1.tar.gz")
  if(NOT EXISTS "${ALLEGRO4_TARBALL}")
    message(FATAL_ERROR "Expected Allegro tarball at ${ALLEGRO4_TARBALL} for bundled build. Place tarball there or disable USE_BUNDLED_ALLEGRO.")
  endif()

  set(ALLEGRO4_INSTALL_DIR "${CMAKE_BINARY_DIR}/allegro4" CACHE PATH "Allegro install dir")
  set(ALLEGRO4_STATIC_LIBRARY "${ALLEGRO4_INSTALL_DIR}/lib/liballeg.a")

  set(ALLEGRO4_TWEAK_SCRIPT "${CMAKE_BINARY_DIR}/allegro4_tweak.cmake")
  file(WRITE "${ALLEGRO4_TWEAK_SCRIPT}" "if(NOT DEFINED HOH_ALLEGRO4_SOURCE_DIR)\n")
  file(APPEND "${ALLEGRO4_TWEAK_SCRIPT}" "  message(FATAL_ERROR \"HOH_ALLEGRO4_SOURCE_DIR is required\")\n")
  file(APPEND "${ALLEGRO4_TWEAK_SCRIPT}" "endif()\n")
  file(APPEND "${ALLEGRO4_TWEAK_SCRIPT}" "set(allegro_cmake \"\${HOH_ALLEGRO4_SOURCE_DIR}/CMakeLists.txt\")\n")
  file(APPEND "${ALLEGRO4_TWEAK_SCRIPT}" "file(READ \"\${allegro_cmake}\" allegro_cmake_contents)\n")
  file(APPEND "${ALLEGRO4_TWEAK_SCRIPT}" "string(REPLACE \"add_subdirectory(docs)\" \"# add_subdirectory(docs)\" allegro_cmake_contents \"\${allegro_cmake_contents}\")\n")
  file(APPEND "${ALLEGRO4_TWEAK_SCRIPT}" "file(WRITE \"\${allegro_cmake}\" \"\${allegro_cmake_contents}\")\n")

  set(ALLEGRO4_PATCH_SCRIPT "${CMAKE_BINARY_DIR}/allegro4_apply_patches.sh")
  file(WRITE "${ALLEGRO4_PATCH_SCRIPT}" "#!/bin/sh\n")
  file(APPEND "${ALLEGRO4_PATCH_SCRIPT}" "'${CMAKE_COMMAND}' -DHOH_ALLEGRO4_SOURCE_DIR=\"$PWD\" -P '${ALLEGRO4_TWEAK_SCRIPT}'\n")
  file(APPEND "${ALLEGRO4_PATCH_SCRIPT}" "if patch -p1 --forward --dry-run -i '${CMAKE_SOURCE_DIR}/external/allegro/false_three_finger.patch' >/dev/null 2>&1; then\n")
  file(APPEND "${ALLEGRO4_PATCH_SCRIPT}" "  patch -p1 -i '${CMAKE_SOURCE_DIR}/external/allegro/false_three_finger.patch'\n")
  file(APPEND "${ALLEGRO4_PATCH_SCRIPT}" "fi\n")
  file(APPEND "${ALLEGRO4_PATCH_SCRIPT}" "'${CMAKE_COMMAND}' -E copy_if_different '${CMAKE_SOURCE_DIR}/gamedata/head48.icon.png' misc/icon.png || true\n")
  file(APPEND "${ALLEGRO4_PATCH_SCRIPT}" "'${CMAKE_COMMAND}' -E copy_if_different '${CMAKE_SOURCE_DIR}/gamedata/icon.xpm' misc/icon.xpm || true\n")
  file(APPEND "${ALLEGRO4_PATCH_SCRIPT}" "'${CMAKE_COMMAND}' -E copy_if_different '${CMAKE_SOURCE_DIR}/gamedata/icon.xpm' src/x/icon.xpm || true\n")

  ExternalProject_Add(allegro4
    PREFIX "${ALLEGRO4_INSTALL_DIR}"
    INSTALL_DIR "${ALLEGRO4_INSTALL_DIR}"
    DOWNLOAD_COMMAND "${CMAKE_COMMAND}" -E make_directory <SOURCE_DIR> && /bin/sh -c "tar xzf '${ALLEGRO4_TARBALL}' -C '<SOURCE_DIR>' --strip-components=1"
    PATCH_COMMAND "${CMAKE_COMMAND}" -E chdir <SOURCE_DIR> /bin/sh "${ALLEGRO4_PATCH_SCRIPT}"
    CONFIGURE_COMMAND "${CMAKE_COMMAND}" -Wno-dev -Wno-deprecated -S <SOURCE_DIR> -B <BINARY_DIR>
      -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
      -DSHARED=OFF
      -DWANT_ALLEGROGL=OFF
      -DWANT_LOADPNG=OFF
      -DWANT_LOGG=OFF
      -DWANT_JPGALLEG=OFF
      -DWANT_EXAMPLES=OFF
      -DWANT_TOOLS=OFF
      -DWANT_TESTS=OFF
      -DCMAKE_C_FLAGS=-fno-common
    BUILD_COMMAND "${CMAKE_COMMAND}" --build <BINARY_DIR>
    INSTALL_COMMAND "${CMAKE_COMMAND}" --install <BINARY_DIR> --prefix <INSTALL_DIR>
    BUILD_BYPRODUCTS "${ALLEGRO4_STATIC_LIBRARY}"
    UPDATE_COMMAND "")

  set(ALLEGRO4_INCLUDE_DIRS "${ALLEGRO4_INSTALL_DIR}/include")
  set(ALLEGRO4_LIBRARIES "${ALLEGRO4_STATIC_LIBRARY}")
  set(HOH_BUNDLED_ALLEGRO4 TRUE)
  set(HOH_BUNDLED_ALLEGRO4_STATIC TRUE)
endif()

add_subdirectory(source)
add_subdirectory(gamedata)

message(STATUS "CMake configuration complete. Example build:")
message(STATUS "  cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=$HOME/.local && cmake --build build --parallel")
