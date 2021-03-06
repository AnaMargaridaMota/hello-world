#/*============================================================================
#
#  PHDPROJECT: Project for visualisation of DBT.
#
#  Copyright (c) University College London (UCL). All rights reserved.
#
#  This software is distributed WITHOUT ANY WARRANTY; without even
#  the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR
#  PURPOSE.
#
#  See LICENSE.txt in the top level directory for details.
#
#============================================================================*/

######################################################################
# Set the minimum CMake version.
######################################################################
set(PHDPROJECT_CMAKE_MINIMUM_REQUIRED_VERSION 3.2)
cmake_minimum_required(VERSION ${PHDPROJECT_CMAKE_MINIMUM_REQUIRED_VERSION})


##################################################################################
# Set some CMake Policies.
# See http://cmake.org/cmake/help/cmake-2-8-docs.html#section_Policies for details
##################################################################################
set(project_policies )
foreach(policy ${project_policies})
  if(POLICY ${policy})
    cmake_policy(SET ${policy} NEW)
  endif()
endforeach()


##############################################################################
# Further variables for the external project names and locations are defined
# in the external project CMake files under CMake/ExternalProjects.
##############################################################################
set(NIFTK_EP_TARBALL_LOCATION "http://cmic.cs.ucl.ac.uk/platform/dependencies")


##############################################################################
# We have a super-build option. (Terminology comes from Kitware/ITK/MITK etc).
##############################################################################
option(BUILD_SUPERBUILD "Build PHDPROJECT and the projects it depends on via SuperBuild.cmake." ON)
option(BUILD_gflags "Build gflags." OFF)
option(BUILD_glog "Build glog." OFF)
option(BUILD_OpenCV "Build OpenCV." OFF)
option(BUILD_Eigen "Build Eigen." OFF)
option(BUILD_Boost "Build Boost." OFF)
option(BUILD_VTK "Build VTK." OFF)
option(BUILD_QTVTKGUI "Build QtVTK." OFF)
option(PHDPROJECT_USE_QT "Use Qt, normally for GUI development, but can be for anything." OFF)

if(BUILD_glog AND NOT BUILD_gflags)
  message("Forcing BUILD_gflags to ON due to BUILD_glog being ON.")
  set(BUILD_gflags ON CACHE BOOL "Build gflags." FORCE)
endif()

if(BUILD_QTVTKGUI AND NOT PHDPROJECT_USE_QT)
  message("Forcing PHDPROJECT_USE_QT to ON due to BUILD_QTVTKGUI being ON.")
  set(PHDPROJECT_USE_QT ON CACHE BOOL "Use Qt, normally for GUI development, but can be for anything." FORCE)
endif()

if(BUILD_QTVTKGUI AND NOT BUILD_VTK)
  message("Forcing BUILD_VTK to ON due to BUILD_QTVTKGUI being ON.")
  set(BUILD_VTK ON CACHE BOOL "Build VTK." FORCE)
endif()

if (BUILD_SUPERBUILD)
  project(PHDPROJECT-superbuild)
  set(PHDPROJECT_SOURCE_DIR ${PROJECT_SOURCE_DIR})
  set(PHDPROJECT_BINARY_DIR ${PROJECT_BINARY_DIR})
else()
  project(PHDPROJECT VERSION 00.00.1)
endif()


######################################################################
# PHDPROJECT uses KWStyle for checking the coding style
######################################################################
include(${CMAKE_SOURCE_DIR}/Utilities/KWStyle/PHDPROJECTKWStyle.cmake)


######################################################################
# PHDPROJECT uses CppCheck for static analysis
######################################################################
include(${CMAKE_SOURCE_DIR}/Utilities/CppCheck/PHDPROJECTCppCheck.cmake)


######################################################################
# Setting supported build types. Should ONLY be Release or Debug.
######################################################################
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Valid options are Release or Debug" FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Release" "Debug")
endif()

