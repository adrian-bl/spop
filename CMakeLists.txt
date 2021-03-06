# Copyright (C) 2010, 2011, 2012, 2013 Thomas Jost
#
# This file is part of spop.
#
# spop is free software: you can redistribute it and/or modify it under the
# terms of the GNU General Public License as published by the Free Software
# Foundation, either version 3 of the License, or (at your option) any later
# version.
#
# spop is distributed in the hope that it will be useful, but WITHOUT ANY
# WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR
# A PARTICULAR PURPOSE. See the GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License along with
# spop. If not, see <http://www.gnu.org/licenses/>.
#
# Additional permission under GNU GPL version 3 section 7
#
# If you modify this Program, or any covered work, by linking or combining it
# with libspotify (or a modified version of that library), containing parts
# covered by the terms of the Libspotify Terms of Use, the licensors of this
# Program grant you additional permission to convey the resulting work.

cmake_minimum_required(VERSION 2.8)

project(spop C)

find_package(PkgConfig)
set(targets)

# Check for libspotify
pkg_check_modules(SPOTIFY REQUIRED libspotify>=12.1.45)
string(REPLACE ";" " " SPOTIFY_CFLAGS "${SPOTIFY_CFLAGS}")

# Check for required libraries
pkg_check_modules(GLIB2 REQUIRED glib-2.0>=2.26)
string(REPLACE ";" " " GLIB2_CFLAGS "${GLIB2_CFLAGS}")
pkg_check_modules(GMODULE2 REQUIRED gmodule-2.0)
string(REPLACE ";" " " GMODULE2_CFLAGS "${GMODULE2_CFLAGS}")
pkg_check_modules(GTHREAD2 REQUIRED gthread-2.0)
string(REPLACE ";" " " GTHREAD2_CFLAGS "${GTHREAD2_CFLAGS}")
pkg_check_modules(JSON_GLIB REQUIRED json-glib-1.0>=0.12)
string(REPLACE ";" " " JSON_GLIB_CFLAGS "${JSON_GLIB_CFLAGS}")

# Check for optional libraries
pkg_check_modules(AO ao)
string(REPLACE ";" " " AO_CFLAGS "${AO_CFLAGS}")
pkg_check_modules(DBUS dbus-glib-1)
string(REPLACE ";" " " DBUS_CFLAGS "${DBUS_CFLAGS}")
pkg_check_modules(GIO_UNIX gio-unix-2.0)
string(REPLACE ";" " " GIO_UNIX_CFLAGS "${GIO_UNIX_CFLAGS}")
pkg_check_modules(NOTIFY libnotify)
string(REPLACE ";" " " NOTIFY_CFLAGS "${NOTIFY_CFLAGS}")
pkg_check_modules(SOUP libsoup-2.4)
string(REPLACE ";" " " SOUP_CFLAGS "${SOUP_CFLAGS}")
pkg_check_modules(SOX sox)
string(REPLACE ";" " " SOX_CFLAGS "${SOX_CFLAGS}")

# spop daemon
set(SPOPD
  src/appkey.c
  src/commands.c
  src/config.c
  src/interface.c
  src/main.c
  src/plugin.c
  src/queue.c
  src/sd-daemon.c
  src/spotify.c
  src/utils.c
)
add_executable(spopd ${SPOPD})

set_target_properties(spopd PROPERTIES
  COMPILE_FLAGS "${SPOTIFY_CFLAGS} ${GLIB2_CFLAGS} ${GMODULE2_CFLAGS} ${GTHREAD2_CFLAGS} ${JSON_GLIB_CFLAGS}"
)
target_link_libraries(spopd dl rt ${SPOTIFY_LIBRARIES} ${GLIB2_LIBRARIES}
  ${GMODULE2_LIBRARIES} ${GTHREAD2_LIBRARIES} ${JSON_GLIB_LIBRARIES})
set(targets ${targets} spopd)

# Audio plugin: OSS
set(AUDIO_OSS
  plugins/oss.c
)
add_library(spop_audio_oss MODULE ${AUDIO_OSS})
set_target_properties(spop_audio_oss PROPERTIES
  COMPILE_FLAGS "${GLIB2_CFLAGS}"
)
target_link_libraries(spop_audio_oss ${GLIB2_LIBRARIES})
set(targets ${targets} spop_audio_oss)

# Audio plugin: libao
if(AO_FOUND)
  set(AUDIO_AO
    plugins/ao.c
  )
  add_library(spop_audio_ao MODULE ${AUDIO_AO})
  set_target_properties(spop_audio_ao PROPERTIES
    COMPILE_FLAGS "${AO_CFLAGS} ${GTHREAD2_CFLAGS}"
  )
  target_link_libraries(spop_audio_ao ${AO_LIBRARIES} ${GTHREAD2_LIBRARIES})
  set(targets ${targets} spop_audio_ao)
endif(AO_FOUND)

# Audio plugin: sox
if(SOX_FOUND)
  set(AUDIO_SOX
    plugins/sox.c
  )
  add_library(spop_audio_sox MODULE ${AUDIO_SOX})
  set_target_properties(spop_audio_sox PROPERTIES
    COMPILE_FLAGS "${SOX_CFLAGS} ${GTHREAD2_CFLAGS}"
  )
  target_link_libraries(spop_audio_sox ${SOX_LIBRARIES} ${GTHREAD2_LIBRARIES})
  set(targets ${targets} spop_audio_sox)
