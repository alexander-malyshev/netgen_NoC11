if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "RelWithDebInfo" CACHE STRING INTERNAL)
endif(NOT CMAKE_BUILD_TYPE)

cmake_minimum_required(VERSION 2.8.12)
if(NOT WIN32)
    option( USE_NATIVE_ARCH  "build which -march=native" ON)
endif(NOT WIN32)

option( USE_GUI     "don't build netgen with GUI" ON )
option( USE_PYTHON  "build with python interface" ON )
option( USE_MPI     "enable mpi parallelization" OFF )
option( USE_OCC     "(not supported) compile with OpenCascade geometry kernel" OFF)
option( USE_JPEG    "enable snapshots using library libjpeg" OFF )
option( USE_MPEG    "enable video recording with FFmpeg, uses libavcodec" OFF )
option( INTEL_MIC        "cross compile for intel xeon phi")
option( INSTALL_PROFILES "install environment variable settings to /etc/profile.d" OFF )
option( INSTALL_DEPENDENCIES "install dependencies like tcl, useful for packaging" OFF )
option( USE_CCACHE       "use ccache")

option( USE_SUPERBUILD       "use ccache" ON)

if (USE_SUPERBUILD)
  project (SUPERBUILD)
  # execute the superbuild (this script will be invoked again without the
  # USE_SUPERBUILD option this time)
  include (cmake_modules/SuperBuild.cmake)
  return()                      # stop processing this file further
else()
  project(Netgen)
endif()

set(NETGEN_VERSION_MAJOR 6)
set(NETGEN_VERSION_MINOR 2)
string(TIMESTAMP NETGEN_VERSION_PATCH "%y%U%w" )
set(NETGEN_VERSION "${NETGEN_VERSION_MAJOR}.${NETGEN_VERSION_MINOR}-dev")
set(PACKAGE_VERSION "${NETGEN_VERSION_MAJOR}.${NETGEN_VERSION_MINOR}-${NETGEN_VERSION_PATCH}")
set(CPACK_PACKAGE_VERSION "${PACKAGE_VERSION}")
# set(CPACK_DEBIAN_PACKAGE_VERSION "${PACKAGE_VERSION}"


#######################################################################
if(USE_CCACHE)
  find_program(CCACHE_FOUND ccache)
  if(CCACHE_FOUND)
    set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
    set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ccache)
  endif(CCACHE_FOUND)
endif(USE_CCACHE)

#######################################################################
if(USE_NATIVE_ARCH)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native")
endif(USE_NATIVE_ARCH)

#######################################################################
if(INTEL_MIC)
    set(MKL_ARCH "mic")
    include(cmake_modules/mic.cmake)
else(INTEL_MIC)
    set(MKL_ARCH "intel64")
endif(INTEL_MIC)

#######################################################################
# Append install paths of software in non-standard paths (e.g. openmpi, metis, intel mkl, ...)
# cmake -DUSE_MPI=ON -DCMAKE_PREFIX_PATH="/opt/openmpi165;/opt/metis51" ../
set(ADDITIONAL_PATHS "" CACHE PATH "List of paths to additional libraries in non-standard locations, seperated by ';'")
if (ADDITIONAL_PATHS)
  set(CMAKE_PREFIX_PATH ${ADDITIONAL_PATHS})
endif (ADDITIONAL_PATHS)

if(APPLE)
  set(CMAKE_INSTALL_PREFIX "${INSTALL_DIR}/Netgen.app/Contents/Resources" CACHE INTERNAL "Prefix prepended to install directories" FORCE)
else(APPLE)
  set(CMAKE_INSTALL_PREFIX "${INSTALL_DIR}" CACHE INTERNAL "Prefix prepended to install directories" FORCE) 
endif(APPLE)