if (NOT (CMAKE_BUILD_TYPE STREQUAL "Release" OR CMAKE_BUILD_TYPE STREQUAL "Debug"))
  message(FATAL_ERROR "Build type \"${CMAKE_BUILD_TYPE}\" is not supported.")
endif()

if(WIN32)
  # Restrict the generated configuration to be what we configured above.
  # No point creating project files for build types that will not compile.
  # Note: it's set to FORCE so that both CMAKE_BUILD_TYPE and CMAKE_CONFIGURATION_TYPES match up.
  set(CMAKE_CONFIGURATION_TYPES ${CMAKE_BUILD_TYPE} CACHE STRING "Build configurations to generate." FORCE)
  mark_as_advanced(CMAKE_CONFIGURATION_TYPES)
endif()


######################################################################
# Make sure Git is available.
######################################################################
find_package(Git REQUIRED)
if (WIN32)
  set(GITCOMMAND ${GIT_EXECUTABLE})
endif()


######################################################################
# Check for Qt up-front, as VTK may need it in SuperBuild.
######################################################################
if(PHDPROJECT_USE_QT)
  set(ADDITIONAL_SEARCH_PATHS)
  set(_qt_components Concurrent OpenGL PrintSupport Script Sql Svg Widgets Xml XmlPatterns UiTools Help)
  if(UNIX AND NOT APPLE)
    list(APPEND _qt_components X11Extras)
  endif()
  set(QT5_LINK_LIBRARIES)
  find_package(Qt5 5.4.2 QUIET COMPONENTS ${_qt_components})
  if(Qt5_DIR)
    message(STATUS "Found Qt5: ${Qt5_DIR}")
    get_filename_component(_Qt5_DIR "${Qt5_DIR}/../../../" ABSOLUTE)
    list(FIND CMAKE_PREFIX_PATH "${_Qt5_DIR}" _result)
    if(_result LESS 0)
      set(CMAKE_PREFIX_PATH "${_Qt5_DIR};${CMAKE_PREFIX_PATH}" CACHE PATH "" FORCE)
    endif()
    set(PHDPROJECT_PREFIX_PATH ${_Qt5_DIR})
    foreach(_component ${_qt_components})
      find_package(Qt5${_component} REQUIRED QUIET)
      include_directories(${Qt5${_component}_INCLUDE_DIRS})
      add_definitions(${Qt5${_component}_DEFINITIONS})
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${Qt5${_component}_EXECUTABLE_COMPILE_FLAGS}")
      list(APPEND QT5_LINK_LIBRARIES Qt5::${_component})
    endforeach()
    set(_qmake_location )
    if(TARGET ${Qt5Core_QMAKE_EXECUTABLE})
      get_property(_qmake_location TARGET ${Qt5Core_QMAKE_EXECUTABLE}
                   PROPERTY IMPORT_LOCATION)
    endif()
    if(_qmake_location)
      if(NOT _qt_install_libs)
        if(WIN32)
          execute_process(COMMAND ${_qmake_location} -query QT_INSTALL_BINS
                          OUTPUT_VARIABLE _qt_install_libs
                          OUTPUT_STRIP_TRAILING_WHITESPACE)
        else()
          execute_process(COMMAND ${_qmake_location} -query QT_INSTALL_LIBS
                          OUTPUT_VARIABLE _qt_install_libs
                          OUTPUT_STRIP_TRAILING_WHITESPACE)
        endif()
        file(TO_CMAKE_PATH "${_qt_install_libs}" _qt_install_libs)
        set(_qt_install_libs ${_qt_install_libs} CACHE INTERNAL "Qt library installation prefix" FORCE)
      endif()
      if(_qt_install_libs)
        list(APPEND ADDITIONAL_SEARCH_PATHS ${_qt_install_libs})
      endif()
    else()
      message(WARNING "The qmake executable could not be found.")
    endif()
  endif()
endif(PHDPROJECT_USE_QT)