endif(SOX_FOUND)

# Generic plugin: awesome
if(DBUS_FOUND)
  set(PLUGIN_AWESOME
    plugins/awesome.c
  )
  add_library(spop_plugin_awesome MODULE ${PLUGIN_AWESOME})
  set_target_properties(spop_plugin_awesome PROPERTIES
    COMPILE_FLAGS "${DBUS_CFLAGS} ${GLIB2_CFLAGS}"
  )
  target_link_libraries(spop_plugin_awesome ${DBUS_LIBRARIES} ${GLIB2_LIBRARIES})
  set(targets ${targets} spop_plugin_awesome)
  install(DIRECTORY awesome DESTINATION share/spop)
endif(DBUS_FOUND)

# Generic plugin: mpris2
if(GIO_UNIX_FOUND)
  macro(GDBUS_CODEGEN DIR XML IFACE_PREFIX CCODE NAMESPACE)
    add_custom_command(
      OUTPUT "${DIR}/${CCODE}.c"
      OUTPUT "${DIR}/${CCODE}.h"
      COMMAND gdbus-codegen --interface-prefix="${IFACE_PREFIX}" --generate-c-code="${CCODE}" --c-namespace="${NAMESPACE}" "${XML}"
      WORKING_DIRECTORY "${DIR}"
      MAIN_DEPENDENCY "${DIR}/${XML}"
    )
  endmacro(GDBUS_CODEGEN)
  gdbus_codegen("${CMAKE_CURRENT_SOURCE_DIR}/plugins/mpris2" "mpris2.xml"
                org.mpris.MediaPlayer2 mpris2-generated Mpris2)
  gdbus_codegen("${CMAKE_CURRENT_SOURCE_DIR}/plugins/mpris2" "mpris2-player.xml"
                org.mpris.MediaPlayer2 mpris2-player-generated Mpris2)

  set(PLUGIN_MPRIS2
    plugins/mpris2/mpris2-generated.c
    plugins/mpris2/mpris2-player-generated.c
    plugins/mpris2/plugin.c
    plugins/mpris2/spop-mpris2.c
  )
  add_library(spop_plugin_mpris2 MODULE ${PLUGIN_MPRIS2})
  set_target_properties(spop_plugin_mpris2 PROPERTIES
    COMPILE_FLAGS "${GIO_UNIX_CFLAGS} ${GLIB2_CFLAGS}"
  )
  target_link_libraries(spop_plugin_mpris2 ${GIO_UNIX_LIBRARIES} ${GLIB2_LIBRARIES})
  set(targets ${targets} spop_plugin_mpris2)
endif(GIO_UNIX_FOUND)

# Generic plugin: notify
if(NOTIFY_FOUND)
  set(PLUGIN_NOTIFY
    plugins/notify.c
  )
  add_library(spop_plugin_notify MODULE ${PLUGIN_NOTIFY})
  set_target_properties(spop_plugin_notify PROPERTIES
    COMPILE_FLAGS "${GLIB2_CFLAGS} ${NOTIFY_CFLAGS}"
  )
  target_link_libraries(spop_plugin_notify ${GLIB2_LIBRARIES} ${NOTIFY_LIBRARIES})
  set(targets ${targets} spop_plugin_notify)
endif(NOTIFY_FOUND)

# Generic plugin: savestate
set(PLUGIN_SAVESTATE
  plugins/savestate.c
)
add_library(spop_plugin_savestate MODULE ${PLUGIN_SAVESTATE})
set_target_properties(spop_plugin_savestate PROPERTIES
  COMPILE_FLAGS "${GLIB2_CFLAGS} ${JSON_GLIB_CFLAGS}"
)
target_link_libraries(spop_plugin_savestate ${GLIB2_LIBRARIES} ${JSON_GLIB_LIBRARIES})
set(targets ${targets} spop_plugin_savestate)

# Generic plugin: scrobble
if(SOUP_FOUND)
  set(PLUGIN_SCROBBLE
    plugins/scrobble.c
  )
  add_library(spop_plugin_scrobble MODULE ${PLUGIN_SCROBBLE})
  set_target_properties(spop_plugin_scrobble PROPERTIES
    COMPILE_FLAGS "${GLIB2_CFLAGS} ${SOUP_CFLAGS}"
  )
  target_link_libraries(spop_plugin_scrobble ${GLIB2_LIBRARIES} ${SOUP_LIBRARIES})
  set(targets ${targets} spop_plugin_scrobble)
endif(SOUP_FOUND)

# dspop client
install(PROGRAMS dspop/dspop DESTINATION bin)

# systemd service files
install(DIRECTORY systemd DESTINATION lib/systemd/system)

# Common to all source files
set(SRC
  ${SPOPD}
  ${AUDIO_OSS}
  ${AUDIO_AO}
  ${AUDIO_SOX}
  ${PLUGIN_AWESOME}
  ${PLUGIN_NOTIFY}
  ${PLUGIN_SAVESTATE}
  ${PLUGIN_SCROBBLE}
)
set_source_files_properties(${SRC}
  COMPILE_FLAGS "-O2 -Wall -Werror"
)
include_directories(src)

# System-wide installation
install(TARGETS ${targets}
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
)
