file(GLOB_RECURSE HOH_CPP_SOURCES CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/*.cpp")
file(GLOB_RECURSE HOH_C_SOURCES CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/*.c")

foreach(conditional_dir IN ITEMS algif algif5 alogg loadpng)
  list(FILTER HOH_CPP_SOURCES EXCLUDE REGEX ".*/${conditional_dir}/.*")
  list(FILTER HOH_C_SOURCES EXCLUDE REGEX ".*/${conditional_dir}/.*")
endforeach()

add_executable(headoverheels ${HOH_CPP_SOURCES} ${HOH_C_SOURCES})

target_include_directories(headoverheels PRIVATE
  "${CMAKE_CURRENT_SOURCE_DIR}"
  "${CMAKE_CURRENT_SOURCE_DIR}/activities"
  "${CMAKE_CURRENT_SOURCE_DIR}/behaviors"
  "${CMAKE_CURRENT_SOURCE_DIR}/gui"
  "${CMAKE_CURRENT_SOURCE_DIR}/guiactions"
)

target_compile_options(headoverheels PRIVATE
  $<$<COMPILE_LANG_AND_ID:C,GNU,Clang>:-Wall>
  $<$<COMPILE_LANG_AND_ID:CXX,GNU,Clang>:-Wall>
)

target_compile_definitions(headoverheels PRIVATE
  "PACKAGE_VERSION=\"${HOH_PACKAGE_VERSION}\""
  "BinDirFromConfigure=\"${CMAKE_INSTALL_FULL_BINDIR}\""
  "DataDirFromConfigure=\"${CMAKE_INSTALL_FULL_DATADIR}\""
  $<$<BOOL:${HOH_HAVE_EXECINFO_H}>:HAVE_EXECINFO_H=1>
  $<$<CONFIG:Debug>:DEBUG=1>
  $<$<BOOL:${APPLE}>:DARWIN=1>
)

if(HOH_GIT_HEAD_HASH)
  target_compile_definitions(headoverheels PRIVATE "GIT_HEAD_HASH=\"${HOH_GIT_HEAD_HASH}\"")
endif()

target_link_libraries(headoverheels PRIVATE Threads::Threads)

# Zlib
if(TARGET ZLIB::ZLIB)
  target_link_libraries(headoverheels PRIVATE ZLIB::ZLIB)
elseif(ZLIB_FOUND)
  target_include_directories(headoverheels PRIVATE ${ZLIB_INCLUDE_DIRS})
  target_link_libraries(headoverheels PRIVATE ${ZLIB_LIBRARIES})
else()
  message(STATUS "Zlib not found via find_package; install zlib dev package (zlib1g-dev / zlib)")
endif()

# PNG
if(TARGET PNG::PNG)
  target_link_libraries(headoverheels PRIVATE PNG::PNG)
elseif(TARGET PkgConfig::PNG)
  target_link_libraries(headoverheels PRIVATE PkgConfig::PNG)
elseif(PNG_FOUND)
  target_include_directories(headoverheels PRIVATE ${PNG_INCLUDE_DIRS})
  target_link_libraries(headoverheels PRIVATE ${PNG_LIBRARIES})
else()
  message(STATUS "libpng not found; install libpng-dev (Debian) or libpng (Arch)")
endif()

# TinyXML2
if(TARGET TinyXML2::TinyXML2)
  target_link_libraries(headoverheels PRIVATE TinyXML2::TinyXML2)
elseif(TARGET tinyxml2::tinyxml2)
  target_link_libraries(headoverheels PRIVATE tinyxml2::tinyxml2)
elseif(TARGET tinyxml2)
  target_link_libraries(headoverheels PRIVATE tinyxml2)
elseif(TinyXML2_FOUND)
  if(DEFINED TinyXML2_INCLUDE_DIRS)
    target_include_directories(headoverheels PRIVATE ${TinyXML2_INCLUDE_DIRS})
  endif()
  if(DEFINED TinyXML2_LIBRARIES)
    target_link_libraries(headoverheels PRIVATE ${TinyXML2_LIBRARIES})
  endif()
else()
  message(STATUS "tinyxml2 not found; install libtinyxml2-dev or enable -DUSE_BUNDLED_TINYXML2=ON")
endif()

# Ogg
if(TARGET PkgConfig::OGG)
  target_link_libraries(headoverheels PRIVATE PkgConfig::OGG)
elseif(OGG_FOUND)
  target_include_directories(headoverheels PRIVATE ${OGG_INCLUDE_DIRS})
  target_link_libraries(headoverheels PRIVATE ${OGG_LIBRARIES})
else()
  message(STATUS "libogg not found; install libogg-dev or libogg")
endif()

# Vorbis
if(TARGET PkgConfig::VORBIS)
  target_link_libraries(headoverheels PRIVATE PkgConfig::VORBIS)
elseif(VORBIS_FOUND)
  target_include_directories(headoverheels PRIVATE ${VORBIS_INCLUDE_DIRS})
  target_link_libraries(headoverheels PRIVATE ${VORBIS_LIBRARIES})
else()
  message(STATUS "libvorbis not found; install libvorbis-dev or libvorbis")
endif()

if(TARGET PkgConfig::VORBISFILE)
  target_link_libraries(headoverheels PRIVATE PkgConfig::VORBISFILE)
else()
  find_library(VORBISFILE_LIB NAMES vorbisfile vorbisfile-3)
  if(VORBISFILE_LIB)
    target_link_libraries(headoverheels PRIVATE ${VORBISFILE_LIB})
  endif()
endif()

if(TARGET PkgConfig::VORBISENC)
  target_link_libraries(headoverheels PRIVATE PkgConfig::VORBISENC)
else()
  find_library(VORBISENC_LIB NAMES vorbisenc)
  if(VORBISENC_LIB)
    target_link_libraries(headoverheels PRIVATE ${VORBISENC_LIB})
  endif()
endif()

set(HOH_ALLEGRO4_SOURCES
  "${CMAKE_CURRENT_SOURCE_DIR}/algif/algif.c"
  "${CMAKE_CURRENT_SOURCE_DIR}/algif/gif.c"
  "${CMAKE_CURRENT_SOURCE_DIR}/algif/lzw.c"
  "${CMAKE_CURRENT_SOURCE_DIR}/alogg/alogg.c"
  "${CMAKE_CURRENT_SOURCE_DIR}/alogg/aloggpth.c"
  "${CMAKE_CURRENT_SOURCE_DIR}/loadpng/loadpng.c"
  "${CMAKE_CURRENT_SOURCE_DIR}/loadpng/regpng.c"
  "${CMAKE_CURRENT_SOURCE_DIR}/loadpng/savepng.c"
)

set(HOH_ALLEGRO5_SOURCES
  "${CMAKE_CURRENT_SOURCE_DIR}/algif5/algif.c"
  "${CMAKE_CURRENT_SOURCE_DIR}/algif5/gif.c"
  "${CMAKE_CURRENT_SOURCE_DIR}/algif5/lzw.c"
  "${CMAKE_CURRENT_SOURCE_DIR}/algif5/bitmap.c"
)

if(USE_ALLEGRO5)
  target_sources(headoverheels PRIVATE ${HOH_ALLEGRO5_SOURCES})
  target_compile_definitions(headoverheels PRIVATE USE_ALLEGRO5=1)
  message(FATAL_ERROR "USE_ALLEGRO5 is not wired to Allegro5 libraries yet")
elseif(TARGET PkgConfig::ALLEGRO4)
  target_sources(headoverheels PRIVATE ${HOH_ALLEGRO4_SOURCES})
  target_link_libraries(headoverheels PRIVATE PkgConfig::ALLEGRO4)
  target_compile_definitions(headoverheels PRIVATE USE_ALLEGRO4=1)
elseif(HOH_BUNDLED_ALLEGRO4)
  target_sources(headoverheels PRIVATE ${HOH_ALLEGRO4_SOURCES})
  add_dependencies(headoverheels allegro4)
  target_include_directories(headoverheels PRIVATE "${ALLEGRO4_INCLUDE_DIRS}")
  target_link_libraries(headoverheels PRIVATE "${ALLEGRO4_LIBRARIES}")
  target_compile_definitions(headoverheels PRIVATE USE_ALLEGRO4=1)

  if(HOH_BUNDLED_ALLEGRO4_STATIC)
    target_compile_definitions(headoverheels PRIVATE ALLEGRO_STATICLINK=1)
    target_link_libraries(headoverheels PRIVATE m ${CMAKE_DL_LIBS})

    find_library(RT_LIBRARY rt)
    if(RT_LIBRARY)
      target_link_libraries(headoverheels PRIVATE ${RT_LIBRARY})
    endif()

    find_package(X11)
    if(X11_FOUND)
      target_link_libraries(headoverheels PRIVATE ${X11_LIBRARIES})
      foreach(x11_lib IN ITEMS X11_Xext_LIB X11_Xcursor_LIB X11_Xpm_LIB X11_Xxf86vm_LIB)
        if(DEFINED ${x11_lib} AND ${x11_lib})
          target_link_libraries(headoverheels PRIVATE "${${x11_lib}}")
        endif()
      endforeach()
    endif()

    foreach(extra_lib IN ITEMS SM ICE)
      find_library(${extra_lib}_LIBRARY ${extra_lib})
      if(${extra_lib}_LIBRARY)
        target_link_libraries(headoverheels PRIVATE "${${extra_lib}_LIBRARY}")
      endif()
    endforeach()
  endif()
elseif(ALLEGRO4_FOUND)
  target_sources(headoverheels PRIVATE ${HOH_ALLEGRO4_SOURCES})
  target_include_directories(headoverheels PRIVATE ${ALLEGRO4_INCLUDE_DIRS})
  target_link_libraries(headoverheels PRIVATE ${ALLEGRO4_LIBRARIES})
  target_compile_definitions(headoverheels PRIVATE USE_ALLEGRO4=1)
else()
  message(FATAL_ERROR "Allegro4 not found. Install the dev package or enable -DUSE_BUNDLED_ALLEGRO=ON.")
endif()

install(TARGETS headoverheels RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}")