if(PHDPROJECT_USE_QT AND NOT Qt5_DIR)
  message(FATAL " Qt was required but not found")
endif()


######################################################################
# Setup the path to load CMake macros, and extra CMake files.
######################################################################
set(CMAKE_MODULE_PATH
    ${CMAKE_BINARY_DIR}
    ${CMAKE_SOURCE_DIR}/CMake
    ${CMAKE_MODULE_PATH}
   )
include(mitkMacroEmptyExternalProject)
include(mitkFunctionCheckCompilerFlags)
include(mpMacroInstall)


######################################################################
# Choose C++ standard
######################################################################
set(PHDPROJECT_CXX_STANDARD 11)
set(CMAKE_CXX_EXTENSIONS 0)
set(CMAKE_CXX_STANDARD ${PHDPROJECT_CXX_STANDARD})
set(CMAKE_CXX_STANDARD_REQUIRED 1)

# This is necessary to avoid problems with compile feature checks.
# CMAKE_CXX_STANDARD seems to only set the -std=c++11 flag for targets.
# However, compile flag checks also need to be done with -std=c++11.
# The PHDPROJECT_CXX11_FLAG variable is also used for external projects
# build during the PHDPROJECT super-build.
mitkFunctionCheckCompilerFlags("-std=c++11" PHDPROJECT_CXX11_FLAG)
if(NOT PHDPROJECT_CXX11_FLAG)
  # Older gcc compilers use -std=c++0x
  mitkFunctionCheckCompilerFlags("-std=c++0x" PHDPROJECT_CXX11_FLAG)
endif()


######################################################################
# Build options.
######################################################################
option(BUILD_TESTING "Build Unit tests." ON)
option(BUILD_SHARED_LIBS "Build Shared Libraries" OFF)
if(WIN32 AND NOT BUILD_SHARED_LIBS)
  add_definitions(-DPHDPROJECT_STATIC)
endif()


######################################################################
# Force MSVC runtime. Depends on BUILD_SHARED_LIBS
######################################################################
if(MSVC)
  set(variables
    CMAKE_C_FLAGS_DEBUG
    CMAKE_C_FLAGS_MINSIZEREL
    CMAKE_C_FLAGS_RELEASE
    CMAKE_C_FLAGS_RELWITHDEBINFO
    CMAKE_CXX_FLAGS_DEBUG
    CMAKE_CXX_FLAGS_MINSIZEREL
    CMAKE_CXX_FLAGS_RELEASE
    CMAKE_CXX_FLAGS_RELWITHDEBINFO
  )
  if(BUILD_SHARED_LIBS)
    message(STATUS "Forcing MSVC to use dynamic runtime variables.")
    foreach(variable ${variables})
      if(${variable} MATCHES "/MT")
        string(REGEX REPLACE "/MT" "/MD" ${variable} "${${variable}}")
      endif()
    endforeach()
  else()
    message(STATUS "Forcing MSVC to use static runtime variables.")
    foreach(variable ${variables})
      if(${variable} MATCHES "/MD")
        string(REGEX REPLACE "/MD" "/MT" ${variable} "${${variable}}")
      endif()
    endforeach()
  endif()
  message(STATUS "Initial build flags:")
  foreach(variable ${variables})
    message(STATUS "  '${variable}': ${${variable}}")
  endforeach()
  message(STATUS "")
endif()


######################################################################
# Configure Dart testing support.  This should be done before any
# message(FATAL_ERROR ...) commands are invoked.
######################################################################
include(${CMAKE_ROOT}/Modules/Dart.cmake)
mark_as_advanced(TCL_TCLSH DART_ROOT)
enable_testing()

if(BUILD_TESTING)
  set(BUILDNAME "PHDPROJECT" CACHE STRING "Name of build on the dashboard")
  mark_as_advanced(BUILDNAME)

  # Setup file for setting custom ctest vars
  configure_file(CMake/CTestCustom.cmake.in ${CMAKE_BINARY_DIR}/CTestCustom.cmake @ONLY)

  # Setup continuous test script
  configure_file(CMake/CTestContinuous.cmake.in ${CMAKE_BINARY_DIR}/CTestContinuous.cmake @ONLY)