#######################################################################
# use rpath
SET(CMAKE_SKIP_BUILD_RPATH  FALSE)
SET(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE) 
SET(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
SET(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
LIST(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/lib" isSystemDir)
IF("${isSystemDir}" STREQUAL "-1")
   SET(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
ENDIF("${isSystemDir}" STREQUAL "-1")

#######################################################################
# build options
add_definitions(-DUSE_TIMEOFDAY)

include_directories ("${PROJECT_SOURCE_DIR}/include")
include_directories ("${PROJECT_SOURCE_DIR}/libsrc/include")
include_directories ("${PROJECT_BINARY_DIR}")

set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH}" "${PROJECT_SOURCE_DIR}/cmake_modules")

set(INCDIR include CACHE PATH "install directory for header files")
if(APPLE)
  set(ng_install_dir_bin ../MacOS)
  set(ng_install_dir_lib ../MacOS)
else(APPLE)
  set(ng_install_dir_bin bin)
  set(ng_install_dir_lib lib)
endif(APPLE)
set(ng_install_dir EXPORT netgen-targets RUNTIME DESTINATION ${ng_install_dir_bin} COMPONENT netgen LIBRARY DESTINATION ${ng_install_dir_lib} COMPONENT netgen_devel ARCHIVE DESTINATION ${ng_install_dir_lib} COMPONENT netgen_devel)
install(EXPORT netgen-targets DESTINATION share/cmake)

include (CheckIncludeFiles)
check_include_files (dlfcn.h HAVE_DLFCN_H)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/config.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/config.h)
add_definitions(-DHAVE_CONFIG_H)
include_directories(BEFORE ${CMAKE_CURRENT_BINARY_DIR})

#######################################################################
# platform specific definitions and flags
macro(get_WIN32_WINNT version)
    if (WIN32 AND CMAKE_SYSTEM_VERSION)
        if("${CMAKE_SYSTEM_VERSION}" MATCHES "^([0-9]+)\\.([0-9]+)")
            math(EXPR ver "${CMAKE_MATCH_1}*100 + ${CMAKE_MATCH_2}")
        endif()
        set(${version} "0x${ver}")
    endif()
endmacro()

macro(get_dll_from_lib dll_path lib_path)
        get_filename_component(parent_lib_path ${lib} DIRECTORY)
        get_filename_component(lib_name ${lib} name)
endmacro()

if(WIN32)
    get_WIN32_WINNT(ver)
    add_definitions(-D_WIN32_WINNT=${ver} -DWNT -DWNT_WINDOW -DNOMINMAX)
    set(CMAKE_MFC_FLAG 0)
#     add_definitions(-DNGINTERFACE_EXPORTS)
#     add_definitions(-DNGLIB_EXPORTS)
    add_definitions(-DMSVC_EXPRESS -D_CRT_SECURE_NO_WARNINGS -DHAVE_STRUCT_TIMESPEC)
    # build convenience (aka object) libraries in windows)
    set(NG_LIB_TYPE OBJECT)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP /wd4244 /wd4800")
else(WIN32)
    # build shared libraries
    set(NG_LIB_TYPE SHARED)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14")
endif(WIN32)
if(APPLE)
#    set(MACOSX_BUNDLE ON)
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -undefined dynamic_lookup")
#     set(CMAKE_SHARED_LIBRARY_SUFFIX ".so")
endif(APPLE)

#######################################################################
if(NOT ZLIB_INCLUDE_DIRS)
  find_package(ZLIB REQUIRED)
endif(NOT ZLIB_INCLUDE_DIRS)
include_directories(${ZLIB_INCLUDE_DIRS})

#######################################################################
if (USE_GUI)
    find_package(TCL 8.5 REQUIRED)
    find_package(Threads REQUIRED)
    if(APPLE)
          set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -framework AppKit")
    else(APPLE)
          find_package(X11 REQUIRED) 
    endif(APPLE)
    find_package(OpenGL REQUIRED)

    add_definitions(-DTCL -DOPENGL -DUSE_TOGL_2)
    include_directories(${TCL_INCLUDE_PATH})
    include_directories(${TK_INCLUDE_PATH})
    set(LIBTOGL togl)

    if(WIN32)
        add_definitions(-DTOGL_WGL)
    else(WIN32)
        if(APPLE)
                ADD_DEFINITIONS(-DTOGL_NSOPENGL)
        else(APPLE)
                ADD_DEFINITIONS(-DTOGL_X11)
        endif(APPLE)
    endif(WIN32)
endif (USE_GUI)

#######################################################################
if (USE_PYTHON)
    add_definitions(-DNG_PYTHON)

    include_directories(${PYBIND_INCLUDE_DIR})
    include_directories(${PYTHON_INCLUDE_DIRS})

    install(DIRECTORY ${PYBIND_INCLUDE_DIR}/pybind11 DESTINATION ${INCDIR} COMPONENT netgen_devel)
    install(FILES ${PYBIND_INCLUDE_DIR}/../LICENSE DESTINATION ${INCDIR}/pybind11 COMPONENT netgen_devel)
endif (USE_PYTHON)

#######################################################################
if (USE_MPI)
    find_package(MPI REQUIRED)
    find_package(METIS REQUIRED)
    add_definitions(-DPARALLEL -DMETIS)
    include_directories(${MPI_CXX_INCLUDE_PATH})
    include_directories(${METIS_INCLUDE_DIR})
endif (USE_MPI)

#######################################################################
if (USE_OCC)
    find_package(OpenCasCade REQUIRED)
    add_definitions(-DOCCGEOMETRY -D_OCC64)
    include_directories(${OCC_INCLUDE_DIR})
endif (USE_OCC)

#######################################################################
if (USE_JPEG)
    find_package(JPEG REQUIRED)
    add_definitions(-DJPEGLIB)
    include_directories(${JPEG_INCLUDE_DIR})
endif (USE_JPEG)

#######################################################################
if (USE_MPEG)
    find_package(FFMPEG REQUIRED)
    add_definitions(-DFFMPEG -D__STDC_CONSTANT_MACROS)
    include_directories(${FFMPEG_INCLUDE_DIR})
endif (USE_MPEG)

#######################################################################
if(INSTALL_PROFILES)
    file(WRITE  ${CMAKE_CURRENT_BINARY_DIR}/netgen.sh "#!/bin/sh\n")
    file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/netgen.sh "export PATH=${INSTALL_DIR}/bin:$PATH\n")
    file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/netgen.sh "export NETGENDIR=${INSTALL_DIR}/bin\n")
    file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/netgen.sh "export PYTHONPATH=${INSTALL_DIR}/${PYTHON_PACKAGES_INSTALL_DIR}:.\n")
    file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/netgen.sh "export LD_LIBRARY_PATH=${INSTALL_DIR}/lib:.\n")
    install(PROGRAMS ${CMAKE_CURRENT_BINARY_DIR}/netgen.sh DESTINATION /etc/profile.d)

    string(ASCII 27 Esc)
    file(WRITE  ${CMAKE_CURRENT_BINARY_DIR}/postinst "#!/bin/sh\n")
    file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/postinst "echo \"${Esc}[0;31mPlease log out and in again or do 'source /etc/profile.d/netgen.sh' to load the correct environment variables!${Esc}[m\"")
    set( CPACK_DEBIAN_PACKAGE_CONTROL_EXTRA "${CMAKE_CURRENT_BINARY_DIR}/postinst")
