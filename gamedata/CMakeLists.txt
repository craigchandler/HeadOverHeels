include(GNUInstallDirs)

set(HOH_GAMEDATA_INSTALL_DIR "${CMAKE_INSTALL_DATADIR}/${HOH_PACKAGE_NAME}")
set(HOH_GAMEDATA_STAGE_PREFIX "${CMAKE_BINARY_DIR}" CACHE PATH
    "Install prefix used by the install-gamedata target")

function(hoh_install_automake_data makefile data_variable destination)
  set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS "${makefile}")
  file(READ "${makefile}" makefile_contents)
  string(REPLACE "\\\n" " " makefile_contents "${makefile_contents}")

  string(REGEX MATCH
         "(^|\n)[ \t]*${data_variable}[ \t]*=[ \t]*([^\n]*)"
         data_assignment
         "${makefile_contents}")
  if(NOT data_assignment)
    message(FATAL_ERROR "Could not find ${data_variable} in ${makefile}")
  endif()

  set(data_files "${CMAKE_MATCH_2}")
  string(REPLACE "\t" " " data_files "${data_files}")
  string(STRIP "${data_files}" data_files)
  separate_arguments(data_files UNIX_COMMAND "${data_files}")

  get_filename_component(source_dir "${makefile}" DIRECTORY)
  set(sources)
  foreach(data_file IN LISTS data_files)
    list(APPEND sources "${source_dir}/${data_file}")
  endforeach()

  # Automake's *_DATA install rules flatten path components in these lists.
  install(FILES ${sources}
          DESTINATION "${destination}"
          COMPONENT gamedata)
endfunction()

hoh_install_automake_data(
  "${CMAKE_CURRENT_SOURCE_DIR}/Makefile.am"
  gamedata_DATA
  "${HOH_GAMEDATA_INSTALL_DIR}")

function(hoh_install_gamedata_subdir subdir data_prefix)
  hoh_install_automake_data(
    "${CMAKE_CURRENT_SOURCE_DIR}/${subdir}/Makefile.am"
    "${data_prefix}_DATA"
    "${HOH_GAMEDATA_INSTALL_DIR}/${subdir}")
endfunction()

hoh_install_gamedata_subdir(map gamedatamap)
hoh_install_gamedata_subdir(music gamedatamusic)
hoh_install_gamedata_subdir(savegame gamedatasavegame)
hoh_install_gamedata_subdir(sound gamedatasound)
hoh_install_gamedata_subdir(text gamedatatext)
hoh_install_gamedata_subdir(gfx gamedatagfx)
hoh_install_gamedata_subdir(gfx.2009 gamedatagfx2009)
hoh_install_gamedata_subdir(gfx.riderx gamedatagfxriderx)
hoh_install_gamedata_subdir(gfx.2003 gamedatagfx2003)
hoh_install_gamedata_subdir(gfx.simple gamedatagfxsimple)

add_custom_target(install-gamedata
  COMMAND "${CMAKE_COMMAND}"
          --install "${CMAKE_BINARY_DIR}"
          --prefix "${HOH_GAMEDATA_STAGE_PREFIX}"
          --component gamedata
  COMMENT "Installing gamedata to ${HOH_GAMEDATA_STAGE_PREFIX}/${HOH_GAMEDATA_INSTALL_DIR}"
  VERBATIM)