endif(BUILD_TESTING)


######################################################################
# Now, if required, do the SuperBuild
# If we are doing SuperBuild
#   We configure up to this point (see the return() statement)
#   and then we call SuperBuild.cmake, which builds all the
#   dependencies as CMake ExternalProjects, and then also builds
#   PHDPROJECT as an ExternalProject. However instead of downloading
#   a tar file, you set the SOURCE_DIR to be THIS project, and force
#   the BUILD_SUPERBUILD flag to be off (to avoid infinite loop).
#
# If we are NOT doing superbuild, then the next statement has no
# effect, and the build goes on the same as before.
######################################################################
if(BUILD_SUPERBUILD)
  include("CMake/SuperBuild.cmake")
  return()
endif(BUILD_SUPERBUILD)


######################################################################
# End of SuperBuild. Print out where the source and binary folders
# are, just to make it really explicit... well, explicit to the user
# that bothers to read these messages! :-)
######################################################################
message("CMAKE_SOURCE_DIR=${CMAKE_SOURCE_DIR}")
message("CMAKE_BINARY_DIR=${CMAKE_BINARY_DIR}")


######################################################################
# Copy some reference files to output.
######################################################################
configure_file(${CMAKE_SOURCE_DIR}/Documentation/License.dox ${CMAKE_BINARY_DIR}/Doxygen/License.dox)
configure_file(${CMAKE_SOURCE_DIR}/LICENSE.txt ${CMAKE_BINARY_DIR}/LICENSE.txt @ONLY)
configure_file(${CMAKE_SOURCE_DIR}/README.md ${CMAKE_BINARY_DIR}/README.md @ONLY)
configure_file(${CMAKE_SOURCE_DIR}/README.txt ${CMAKE_BINARY_DIR}/README.txt @ONLY)
configure_file(${CMAKE_SOURCE_DIR}/INSTALLATION.txt ${CMAKE_BINARY_DIR}/INSTALLATION.txt @ONLY)
configure_file(${CMAKE_SOURCE_DIR}/CMake/Findglog.cmake.in ${CMAKE_BINARY_DIR}/Findglog.cmake @ONLY)
configure_file(${CMAKE_SOURCE_DIR}/CMake/Findgflags.cmake.in ${CMAKE_BINARY_DIR}/Findgflags.cmake @ONLY)


######################################################################
# Provide support for manually running Doxygen in build folder.
######################################################################
configure_file(${CMAKE_SOURCE_DIR}/Utilities/Doxygen/phdprojectdoxygen.pl.in ${CMAKE_BINARY_DIR}/phdprojectdoxygen.pl)
configure_file(${CMAKE_SOURCE_DIR}/Utilities/Doxygen/doxygen.config.in ${CMAKE_BINARY_DIR}/doxygen.config)


######################################################################
# Organise module/plugin/etc projects better within the IDE.
######################################################################
set_property(GLOBAL PROPERTY USE_FOLDERS ON)


######################################################################
# Add Mandatory Requirements
######################################################################
set(ALL_THIRD_PARTY_LIBRARIES)
list(APPEND ADDITIONAL_SEARCH_PATHS ${CMAKE_BINARY_DIR}/bin/${VS_BUILD_TYPE})


######################################################################
# Add Optional Requirements
######################################################################
if(WIN32)
  set(_library_sub_dir "/bin")
else()
  set(_library_sub_dir "/lib")
endif()