endif(INSTALL_PROFILES)

#######################################################################
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/fixup.cmake "\
include(${CMAKE_CURRENT_SOURCE_DIR}/cmake_modules/netgen_fixup.cmake)
set(APP ${INSTALL_DIR}/Netgen.app)
message(\${APP})
set(BU_CHMOD_BUNDLE_ITEMS ON)

file(GLOB libs ${CMAKE_INSTALL_PREFIX}/${ng_install_dir_lib}/*.dylib ${CMAKE_INSTALL_PREFIX}/${ng_install_dir_lib}/*.so)
message(\"\${libs}\")
netgen_fixup_bundle( \${APP}/Contents/MacOS/netgen \"\${libs}\" ${CMAKE_INSTALL_PREFIX}/${ng_install_dir_lib} )
execute_process(COMMAND ln -s /Applications ${INSTALL_DIR}/Applications)
set (bundle_filename \$ENV{NETGEN_BUNDLE_NAME})
if(NOT bundle_filename)
  set(bundle_filename netgen)
endif(NOT bundle_filename)
execute_process(COMMAND hdiutil create -volname Netgen -srcfolder ${INSTALL_DIR} -ov -format UDZO \${bundle_filename}-${PACKAGE_VERSION}.dmg)
")
  add_custom_target(bundle COMMAND ${CMAKE_COMMAND} "-P" "${CMAKE_CURRENT_BINARY_DIR}/fixup.cmake")
#######################################################################

add_subdirectory(libsrc)
add_subdirectory(ng)
add_subdirectory(tutorials)
if (USE_PYTHON)
  add_subdirectory(python)
endif (USE_PYTHON)
add_subdirectory(py_tutorials)
add_subdirectory(doc)
add_subdirectory(windows)
add_subdirectory(nglib)


#######################################################################
# CTest
enable_testing()
include(CTest)

#######################################################################
# Debian packager

if(UNIX)
    set(CPACK_SOURCE_GENERATOR "TGZ")
    set(CPACK_SOURCE_IGNORE_FILES "/cmake/;/build/;/.gz/;~$;${CPACK_SOURCE_IGNORE_FILES}")
    set(CPACK_PACKAGE_VERSION ${PACKAGE_VERSION} )

    set(CPACK_PACKAGE_NAME netgen)
    set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "automatic 3d tetrahedral mesh generator")
    set(CPACK_PACKAGE_DESCRIPTION "NETGEN is an automatic 3d tetrahedral mesh generator. It accepts input from constructive solid geometry (CSG) or boundary representation (BRep) from STL file format. The connection to a geometry kernel allows the handling of IGES and STEP files. NETGEN contains modules for mesh optimization and hierarchical mesh refinement. Netgen is open source based on the LGPL license. It is available for Unix/Linux and Windows.")

    set(CPACK_PACKAGING_INSTALL_PREFIX "/opt/netgen")
    execute_process(COMMAND grep CODENAME /etc/lsb-release OUTPUT_VARIABLE temp OUTPUT_STRIP_TRAILING_WHITESPACE)
    execute_process(COMMAND dpkg --print-architecture OUTPUT_VARIABLE CPACK_DEBIAN_PACKAGE_ARCHITECTURE OUTPUT_STRIP_TRAILING_WHITESPACE)
    if(temp)
        set(CPACK_GENERATOR "DEB")
        string(SUBSTRING ${temp} 17 -1 UBUNTU_VERSION)
        message("ubuntu version: ${UBUNTU_VERSION}")

        set(CPACK_DEBIAN_PACKAGE_DEPENDS "python3, libtk8.5, libtcl8.5, tix, libxmu6")
        execute_process(COMMAND dpkg --print-architecture OUTPUT_VARIABLE CPACK_DEBIAN_PACKAGE_ARCHITECTURE OUTPUT_STRIP_TRAILING_WHITESPACE)
        set(CPACK_DEBIAN_PACKAGE_MAINTAINER "Matthias Hochsteger <matthias.hochsteger@tuwien.ac.at>")
        if(USE_MPI)
            set(CPACK_DEBIAN_PACKAGE_DEPENDS "${CPACK_DEBIAN_PACKAGE_DEPENDS}, libmetis5, openmpi-bin")
            set(CPACK_PACKAGE_NAME "${CPACK_PACKAGE_NAME}_mpi")
        endif(USE_MPI)
        if(USE_OCC)
            set(CPACK_DEBIAN_PACKAGE_DEPENDS "${CPACK_DEBIAN_PACKAGE_DEPENDS}, liboce-ocaf-dev")
        endif(USE_OCC)
        set(CPACK_DEBIAN_PACKAGE_SECTION Science)
        set(CPACK_DEBIAN_PACKAGE_NAME ${CPACK_PACKAGE_NAME})
        set(CPACK_PACKAGE_FILE_NAME "netgen-${PACKAGE_VERSION}_${CPACK_DEBIAN_PACKAGE_ARCHITECTURE}")   
    endif(temp)
endif(UNIX)

if(APPLE)
    # create some auxiliary files
    set(mac_startup ${CMAKE_CURRENT_BINARY_DIR}/startup.sh)
    file(WRITE ${mac_startup} "\
#!/bin/sh
Netgen_BUNDLE=\"`echo \"$0\" | sed -e 's/\\/Contents\\/MacOS\\/startup.sh//'`\"
Netgen_MACOS=\"$Netgen_BUNDLE/Contents/MacOS\"
export NETGENDIR=$Netgen_MACOS  
export DYLD_LIBRARY_PATH=$Netgen_MACOS:$DYLD_LIBRARY_PATH
# export TIX_LIBRARY=$Netgen_MACOS/library
# export TCLLIBPATH=$Netgen_MACOS:$TCLLIBPATH
export PYTHONPATH=$Netgen_BUNDLE/Contents/Resources/${PYTHON_PACKAGES_INSTALL_DIR}:$PYTHONPATH
cd $Netgen_MACOS
$Netgen_MACOS/netgen
")
    install(PROGRAMS ${mac_startup} DESTINATION ${ng_install_dir_bin})

    set(mac_ngsuite ${CMAKE_CURRENT_BINARY_DIR}/ngsuite.sh)
    file(WRITE ${mac_ngsuite} "\
#!/bin/sh
Netgen_BUNDLE=\"`echo \"$0\" | sed -e 's/\\/Contents\\/MacOS\\/Netgen1//'`\"
Netgen_MACOS=\"$Netgen_BUNDLE/Contents/MacOS\"
open -a /Applications/Utilities/Terminal.app $Netgen_MACOS/startup.sh
")
    install(PROGRAMS ${mac_ngsuite} DESTINATION ../MacOS RENAME Netgen1)

    set(mac_plist ${CMAKE_CURRENT_BINARY_DIR}/Info.plist)
    file(WRITE ${mac_plist} "<?xml version=\"1.0\" encoding=\"UTF-8\"?>
<!DOCTYPE plist PUBLIC \"-//Apple Computer//DTD PLIST 1.0//EN\" \"http://www.apple.com/DTDs/PropertyList-1.0.dtd\">
<plist version=\"1.0\">
<dict>
	<key>CFBundleDevelopmentRegion</key>
	<string>English</string>
	<key>CFBundleExecutable</key>
	<string>Netgen1</string>
	<key>CFBundleIconFile</key>
	<string>Netgen.icns</string>
        <key>NSHighResolutionCapable</key>
        <string>True</string>
</dict>
</plist>
")
    install(FILES ${mac_plist} DESTINATION ../)
    install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/netgen.icns DESTINATION . RENAME Netgen.icns)

endif(APPLE)

if(NOT APPLE)
  include(CPack) 
endif()

#######################################################################
# uninstall target
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake_modules/cmake_uninstall.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
    IMMEDIATE @ONLY)
add_custom_target(uninstall
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake_modules/NetgenConfig.cmake.in
	${CMAKE_CURRENT_BINARY_DIR}/NetgenConfig.cmake @ONLY)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/NetgenConfig.cmake DESTINATION share/cmake)

#######################################################################
# Configure message
# TODO: other message in case of failure
string(TOUPPER "CMAKE_CXX_FLAGS_${CMAKE_BUILD_TYPE}" name)
set(flags "${${name}} ${CMAKE_CXX_FLAGS}")
message("
------------------------------------------------------------------------
  ${PROJECT_NAME} ${PACKAGE_VERSION}:  Automatic configuration OK.

  Build type: ${CMAKE_BUILD_TYPE}
  Flags: ${flags}

  Enabled functionality:

    OCC: ............... ${USE_OCC}
    JPEGlib: ........... ${USE_JPEG}
    FFMPEG: ............ ${USE_MPEG}
    GUI: ............... ${USE_GUI}
    MPI: ............... ${USE_MPI}
    PYTHON: ............ ${USE_PYTHON}


  Building: ")
if(WIN32)
message("
    Open ${CMAKE_BINARY_DIR}/Netgen.sln and build solution to compile ${PROJECT_NAME}.

    Build \"INSTALL\" to install ${PROJECT_NAME}.
")
else(WIN32)
message("
    Type 'make' to compile ${PROJECT_NAME}.

    Type 'make install' to install ${PROJECT_NAME}.
")
endif(WIN32)
message("
  Install directory:

    ${CMAKE_INSTALL_PREFIX}

  Please set the following environment variables:

    NETGENDIR=${CMAKE_INSTALL_PREFIX}/bin")
if(USE_PYTHON)
message("    PYTHONPATH=.:${CMAKE_INSTALL_PREFIX}/${PYTHON_PACKAGES_INSTALL_DIR}")
endif(USE_PYTHON)
message("
------------------------------------------------------------------------
")
if (ADDITIONAL_PATHS)
  message(WARNING "The use of ADDITIONAL_PATHS is depricated, use CMAKE_PREFIX_PATH instead")
endif (ADDITIONAL_PATHS)

