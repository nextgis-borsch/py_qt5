################################################################################
# Project:  Lib PyQt
# Purpose:  CMake build scripts
################################################################################
# Copyright (C) 2020-2022, NextGIS <info@nextgis.com>
#
# Permission is hereby granted, free of charge, to any person obtaining a
# copy of this software and associated documentation files (the "Software"),
# to deal in the Software without restriction, including without limitation
# the rights to use, copy, modify, merge, publish, distribute, sublicense,
# and/or sell copies of the Software, and to permit persons to whom the
# Software is furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included
# in all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
# OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
# THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
# FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
# DEALINGS IN THE SOFTWARE.
################################################################################
cmake_minimum_required(VERSION 2.8.12 FATAL_ERROR)

project(pyqt5)

if(NOT DEFINED PACKAGE_VENDOR)
    set(PACKAGE_VENDOR NextGIS)
endif()

if(NOT DEFINED PACKAGE_BUGREPORT)
    set(PACKAGE_BUGREPORT info@nextgis.com)
endif()

# some init settings
set(CMAKE_COLOR_MAKEFILE ON)
# set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

if(OSX_FRAMEWORK AND (BUILD_SHARED_LIBS OR BUILD_STATIC_LIBS))
  message(FATAL_ERROR "Only OSX_FRAMEWORK key or any or both BUILD_SHARED_LIBS
                       and BUILD_STATIC_LIBS keys are permitted")
endif()

if(OSX_FRAMEWORK)
    set(CMAKE_MACOSX_RPATH ON)
endif()

set(INSTALL_CMAKECONF_DIR CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)

find_program(PYTHON3_EXECUTABLE python3)
set(PYTHON_EXECUTABLE ${PYTHON3_EXECUTABLE})

find_package(PythonInterp 3)
if(PYTHONINTERP_FOUND)
    set(PYTHON3_EXECUTABLE ${PYTHON_EXECUTABLE})
    set(PYTHON3_VERSION_MAJOR ${PYTHON_VERSION_MAJOR})
    set(PYTHON3_VERSION_MINOR ${PYTHON_VERSION_MINOR})
else()
    message(FATAL_ERROR "Cannot find python 3")
endif()

set(PYTHON_EXECUTABLE ${PYTHON3_EXECUTABLE})
set(PYTHON_VERSION_MAJOR ${PYTHON3_VERSION_MAJOR})
set(PYTHON_VERSION_MINOR ${PYTHON3_VERSION_MINOR})

file(DOWNLOAD "https://pypi.python.org/pypi/${PROJECT_NAME}/json" "${CMAKE_CURRENT_BINARY_DIR}/meta.json")

include(util)
get_compiler_version(COMPILER)
# get version and download url
execute_process(COMMAND ${PYTHON_EXECUTABLE}
    ${CMAKE_SOURCE_DIR}/cmake/pypi_process.py ${PLUGIN} ${CMAKE_CURRENT_BINARY_DIR}/meta.json ${COMPILER} ${LATEST_AVAILABLE_VERSION}
    OUTPUT_VARIABLE PACKAGE_DETAILS
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

list(GET PACKAGE_DETAILS 0 DOWNLOAD_URL)
list(GET PACKAGE_DETAILS 1 VERSION)
list(GET PACKAGE_DETAILS 2 FULL_VERSION)
list(GET PACKAGE_DETAILS 3 PROJECT_CPACK_FILENAME)

report_version(${PROJECT_NAME} ${VERSION})

set(TMP_INSTALL_PREFIX "${CMAKE_CURRENT_BINARY_DIR}/inst")

set(CONFIGURE_ARGS --verbose --build-dir ${CMAKE_CURRENT_BINARY_DIR}/bld_tmp --target-dir ${TMP_INSTALL_PREFIX} --scripts-dir ${CMAKE_CURRENT_BINARY_DIR}/s_tmp --confirm-license --no-docstrings --qt-shared)

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} --debug)
endif()

# Download tarbal
set(OUT_DIR_NAME ${PROJECT_NAME}-${VERSION})
set(OUT_NAME "${OUT_DIR_NAME}.tar.gz")