if(BUILD_gflags)
  # Example of: Small library where Findgflags.cmake is
  #             generated into the build folder to always pick up our one.
  find_package(gflags REQUIRED)
  find_package(Threads REQUIRED)
  include_directories(${gflags_INCLUDE_DIR})
  list(APPEND ALL_THIRD_PARTY_LIBRARIES ${gflags_LIBRARY} ${CMAKE_THREAD_LIBS_INIT})
  add_definitions(-DBUILD_gflags)
  if(WIN32)
    add_definitions(-DGOOGLE_GLOG_DLL_DECL=)
    list(APPEND ALL_THIRD_PARTY_LIBRARIES Shlwapi)
  endif()
  set(ADDITIONAL_SEARCH_PATHS "${gflags_DIR}${_library_sub_dir};${ADDITIONAL_SEARCH_PATHS}")
endif()

if(BUILD_glog)
  # Example of: Small library where Findglog.cmake is
  #             generated into the build folder to always pick up our one.
  find_package(glog REQUIRED)
  include_directories(${glog_INCLUDE_DIR})
  list(APPEND ALL_THIRD_PARTY_LIBRARIES ${glog_LIBRARY})
  add_definitions(-DBUILD_glog)
  set(ADDITIONAL_SEARCH_PATHS "${glog_DIR}${_library_sub_dir};${ADDITIONAL_SEARCH_PATHS}")
endif()

if(BUILD_Eigen)
  # Example of: Header only library
  find_package(Eigen REQUIRED)
  include_directories(${Eigen_INCLUDE_DIR})
  add_definitions(-DBUILD_Eigen)
  configure_file(${CMAKE_SOURCE_DIR}/Documentation/Licenses/Eigen.txt ${CMAKE_BINARY_DIR}/LICENSE_Eigen.txt)
endif()

if(BUILD_Boost)
  # Example of:
  #   (1) Standard, widely used library
  #   (2) Mostly header library, some compiled libraries
  #   (3) Your CMake probably has a standard FindBoost.cmake in its distribution,
  #       so Boost is probably found by CMake's own FindBoost.cmake.
  set(BOOST_LIBRARYDIR ${BOOST_ROOT}/lib)
  set(Boost_LIBRARY_DIR_DEBUG ${BOOST_ROOT}/lib)
  set(Boost_LIBRARY_DIR_RELEASE ${BOOST_ROOT}/lib) 
  set(BOOST_INCLUDEDIR ${BOOST_ROOT}/include)
  #set(Boost_DEBUG ON)
  set(Boost_NO_SYSTEM_PATHS ON)
  set(Boost_NO_BOOST_CMAKE ON)
  if(BUILD_SHARED_LIBS)
    set(Boost_USE_STATIC_LIBS OFF)
    set(Boost_USE_STATIC_RUNTIME OFF)
  else()
    set(Boost_USE_STATIC_LIBS ON)
    set(Boost_USE_STATIC_RUNTIME ON) 
  endif()
  find_package(Boost 1.56 EXACT COMPONENTS filesystem system date_time regex REQUIRED)
  include_directories(${Boost_INCLUDE_DIRS})
  link_directories(${Boost_LIBRARY_DIRS})
  if(WIN32)
    if(WITHIN_SUPERBUILD)
      add_definitions(-DBoost_LIB_DIAGNOSTIC_DEFINITIONS)  # To get debug messages
      add_definitions(-DBOOST_ALL_NO_LIB)                  # To stop auto-linking, which seems to be adding "lib" as library prefix in .obj files.
    endif()
    if(BUILD_SHARED)
      list(APPEND ALL_COMPILE_OPTIONS -DBOOST_ALL_DYN_LINK)
    endif()
  endif()
  list(APPEND ALL_THIRD_PARTY_LIBRARIES ${Boost_LIBRARIES})
  add_definitions(-DBUILD_Boost)
  set(ADDITIONAL_SEARCH_PATHS "${BOOST_ROOT}/bin;${ADDITIONAL_SEARCH_PATHS}")
  configure_file(${CMAKE_SOURCE_DIR}/Documentation/Licenses/Boost.txt ${CMAKE_BINARY_DIR}/LICENSE_Boost.txt)
endif()

