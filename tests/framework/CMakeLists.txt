# **********************************************************
# Copyright (c) 2012-2021 Google, Inc.  All rights reserved.
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

# XXX i#1652: make this a separate project and use --build-and-test

cmake_minimum_required(VERSION 3.7)

include(${PROJECT_SOURCE_DIR}/make/policies.cmake NO_POLICY_SCOPE)

# Always build Debug
set(CMAKE_BUILD_TYPE "Debug")

set_output_dirs("${PROJECT_BINARY_DIR}/tests")

find_package(DrMemoryFramework PATHS ${framework_dir})

# As we'll be calling configure_DynamoRIO_{client,standalone} from within
# a function scope, we must set the global vars ahead of time:
configure_DynamoRIO_global(OFF ON)

# We save pdb+dll space by not using libc (xref DRi#714).
# For Android, we allow running w/o a private Bionic.
# For Linux x86 we need libc for div+mod routines, as well
# as for Umbra.
if (WIN32 OR ANDROID)
  set(DynamoRIO_USE_LIBC OFF)
endif ()

function(add_drmf_test_app app_name src_app)
  tobuild(${app_name} ${src_app})
  if (WIN32)
    # i#1748: disable inlining optimization to avoid calls to
    # fuzzing targets being inlined
    append_property_string(TARGET ${app_name} COMPILE_FLAGS "/Zi /Ob0")
  endif (WIN32)
  copy_target_to_device(${app_name})
endfunction(add_drmf_test_app)

# We only expect a few tests, so we simplify things by having a simple
# regex output for whether they passed.
# ext_list should be a list of extensions minus the drmf_ prefix.
function(add_drmf_test test_name app_name src_client ext_list dr_options
    client_options pass_regex)
  set(client_name ${test_name}.client)
  add_library(${client_name} SHARED ${src_client})
  _DR_append_property_list(TARGET ${client_name} COMPILE_DEFINITIONS "${arch_defs}")
  # We rely on i#955's "rpath file" to locate the extension on Windows
  set(DynamoRIO_RPATH ON)
  configure_DynamoRIO_client(${client_name})

  foreach (ext ${ext_list})
    if ("${CMAKE_VERSION}" VERSION_EQUAL "3.0" OR
        "${CMAKE_VERSION}" VERSION_GREATER "3.0")
      use_DynamoRIO_extension(${client_name} ${ext})
    else ()
      use_DynamoRIO_extension(${client_name} drmf_${ext}) # drmf_ prefix b/c internal
    endif ()
    use_DynamoRIO_extension(${client_name} drmgr)
    # We use the namespace prefix to allow us to avoid target conflicts
    # (else we'd need a completely separate build for these tests),
    # but cmake gets confused and we have to tell it the two are the same.
    add_dependencies(${client_name} ${ext})
  endforeach (ext)

  copy_target_to_device(${client_name})
  copy_and_adjust_drpaths(${CMAKE_RUNTIME_OUTPUT_DIRECTORY} ${client_name})

  get_target_path_for_execution(app_path ${app_name})
  get_target_path_for_execution(client_path ${client_name})
  if (DEBUG_BUILD)
    set(drrun_extra -quiet -debug)
  else ()
    set(drrun_extra -quiet)
  endif ()
  set(drrun_path ${DynamoRIO_DIR}/../${BIN_ARCH}/drrun)
  convert_local_path_to_device_path(drrun_path ${drrun_path})
  prefix_cmd_if_necessary(drrun_path OFF ${drrun_path})
  if (NOT ANDROID) # FIXME i#1860: fix for Android
    add_test(${test_name}
      ${drrun_path} ${drrun_extra}
      -client ${client_path} 0 "${client_options}"
      -msgbox_mask 0 -stderr_mask 0xc ${dr_options}
      "--" ${app_path})
    set_tests_properties(${test_name} PROPERTIES PASS_REGULAR_EXPRESSION "${pass_regex}")
  endif ()
endfunction(add_drmf_test)

