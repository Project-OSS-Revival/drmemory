# **********************************************************
# Copyright (c) 2013-2020 Google, Inc.  All rights reserved.
# **********************************************************

# Dr. Memory: the memory debugger
#
# This library is free software; you can redistribute it and/or
# modify it under the terms of the GNU Lesser General Public
# License as published by the Free Software Foundation;
# version 2.1 of the License, and no later version.
#
# This library is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
# Library General Public License for more details.
#
# You should have received a copy of the GNU Lesser General Public
# License along with this library; if not, write to the Free Software
# Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.

cmake_minimum_required(VERSION 3.7)

include(${PROJECT_SOURCE_DIR}/make/policies.cmake NO_POLICY_SCOPE)

find_package(DrMemoryFramework PATHS ${framework_dir})

##################################################
# drstracelib

set(srcs
    drstrace.c
    drstrace_named_consts.c)

if (WIN32)
  set(srcs ${srcs} ${PROJECT_SOURCE_DIR}/make/resources.rc)
endif ()

# We set DR libc option on b/c this is needed to link with drsyms_static
# on VS2010. It has a downside of increase in pdb and dll size (xref DRi#714).
set(DynamoRIO_USE_LIBC ON)

add_library(drstracelib SHARED ${srcs})

# We share the framework version # for now
set_library_version(drstracelib ${DRMF_VERSION_MAJOR_MINOR})

if (WIN32)
  _DR_append_property_list(TARGET drstracelib COMPILE_DEFINITIONS
    "${DEFINES_NO_D};RC_IS_DRSTRACELIB")
else ()
  _DR_append_property_list(TARGET drstracelib COMPILE_DEFINITIONS "${DEFINES_NO_D}")
endif ()

set_target_properties(drstracelib PROPERTIES
  RUNTIME_OUTPUT_DIRECTORY${location_suffix} "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}")

# We rely on i#955's "rpath file" to locate the extension on Windows, for
# the build dir at least.
set(DynamoRIO_RPATH ON)
configure_DynamoRIO_client(drstracelib)

# We could link with the internal static lib but it's a good exercise
# to test the public version:
# XXX: We could use a dynamic drsyms, although drmemory is using static.
# So we'd need special deployment here.
use_DynamoRIO_extension(drstracelib drsyms_static)
# While we seem to need the drmf_ prefix here for shared drsyscall, for static
# that doesn't work while this does -- go figure (and we can't use drmf_ w/
# cmake 3.+ anyway: i#1652):
use_DynamoRIO_extension(drstracelib drmf_drsyscall_static)
use_DynamoRIO_extension(drstracelib drmgr_static)
use_DynamoRIO_extension(drstracelib drx_static)
add_dependencies(drstracelib drmf_drsyscall)

if (WIN32)
  # i#1805: to avoid strtol dup symbol errors (libcmt vs ntdll_imports) we just
  # force the link.  Alternatives include splitting ntdll_imports into Nt+Rtl vs
  # libc, dropping support for cmake < 3, or dropping non-Ninja support.
  # We also do this for Ninja generators with VS2017 on Appveyor where we hit
  # _isdigit dup symbol errors (libucrt vs ntdll_imports).
  append_property_string(TARGET drstracelib LINK_FLAGS "/force:multiple")
endif ()

install(TARGETS drstracelib
  RUNTIME DESTINATION "${INSTALL_LIB}" # dll
  LIBRARY DESTINATION "${INSTALL_LIB}" # .so
  PERMISSIONS ${owner_access} OWNER_EXECUTE GROUP_READ GROUP_EXECUTE
  WORLD_READ WORLD_EXECUTE)

##################################################
# symbol_fetch to fetch symbols from MS Symbol Server

# We only need to fetch the Wintypes.pdb but the Wintypes.dll is only available
# on Windows 8 or later. So we make a special fake dll here and then change
# GUID & name to Wintypes.pdb in the debug section of the new dll. We use a special
# mksymguid.pl script to do that.
# Note: Fake dll name should be the same length as Wintypes.pdb to avoid PE debug
# section realignment.

if (WIN32)
  add_library(symfetch SHARED symfetch.c ${PROJECT_SOURCE_DIR}/make/resources.rc)

  # We manually set PDBALTPATH here b/c compiler sets absolute path
  # to pdb which we don't need here. We disable INCREMENTAL b/c we
  # don't want to produce additional files with dll. We statically
  # set image base to avoid bug on x64 environment when symbols
  # are fetched but couldn't be loaded by using SymLoadModuleExW.
  append_property_string(TARGET symfetch LINK_FLAGS
                         "/BASE:0x10000000 /INCREMENTAL:NO /PDBALTPATH:symfetch.pdb")

  set_property(TARGET symfetch PROPERTY COMPILE_DEFINITIONS
               "${DEFINES_NO_D};RC_IS_DRSTRACE_SYMFETCH_LIB")
  # locate perl
  if (PERL_PATH AND EXISTS "${PERL_PATH}/perl.exe")
    set(PERL_EXECUTABLE "${PERL_PATH}/perl.exe")
  else ()
    include(FindPerl)
    if (PERL_FOUND)
      get_filename_component(PERL_PATH "${PERL_EXECUTABLE}" PATH)
    endif (PERL_FOUND)
  endif ()
  add_custom_command(TARGET symfetch POST_BUILD COMMAND ${PERL_EXECUTABLE}
                     "${CMAKE_CURRENT_SOURCE_DIR}/mksymguid.pl"
                     "$<TARGET_FILE:symfetch>" VERBATIM)
  install(TARGETS symfetch DESTINATION "${INSTALL_BIN}"
    PERMISSIONS ${owner_access} OWNER_EXECUTE GROUP_READ GROUP_EXECUTE
    WORLD_READ WORLD_EXECUTE)