if(BUILD_OpenCV)
  # Example of:
  #   (1) CMake doesn't include an OpenCV FindModule.cmake,
  #       but the installed package contains a OpenCVConfig.cmake
  #       which is found via the CMAKE_PREFIX_PATH. So, this is an
  #       example of 'config' mode.
  find_package(OpenCV REQUIRED)
  include_directories(${OpenCV_INCLUDE_DIRS})
  list(APPEND ALL_THIRD_PARTY_LIBRARIES ${OpenCV_LIBS})
  add_definitions(-DBUILD_OpenCV)
  set(ADDITIONAL_SEARCH_PATHS "${OpenCV_LIB_DIR}/../bin;${ADDITIONAL_SEARCH_PATHS}")
  configure_file(${CMAKE_SOURCE_DIR}/Documentation/Licenses/OpenCV.txt ${CMAKE_BINARY_DIR}/LICENSE_OpenCV.txt)
endif()

if(BUILD_VTK)
  # Example of:
  #   (1) Large Rendering library - oooooh, pretty pictures etc.
  #   (2) When SuperBuild builds VTK it adds VTK_DIR to CMAKE_PREFIX_PATH
  #       So, VTK is found using VTK's provided VTKConfig.cmake.
  #       Its called 'config mode' when running find_package
  #       Its the preferred approach because VTK can then control what is exposed.
  find_package(VTK REQUIRED)
  include(${VTK_USE_FILE})
  include_directories(${VTK_INCLUDE_DIRS})
  list(APPEND ALL_THIRD_PARTY_LIBRARIES ${VTK_LIBRARIES})
  add_definitions(-DBUILD_VTK)
endif()

set(ALL_LIBRARIES phdproject ${ALL_THIRD_PARTY_LIBRARIES})


######################################################################
# Compilation specific stuff, like flags etc.
######################################################################
if(WIN32)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DNOMINMAX /W2")
  set(CMAKE_CXX_WARNING_LEVEL 2)
  if(BUILD_SHARED AND WITHIN_SUPERBUILD)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DBOOST_LIB_PREFIX=\"\"")
  endif()
endif(WIN32)


######################################################################
# This is a list of all known GUI apps. Used for things like
# creating Mac OSX bundles, and each command line app gets copied
# into each OSX bundle.
######################################################################
set(_known_apps "")
if (BUILD_QTVTKGUI)
  list(APPEND _known_apps QtVTKApp)
endif()
set_property(GLOBAL PROPERTY PHDPROJECT_APPS ${_known_apps})


######################################################################
# Add code folders.
######################################################################
set(EXECUTABLE_OUTPUT_PATH ${CMAKE_BINARY_DIR}/bin)

foreach(type LIBRARY RUNTIME ARCHIVE)
  set(output_dir ${CMAKE_BINARY_DIR}/bin)
  set(CMAKE_${type}_OUTPUT_DIRECTORY ${output_dir} CACHE INTERNAL "Single output directory for building all libraries.")
  mark_as_advanced(CMAKE_${type}_OUTPUT_DIRECTORY)
endforeach()

include_directories(${CMAKE_SOURCE_DIR}/Code/Lib)

add_subdirectory(Code)

add_subdirectory(Documentation)

if(BUILD_TESTING)
  set(TEMP_DIR ${CMAKE_BINARY_DIR}/Testing/Temporary)
  include_directories(${CMAKE_SOURCE_DIR}/Testing/)
  add_subdirectory(Testing)
endif()


######################################################################
# Packaging code.
######################################################################
set(PHDPROJECT_PACKAGE_NAME "PhD_project" CACHE STRING "")
if(WIN32)
  set(PHDPROJECT_BASE_NAME "PhD_project" CACHE STRING "")
else()
  set(PHDPROJECT_BASE_NAME "phd_project" CACHE STRING "")