# We only expect a few tests, so we simplify things by having a simple
# regex output for whether they passed.
# ext_list should be a list of extensions minus the drmf_ prefix.
function(add_drmf_additional_test test_name app_name client_name dr_options
    client_options pass_regex)
  get_target_path_for_execution(app_path ${app_name})
  get_target_path_for_execution(client_path ${client_name})
  if (DEBUG_BUILD)
    set(drrun_extra -quiet -debug)
  else ()
    set(drrun_extra -quiet)
  endif ()
  set(drrun_path ${DynamoRIO_DIR}/../${BIN_ARCH}/drrun)
  convert_local_path_to_device_path(drrun_path ${drrun_path})
  prefix_cmd_if_necessary(drrun_path OFF ${drrun_path})
  if (NOT ANDROID) # FIXME i#1860: fix for Android
    add_test(${test_name}
      ${drrun_path} ${drrun_extra}
      -client ${client_path} 0 "${client_options}"
      -msgbox_mask 0 -stderr_mask 0xc ${dr_options}
      "--" ${app_path})
    set_tests_properties(${test_name} PROPERTIES PASS_REGULAR_EXPRESSION "${pass_regex}")
  endif ()
endfunction()

# i#2273: We need to locate a sysnum file.  Unfortunately creating one is only
# supported from a front end so we rely on a Dr. Memory test having created one.
# These will fail w/ direct-named test launches in a fresh dir but we can live with
# that for now: a suite run will run "hello" first.
# TODO: i#2279: Come up with a better method.
set(symcache_dir "${PROJECT_BINARY_DIR}/logs/symcache")

add_drmf_test_app(drsyscall_app drsyscall_app.c)
add_drmf_test(drmf_drsyscall_test drsyscall_app drsyscall_client.c
  drmf_drsyscall "" "${symcache_dir}" "done\nTEST PASSED\n$")
if (NOT ANDROID) # XXX i#1860: Android tests not enabled yet.
  set_property(TEST drmf_drsyscall_test APPEND PROPERTY DEPENDS hello)
endif ()
add_drmf_test(drmf_strace_test drsyscall_app strace_client.c
  drmf_drsyscall "" "${symcache_dir}" "done\n.*TEST PASSED\n$")
if (NOT ANDROID) # XXX i#1860: Android tests not enabled yet.
  set_property(TEST drmf_strace_test APPEND PROPERTY DEPENDS hello)
endif ()

# drfuzz tests
add_drmf_test_app(drfuzz_app_empty drfuzz_app_empty.c)
add_drmf_test(drfuzz_test_empty drfuzz_app_empty drfuzz_client_empty.c
  drfuzz "" "" "done\nTEST PASSED\n$")
add_drmf_test(drfuzz_test_mutator drfuzz_app_empty drfuzz_client_mutator.c
  drfuzz "" "" "TEST PASSED\n.*done")

add_drmf_test_app(drfuzz_app_repeat drfuzz_app_repeat.c)
add_drmf_test(drfuzz_test_repeat drfuzz_app_repeat drfuzz_client_repeat.c
  drfuzz "" "" "hello 1\nhello 2\nhello 3\nhello 4\nhello 5\ndone\nTEST PASSED\n$")

add_drmf_test_app(drfuzz_app_segfault drfuzz_app_segfault.c)

set(segfault_regex_1 "Fault occured in target.*with 1 args.*")
set(segfault_regex_2 "Crash originated in target.*with 1 args.*")
set(segfault_regex_3 "TEST PASSED\n$")

add_drmf_test(drfuzz_test_segfault drfuzz_app_segfault drfuzz_client_segfault.c
  drfuzz "" "-crash" "${segfault_regex_1}${segfault_regex_2}${segfault_regex_3}")
use_DynamoRIO_extension(drfuzz_test_segfault.client drsyms)

add_drmf_test(drfuzz_test_app_abort drfuzz_app_segfault drfuzz_client_segfault.c
  drfuzz "" "-abort" "${segfault_regex_2}${segfault_regex_3}")
use_DynamoRIO_extension(drfuzz_test_app_abort.client drsyms)