if(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}")
    status_message("Download ${DOWNLOAD_URL}")
    file(DOWNLOAD "${DOWNLOAD_URL}" "${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}"
        SHOW_PROGRESS
    )
endif()

# Extract archive
if(NOT EXISTS ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME})
    status_message("Extract ${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}")
    execute_process(COMMAND ${CMAKE_COMMAND} -E tar xzf ${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME})
endif()

include(FindAnyProject)
# find_anyproject(ZLIB REQUIRED)
find_anyproject(Qt5 REQUIRED COMPONENTS Core)
# Create qt.conf
get_property(QT_QMAKE_EXECUTABLE TARGET Qt5::qmake PROPERTY IMPORTED_LOCATION)
get_filename_component(QT_BIN_DIR "${QT_QMAKE_EXECUTABLE}" PATH)
get_filename_component(QT_ROOT_DIR "${QT_BIN_DIR}" PATH)
# file(WRITE ${QT_BIN_DIR}/qt.conf "[Paths]\nPrefix = ${QT_ROOT_DIR}")
set(CONFIGURE_ARGS ${CONFIGURE_ARGS} --qmake=${QT_QMAKE_EXECUTABLE})
set(CONFIGURE_ARGS ${CONFIGURE_ARGS} --qmake-setting "QMAKE_RPATHDIR+=${QT_ROOT_DIR}/lib")

include(ProcessorCount)
ProcessorCount(N)
if(NOT N EQUAL 0)
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} --jobs ${N})
endif()

set(PYQT_INPUT_NAME ${OUT_DIR_NAME})

# Copy dependencies to bin dir
set(RUN_DEPENDENCIES
    lib_png
    lib_tiff
    lib_jpeg
    lib_jbig
    lib_lzma
    lib_webp
    lib_openssl
    lib_curl
    lib_jsonc
    lib_bzip2
    lib_qhull
    lib_proj
    lib_pq
    lib_oci
    lib_geotiff
    lib_gif
    lib_hdf4
    lib_openjpeg
    lib_kml
    lib_opencad
    lib_xml2
    lib_spatialite
    lib_szip
    lib_uriparser
    lib_z
)
get_compiler_version(COMPILER)

foreach(RUN_DEPENDENCY ${RUN_DEPENDENCIES})
    execute_process(
        COMMAND ${CMAKE_COMMAND} -DDST_PATH=${QT_ROOT_DIR}/lib -DSRC_PATH=${PROJECT_BINARY_DIR} -DREPKA_PACKAGE=${RUN_DEPENDENCY} -DCOMPILER=${COMPILER} -DOSX_FRAMEWORK=${OSX_FRAMEWORK} -P ${PROJECT_SOURCE_DIR}/cmake/get_dependency.cmake
    )
endforeach()

set(DISABLE_PYQT_COMPONENTS
    QtBluetooth
    QtLocation
    QtNfc
    QtPositioning
    QtSensors
    QtSerialPort
    QtTextToSpeech
)

foreach(_COMPONENT ${DISABLE_PYQT_COMPONENTS})
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} --disable ${_COMPONENT})
endforeach()
set(CONFIGURE_ARGS ${CONFIGURE_ARGS} --no-qml-plugin --no-designer-plugin)