endif (WIN32)

##################################################
# drstrace frontend

# We don't restore global flags (from each ${SAVE_${var}}) b/c cmake
# waits until the end of the file to take the global values into
# effect.  We don't seem to need them for the front-end though.

set(front_srcs drstrace_frontend.c)
if (WIN32)
  set(front_srcs ${front_srcs} ${PROJECT_SOURCE_DIR}/make/resources.rc)
endif ()
add_executable(drstrace ${front_srcs})
configure_DynamoRIO_standalone(drstrace)
target_link_libraries(drstrace drinjectlib drconfiglib drfrontendlib drmf_drsyscall_static)
set_library_version(drstrace ${DRMF_VERSION})
if (WIN32)
  _DR_append_property_list(TARGET drstrace COMPILE_DEFINITIONS
    "${DEFINES_NO_D};RC_IS_DRSTRACE;SYMBOL_DLL_NAME=\"$<TARGET_FILE_NAME:symfetch>\"")
  # DRi#1409/DRi#1503/i#1805: Avoid dup symbol _isdigit with VS2017.
  append_property_string(TARGET drstrace LINK_FLAGS "/force:multiple")
else ()
  _DR_append_property_list(TARGET drstrace COMPILE_DEFINITIONS "${DEFINES_NO_D}")
endif ()

install(TARGETS drstrace DESTINATION "${INSTALL_BIN}"
  PERMISSIONS ${owner_access} OWNER_EXECUTE GROUP_READ GROUP_EXECUTE
  WORLD_READ WORLD_EXECUTE)

##################################################
# drstrace unit tests build

if (WIN32)
  add_executable(drstrace_unit_tests ${srcs} drstrace_unit_tests.c)

  set_property(TARGET drstrace_unit_tests PROPERTY COMPILE_DEFINITIONS
    # We pass the absolute build dir path here as unit tests never move
    ${DEFINES_NO_D} DRSTRACE_UNIT_TESTS RC_IS_DRSTRACE_UNIT_TESTS
    SYMBOL_DLL_PATH="${symfetch_path}")
  configure_DynamoRIO_standalone(drstrace_unit_tests)
  use_DynamoRIO_extension(drstrace_unit_tests drmf_drsyscall_static)
  use_DynamoRIO_extension(drstrace_unit_tests drmgr_static)
  use_DynamoRIO_extension(drstrace_unit_tests drx_static)
  use_DynamoRIO_extension(drstrace_unit_tests drsyms_static)
  # XXX DRi#1409/DRi#1503/i#1805: we get link errors about dup symbols
  # dr_get_stderr_file() in drdecode and dynamorio libs unless we link w/
  # drinjectlib, and if we link w/ drinjectlib we get duplicate IR link errors
  # w/ drdecode (i#1805).  We used to solve by moving drfrontendlib earlier via
  # LINK_FLAGS, but generator expressions are not honored there, and we'd need
  # CMake 3.13 for LINK_OPTIONS.  We instead solve by /force:multiple..
  target_link_libraries(drstrace_unit_tests drfrontendlib)
  add_test(drstrace_unit_tests
           "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/drstrace_unit_tests")
  # We increase test timeout in case of slow Internet connection, esp now
  # that we try multiple times (i#1925).
  set_tests_properties(drstrace_unit_tests PROPERTIES TIMEOUT 240)

  if (WIN32)
    # i#1805: avoid tolower dup symbol errors.  For VS2017 avoid _isdigit.
    # Also avoid drfrontendlib duplicate IR link errors as documented above
    # DRi#1409/DRi#1503/i#1805.
    append_property_string(TARGET drstrace_unit_tests LINK_FLAGS "/force:multiple")
  endif ()
endif (WIN32)

##################################################
# drstrace test

if (BUILD_TOOL_TESTS)
  add_test(NAME drstrace COMMAND $<TARGET_FILE:drstrace> -dr ${DynamoRIO_DIR}/.. --
    $<TARGET_FILE:drsyscall_app>)
endif (BUILD_TOOL_TESTS)

##################################################
# tool config files for running with "drrun -t"

create_drrun_file(drstrace unused "drstrace")