endif()
set(PHDPROJECT_VERSION_STRING "${PHDPROJECT_VERSION_MAJOR}.${PHDPROJECT_VERSION_MINOR}.${PHDPROJECT_VERSION_PATCH}" CACHE STRING "String to describe fully named version" FORCE)
set(PHDPROJECT_DEPLOY_NAME "${PHDPROJECT_BASE_NAME}-${PHDPROJECT_VERSION_STRING}" CACHE STRING "")
set(PHDPROJECT_CONTACT "Ana Margarida Mota <ninimota87@gmail.com>" CACHE STRING "")

# 1. Setup defaults, common for all generators.
include(CPackSetup)

# 2. Configuring the Start Menu and Desktop icon strings for NSIS
if (WIN32)
  set(ICONS_SETUP_STRING "")
  set(ICONS_REMOVE_STRING "")
  set(DESKTOP_ICONS_SETUP_STRING "")
  set(DESKTOP_ICONS_REMOVE_STRING "")
  set(DESKTOP_ICONS_REMOVE_STRING "")

  get_property(_apps GLOBAL PROPERTY PHDPROJECT_APPS)

  foreach(app_name ${_apps})

    message("Current App Name: ${app_name}")

    set(ICONS_SETUP_STRING "${ICONS_SETUP_STRING}CreateShortCut '$SMPROGRAMS\\\\$STARTMENU_FOLDER\\\\${app_name}.lnk' '$INSTDIR\\\\bin\\\\${app_name}.exe'\n")
    set(ICONS_REMOVE_STRING "${ICONS_REMOVE_STRING}Delete '$SMPROGRAMS\\\\$MUI_TEMP\\\\${app_name}.lnk'\n")

    set(DESKTOP_ICONS_SETUP_STRING "${DESKTOP_ICONS_SETUP_STRING}CreateShortCut '$DESKTOP\\\\${app_name}.lnk' '$INSTDIR\\\\bin\\\\${app_name}.exe'\n")
    set(DESKTOP_ICONS_REMOVE_STRING "${DESKTOP_ICONS_REMOVE_STRING}Delete '$DESKTOP\\\\${app_name}.lnk'\n")
  endforeach()
endif()

# 3. Set variables that may be platform (Windows/Linux/Mac) or Generator (TGZ,DEB,NSIS) specific.
#    When CPack runs, it just uses all the information in the generated files cmake_install.cmake.
#    So, CPack does not read all your configuration information in CMakeLists.txt, and CPack
#    does not re-run any cmake process.  So, it is cmake that reads all the CMakeLists.txt files
#    and generates all the cmake_install.cmake.  So this command will use cmake to generate
#    and additional file, that we can politely ask cpack to include, in addition to all the
#    cmake_install.cmake files.

configure_file(${CMAKE_SOURCE_DIR}/CMake/CPackOptions.cmake.in
               ${CMAKE_BINARY_DIR}/PHDPROJECTCPackOptions.cmake)

# 4. Set a variable with the name of this file.
set(CPACK_PROJECT_CONFIG_FILE "${CMAKE_BINARY_DIR}/PHDPROJECTCPackOptions.cmake")

# 5. Include this optional file.
include(PHDPROJECTCPackOptions)

# 6. Include CPack module once all variables are set.
include(CPack)

######################################################################
# If we are under Windows, create batch files which correctly
# set up the environment for Visual Studio. These are only used to
# start VS when developing. Not used in the final installation package.
######################################################################
if(WIN32)
  set(VS_SOLUTION_FILE "${PROJECT_BINARY_DIR}/${PROJECT_NAME}.sln")
  foreach(VS_BUILD_TYPE ${CMAKE_CONFIGURATION_TYPES})
    configure_file("${CMAKE_SOURCE_DIR}/CMake/StartVS.bat.in" ${PROJECT_BINARY_DIR}/StartVS_${VS_BUILD_TYPE}.bat @ONLY)
    message( "CreateWindowsBatchScript: Creating ${PROJECT_BINARY_DIR}/StartVS_${VS_BUILD_TYPE}.bat" )
  endforeach()
endif(WIN32)