add_drmf_test(drfuzz_test_no_crash drfuzz_app_segfault drfuzz_client_segfault.c
  drfuzz "" "" ".*TEST PASSED\n$")
use_DynamoRIO_extension(drfuzz_test_no_crash.client drsyms)

# XXX i#1734: add multi-threaded test, checking for aborted fuzz targets on all threads

# umbra tests

set(asm_defs "${asm_defs}" -I "${CMAKE_CURRENT_SOURCE_DIR}") # for umbra_test_shared.h
add_drmf_test_app(umbra_app umbra_app.c)
if (WIN32)
  # Try to test i#2184 where Control Flow Guard allocates a 2TB reservation
  # crossing multiple Umbra segments.  I can only repro that with VS2017, but
  # we put this in place in anticipation of updating to VS2017.
  append_property_string(TARGET umbra_app COMPILE_FLAGS "/guard:cf")
endif ()
target_include_directories(umbra_app PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})

add_drmf_test(umbra_test_empty umbra_app umbra_client_empty.c
  umbra "" "" ".*TEST PASSED\n$")

if (X64)
  # Test i#1712 with a region overlapping 0x7e00'-0x7f00'.
  add_drmf_test(umbra_test_overlap umbra_app umbra_client_empty.c
    umbra "-vm_base;0x7efff0000000;-no_vm_base_near_app" "" ".*TEST PASSED\n$")
endif ()

add_drmf_test(umbra_test_shadow_mem umbra_app umbra_client_shadow_mem.c
  umbra "" "" ".*TEST PASSED\n$")
use_DynamoRIO_extension(umbra_test_shadow_mem.client drreg)
use_DynamoRIO_extension(umbra_test_shadow_mem.client drutil)
target_include_directories(umbra_test_shadow_mem.client PRIVATE
  ${CMAKE_CURRENT_SOURCE_DIR})

add_drmf_test(umbra_test_insert_app_to_shadow umbra_app
  umbra_client_insert_app_to_shadow.c umbra "" "" ".*TEST PASSED\n$")
use_DynamoRIO_extension(umbra_test_insert_app_to_shadow.client drreg)
use_DynamoRIO_extension(umbra_test_insert_app_to_shadow.client drutil)
target_include_directories(umbra_test_insert_app_to_shadow.client PRIVATE
  ${CMAKE_CURRENT_SOURCE_DIR})

add_drmf_test_app(umbra_wild umbra_wild.c)
add_drmf_additional_test(umbra_wild_shadow umbra_wild
  umbra_test_insert_app_to_shadow.client "" "" ".*TEST PASSED\n$")

# Faulty redzones are only available on 32-bit.
if (NOT X64)
  add_drmf_test(umbra_client_faulty_redzone umbra_app
    umbra_client_faulty_redzone.c umbra "" "" ".*TEST PASSED\n$")
  use_DynamoRIO_extension(umbra_client_faulty_redzone.client drreg)
  use_DynamoRIO_extension(umbra_client_faulty_redzone.client drutil)
  target_include_directories(umbra_client_faulty_redzone.client PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR})
  # XXX i#2341: Historically, umbra_client_faulty_redzone was flaky and hung.
  # We left this timeout here just in case we encounter the issue again.
  if (NOT ANDROID) # TODO i#1860: enable tests for Android
    set_tests_properties(umbra_client_faulty_redzone PROPERTIES TIMEOUT 60)
  endif ()
endif()

add_drmf_test(umbra_test_consistency umbra_app
  umbra_client_consistency.c umbra "" "" ".*TEST PASSED\n$")
use_DynamoRIO_extension(umbra_test_consistency.client drreg)
use_DynamoRIO_extension(umbra_test_consistency.client drutil)
target_include_directories(umbra_test_consistency.client PRIVATE
  ${CMAKE_CURRENT_SOURCE_DIR})

add_drmf_test(umbra_test_allscales umbra_app umbra_client_allscales.c
  umbra "" "" ".*TEST PASSED\n$")