# Find sip-install
execute_process(
    COMMAND ${PYTHON_EXECUTABLE} -m site --user-base
    RESULT_VARIABLE _process_status
    OUTPUT_VARIABLE _process_output
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

if(${_process_status} EQUAL 0)
    string(STRIP ${_process_output} BASE_PATH)
endif()

find_program(SIP_INSTALL_EXECUTABLE sip-install PATHS ${BASE_PATH}/bin)

message(STATUS "Configure args: ${CONFIGURE_ARGS}")

macro(build_py ver)
    add_custom_target(build_py${ver} ALL
        COMMAND ${SIP_INSTALL_EXECUTABLE} ${CONFIGURE_ARGS}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${PYQT_INPUT_NAME}
        VERBATIM
    )

    if(OSX_FRAMEWORK)
        add_custom_target(fix_py${ver} ALL
            COMMAND ${CONFIGURE_CMD}
            ${PYTHON${ver}_EXECUTABLE} ${CMAKE_SOURCE_DIR}/cmake/mac_rpath_fix.py
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
            DEPENDS build_py${ver}
            VERBATIM
        )
    endif()
endmacro()

if(PYTHON3_EXECUTABLE)
    build_py(3)
endif()

set(PACKAGE_NAME ${PROJECT_NAME})
string(TOUPPER ${PACKAGE_NAME} PACKAGE_UPPER_NAME)

set(PYQT_SCRIPTS_DIR ${CMAKE_CURRENT_BINARY_DIR}/pyqt5-scripts)
if(WIN32)
    set(INSTALL_PYUIC_PROGRAM bin/pyuic5.bat)
    set(INSTALL_PYRCC_PROGRAM bin/pyrcc5.bat)
    set(INSTALL_PYLUPDATE_PROGRAM bin/pylupdate5.bat)
else()
    set(PYQT_SCRIPTS 
        ${CMAKE_CURRENT_BINARY_DIR}/pyqt5-scripts/pyrcc5
        ${CMAKE_CURRENT_BINARY_DIR}/pyqt5-scripts/pyuic5
        ${CMAKE_CURRENT_BINARY_DIR}/pyqt5-scripts/pylupdate5
    )
    set(INSTALL_PYUIC_PROGRAM bin/pyuic5)
    set(INSTALL_PYRCC_PROGRAM bin/pyrcc5)
    set(INSTALL_PYLUPDATE_PROGRAM bin/pylupdate5)

    file(MAKE_DIRECTORY ${PYQT_SCRIPTS_DIR})
    file(WRITE ${PYQT_SCRIPTS_DIR}/pyrcc5 "#!/bin/sh\nexec python3 -m PyQt5.pyrcc_main \${1+\"\$@\"}")
    file(WRITE ${PYQT_SCRIPTS_DIR}/pyuic5 "#!/bin/sh\nexec python3 -m PyQt5.uic.pyuic \${1+\"\$@\"}")
    file(WRITE ${PYQT_SCRIPTS_DIR}/pylupdate5 "#!/bin/sh\nexec python3 -m PyQt5.pylupdate_main \${1+\"\$@\"}")
endif()

if(OSX_FRAMEWORK)
    set(INSTALL_PYTHON_DIR "Library/Python/${PYTHON_VERSION_MAJOR}/site-packages/")
elseif(WIN32)
    set(INSTALL_PYTHON_DIR "${INSTALL_LIB_DIR}/Python${PYTHON_VERSION_MAJOR}/site-packages/")
else()
    set(INSTALL_PYTHON_DIR "lib/python${PYTHON_VERSION_MAJOR}/dist-packages/")
endif()

# set(INSTALL_SIP_DIR share/pyqt5) # ?

configure_file(cmake/PackageConfig.cmake.in
    ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake @ONLY)

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL )
    install(DIRECTORY ${TMP_INSTALL_PREFIX}/ 
        DESTINATION "${INSTALL_PYTHON_DIR}" USE_SOURCE_PERMISSIONS)
    
    install(FILES ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake
      DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)
    
    install(FILES ${PYQT_SCRIPTS}
        PERMISSIONS OWNER_EXECUTE OWNER_READ
        DESTINATION usr/bin
    )
endif()

# Archiving ====================================================================
set(PACKAGE_NAME ${PROJECT_NAME})

set(CPACK_PACKAGE_NAME "${PACKAGE_NAME}")
set(CPACK_PACKAGE_VENDOR "${PACKAGE_VENDOR}")
set(CPACK_PACKAGE_VERSION "${VERSION}")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${PACKAGE_NAME} Installation")
set(CPACK_PACKAGE_RELOCATABLE TRUE)
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
set(CPACK_GENERATOR "ZIP")
set(CPACK_MONOLITHIC_INSTALL ON)
set(CPACK_STRIP_FILES TRUE)

# Get cpack zip archive name
get_cpack_filename(${VERSION} PROJECT_CPACK_FILENAME)
set(CPACK_PACKAGE_FILE_NAME ${PROJECT_CPACK_FILENAME})

include(CPack)
