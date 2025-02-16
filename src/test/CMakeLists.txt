# SPDX-FileCopyrightText: Allen Winter <winter@kde.org>
# SPDX-License-Identifier: LGPL-2.1-only OR MPL-2.0

include_directories(
  ${CMAKE_BINARY_DIR}
  ${CMAKE_SOURCE_DIR}/src
  ${CMAKE_BINARY_DIR}/src
  ${CMAKE_SOURCE_DIR}/src/libical
  ${CMAKE_BINARY_DIR}/src/libical
  ${CMAKE_SOURCE_DIR}/src/libicalss
)

if(ICU_FOUND)
  include_directories(${ICU_INCLUDE_DIRS})
endif()

if(BDB_FOUND)
  include_directories(${BDB_INCLUDE_DIR})
endif()

set(EXECUTABLE_OUTPUT_PATH ${CMAKE_CURRENT_BINARY_DIR})

set(TEST_DATADIR "\"${CMAKE_SOURCE_DIR}/test-data\"")
add_definitions(-DTEST_DATADIR=${TEST_DATADIR} -DTEST_ZONEDIR="${CMAKE_SOURCE_DIR}/zoneinfo")

# Set properties for the test
macro(setprops _name)
  if(UNIX)
    set_tests_properties(${_name} PROPERTIES
      WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
    )
  endif()
  if(WIN32)
    set(tpath "${CMAKE_BINARY_DIR}/bin;%PATH%")
    if(ICU_FOUND)
      set(tpath "${ICU_BINARY_DIR};${tpath}")
    endif()
    set_tests_properties(${_name} PROPERTIES
      ENVIRONMENT "PATH=${tpath}"
      WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
    )
  endif()
  if(CYGWIN OR MSYS)
    set_tests_properties(${_name} PROPERTIES
      ENVIRONMENT "PATH=${EXECUTABLE_OUTPUT_PATH}:${CMAKE_BINARY_DIR}/lib:$ENV{PATH}"
      WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/lib"
    )
  endif()
endmacro()

#macro to build (but do not create a test)
macro(buildme _name _srcs)
  add_executable(${_name} ${_srcs})
  add_dependencies(${_name} ical icalss icalvcal)
  if(NOT SHARED_ONLY)
    add_dependencies(${_name} ical-static icalss-static icalvcal-static)
  endif()
  target_link_libraries(${_name} ical icalss icalvcal)

  if(WITH_CXX_BINDINGS)
    add_dependencies(${_name} ical_cxx icalss_cxx)
    if(NOT SHARED_ONLY)
      add_dependencies(${_name} ical_cxx-static icalss_cxx-static)
    endif()
    target_link_libraries(${_name} ical_cxx icalss_cxx)
  endif()

  if(DEFINED CMAKE_THREAD_LIBS_INIT)
    target_link_libraries(${_name} ${CMAKE_THREAD_LIBS_INIT})
  endif()
  if(ICU_FOUND)
    target_link_libraries(${_name} ${ICU_LIBRARIES})
  endif()
  if(BDB_FOUND)
    target_link_libraries(${_name} ${BDB_LIBRARY})
  endif()
endmacro()

# Build the test and add the test, making sure the test properties are set
macro(testme _name _srcs)
  buildme(${_name} "${_srcs}")
  add_test(NAME ${_name} COMMAND ${_name})
  setprops(${_name})
endmacro()

########## libical-glib tests #########
if(ICAL_GLIB AND HAVE_INTROSPECTION)
  add_subdirectory(libical-glib)
endif()

########### next target ###############

set(copycluster_SRCS copycluster.c)
buildme(copycluster "${copycluster_SRCS}")

########### next target ###############

set(regression_SRCS
  regression.c
  regression.h
  regression-component.c
  regression-classify.c
  regression-utils.c
  regression-recur.c
  regression-storage.c
  test-malloc.c
  test-malloc.h
)
if(WITH_CXX_BINDINGS)
  list(APPEND regression_SRCS regression-cxx.cpp)
endif()
testme(regression "${regression_SRCS}")

########### next target ###############

set(parser_SRCS icaltestparser.c)
buildme(parser "${parser_SRCS}")

file(GLOB TEST_FILES ${CMAKE_SOURCE_DIR}/test-data/*.ics)
foreach(test_file ${TEST_FILES})
  get_filename_component(test_name ${test_file} NAME_WE)
  add_test(NAME parser-${test_name} COMMAND parser ${test_file})
  setprops(parser-${test_name})
endforeach()

########### next target ###############

if(NOT WIN32 AND NOT ANDROID)
  #stow is very Unix specific
  set(stow_SRCS stow.c)
  buildme(stow "${stow_SRCS}")
endif()

########### next target ###############

set(recur_SRCS recur.c)
testme(recur "${recur_SRCS}")

########### next target ###############

if(HAVE_GETOPT) #getopt is required
  set(icalrecur_test_SRCS icalrecur_test.c)
  buildme(icalrecur_test "${icalrecur_test_SRCS}")
endif()

########### next target ###############
if(CMAKE_USE_PTHREADS_INIT)
  set(icaltm_test_SRCS icaltm_test.c)
  testme(icaltm_test "${icaltm_test_SRCS}")
endif()

########### next target ###############

if(HAVE_GETOPT) #getopt is required
  set(testmime_SRCS testmime.c)
  testme(testmime "${testmime_SRCS}")
endif()

########### next target ###############

set(testvcal_SRCS testvcal.c)
testme(testvcal "${testvcal_SRCS}")

########### next target ###############

set(process_SRCS process.c)
testme(process "${process_SRCS}")

########### next target ###############

if(NOT USE_BUILTIN_TZDATA AND NOT CYGWIN) #seeing >15% error rate on Cygwin. why?
  set(timezones_SRCS timezones.c)
  testme(timezones "${timezones_SRCS}")
endif()

########### next target ###############

set(builtin_timezones_SRCS builtin_timezones.c)
testme(builtin_timezones "${builtin_timezones_SRCS}")

########### next target ###############

#cmake v3.14 compare_files can ignore line endings => needed on Windows
set(CMAKE_COMPARE_FILES_IGNORE_EOL FALSE)
if(NOT ${CMAKE_VERSION} VERSION_LESS 3.14)
  set(CMAKE_COMPARE_FILES_IGNORE_EOL TRUE)
endif()
if(NOT WIN32 OR CMAKE_COMPARE_FILES_IGNORE_EOL)
  #ignore_eol doesn't work on Cygwin. testing years greater than 2037 will fail
  if(NOT CYGWIN AND NOT USE_32BIT_TIME_T)
    set(icalrecurtest_SRCS icalrecur_test.c)
    add_executable(icalrecurtest ${icalrecurtest_SRCS})
    target_link_libraries(icalrecurtest ical icalss icalvcal)
    set(test_cmd "${CMAKE_BINARY_DIR}/src/test/icalrecurtest${CMAKE_EXECUTABLE_SUFFIX}")

    #test rscale capable rrules
    set(test_args "-r")
    if(HAVE_ICU_DANGI)
      set(reference_data "icalrecur_withicu_dangi_test.out")
    elseif(ICU_FOUND)
      set(reference_data "icalrecur_withicu_test.out")
    else()
      set(reference_data "icalrecur_withouticu_test.out")
    endif()
    add_test(NAME icalrecurtest-r
      COMMAND ${CMAKE_COMMAND}
      -D test_cmd=${test_cmd}
      -D test_args:string=${test_args}
      -D output_blessed=${CMAKE_SOURCE_DIR}/src/test/${reference_data}
      -D output_test=${CMAKE_BINARY_DIR}/bin/test.out
      -P ${CMAKE_SOURCE_DIR}/cmake/run_test.cmake
    )
    setprops(icalrecurtest-r)

    #test non-rscale rrules
    set(test_args "")
    set(reference_data "icalrecur_test.out")
    add_test(NAME icalrecurtest
      COMMAND ${CMAKE_COMMAND}
      -D test_cmd=${test_cmd}
      -D test_args:string=${test_args}
      -D output_blessed=${CMAKE_SOURCE_DIR}/src/test/${reference_data}
      -D output_test=${CMAKE_BINARY_DIR}/bin/test.out
      -P ${CMAKE_SOURCE_DIR}/cmake/run_test.cmake
    )
    setprops(icalrecurtest)
  endif()
endif()

########### next target ###############

set(icalparser_ctrl_test_SRCS icalparser_ctrl_test.c)
testme(parser_ctrl "${icalparser_ctrl_test_SRCS}")
