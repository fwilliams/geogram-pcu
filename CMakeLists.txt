##############################################################################
# Geogram/Vorpaline root CMakeList
##############################################################################

# CMake 2.8.11 is required for 2 reasons:
# - it is the first version that fully supports the specification of Visual
# Studio toolsets (v110_xp).
# - it is the version that supports the command string(TIMESTAMP ...)
cmake_minimum_required(VERSION 3.10)

# Note: geogram.cmake defines GEOGRAM_WITH_VORPALINE
# that we could have used instead,
# but geogram.cmake needs to be included after the project() 
# command, since project() resets CFLAGS and CXXFLAGS.

if(IS_DIRECTORY ${CMAKE_SOURCE_DIR}/src/lib/vorpalib)
    project(Vorpaline)
else()
    project(Geogram)
endif()

# Optional modules
# (can be overriden in CMakeOptions.txt)

# Set GEOGRAM_SUB_BUILD if Geogram sources included in buildtree, then
# VORPALINE_PLATFORM can be set directly in parent CMakeLists.txt
if(NOT GEOGRAM_SUB_BUILD)
   option(GEOGRAM_WITH_GRAPHICS "Viewers and geogram_gfx library" ON)
   option(GEOGRAM_WITH_LEGACY_NUMERICS "Legacy numerical libraries" ON)
   option(GEOGRAM_WITH_HLBFGS "Non-linear solver (Yang Liu's HLBFGS)" ON)
   option(GEOGRAM_WITH_TETGEN "Tetrahedral mesher (Hang Si's TetGen)" ON)
   option(GEOGRAM_WITH_TRIANGLE "Triangle mesher (Jonathan Shewchuk's triangle)" ON)
   option(GEOGRAM_WITH_EXPLORAGRAM "Experimental code (hexahedral meshing pipeline and optimal transport)" ON)
   option(GEOGRAM_WITH_LUA "Built-in LUA interpreter" ON)
   option(GEOGRAM_LIB_ONLY "Libraries only (no example programs/no viewer)" OFF)
   option(GEOGRAM_WITH_FPG "Predicate generator (Sylvain Pion's FPG)" OFF)
   option(GEOGRAM_USE_SYSTEM_GLFW3 "Use the version of GLFW3 installed in the system if found" OFF)
   set(VORPALINE_PLATFORM "" CACHE STRING "")
endif()

include(cmake/geogram.cmake)

set(VORPALINE_VERSION_MAJOR 1)
set(VORPALINE_VERSION_MINOR 6)
set(VORPALINE_VERSION_PATCH 8)
set(VORPALINE_VERSION ${VORPALINE_VERSION_MAJOR}.${VORPALINE_VERSION_MINOR}.${VORPALINE_VERSION_PATCH})

set(VORPALINE_INCLUDE_SUBPATH geogram${VORPALINE_VERSION_MAJOR})

# Determine the current Build-OS (Build-platform without the compiler info)
string(REGEX REPLACE "-[^-]+$" "" VORPALINE_OS ${VORPALINE_PLATFORM})

# Determine the current build date
string(TIMESTAMP VORPALINE_BUILD_DATE "%Y-%m-%d %H:%M:%S")
string(TIMESTAMP YEAR "%Y")

# Determine the current build number
# This is set by Jenkins in environment variable BUILD_NUMBER
set(VORPALINE_BUILD_NUMBER $ENV{BUILD_NUMBER})


##############################################################################
# Get SVN revision info

if(GEOGRAM_WITH_VORPALINE)
   find_package(Subversion QUIET)
   if(NOT SUBVERSION_FOUND)
       message(WARNING "Subversion executable not found - cannot determine current revision") 
   else()
       Subversion_WC_INFO(${PROJECT_SOURCE_DIR} Vorpaline)
       message(STATUS "Vorpaline revision is ${Vorpaline_WC_REVISION}")
       set(VORPALINE_SVN_REVISION ${Vorpaline_WC_REVISION})
   endif()
endif()   

##############################################################################
# RPATH (where executables find the .so / DLLs)
# - Enables RPATH support for MACOSX
# - Makes RPATH of dynamic libraries and executable point to the directory
#   where libraries are installed.

if(VORPALINE_BUILD_DYNAMIC)
   set(CMAKE_MACOSX_RPATH 1)
   set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
endif()

set(CMAKE_INSTALL_NAME_DIR "${CMAKE_INSTALL_PREFIX}/lib")

# uninstall target
# Needs to be created before add_subdirectory() because GLFw has
# also an uninstall target that will be inhibited if there is
# already one (Geogram's one needs to be first)

configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
    IMMEDIATE @ONLY)

#add_custom_target(uninstall
#    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)

##############################################################################
# Geogram/Vorpaline sources

add_subdirectory(src/lib/geogram)
if(GEOGRAM_WITH_VORPALINE)
   add_subdirectory(src/lib/vorpalib)
endif()

add_subdirectory(src/lib/third_party)

if(GEOGRAM_WITH_GRAPHICS)
  add_subdirectory(src/lib/geogram_gfx)
endif()

if(IS_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/src/lib/exploragram)
  if(GEOGRAM_WITH_EXPLORAGRAM)
     add_subdirectory(src/lib/exploragram)
  endif()
endif()

if(NOT GEOGRAM_LIB_ONLY)
  add_subdirectory(src/bin)
  add_subdirectory(src/tests)
  add_subdirectory(src/examples)
  add_subdirectory(tests)
endif()

add_subdirectory(doc)

##############################################################################
# Cleanup from previous builds

file(REMOVE ${CMAKE_BINARY_DIR}/doc/LICENSE.txt)


##############################################################################
# Geogram installation


# FindGeogram.cmake

install(FILES cmake/FindGeogram.cmake DESTINATION lib/cmake/modules COMPONENT devkit)

# Configure CPack

set(CPACK_PACKAGE_NAME ${CMAKE_PROJECT_NAME})
set(CPACK_SYSTEM_NAME ${VORPALINE_OS})
set(CPACK_PACKAGE_VENDOR "INRIA - ALICE")


if(${GEOGRAM_WITH_VORPALINE})
    set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "A flexible mesh generator")
else()
    set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "fast, simple and easy-to-use primitives for geometric programming")
endif()

set(CPACK_PACKAGE_VERSION_MAJOR ${VORPALINE_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${VORPALINE_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${VORPALINE_VERSION_PATCH})
set(CPACK_PACKAGE_VERSION ${VORPALINE_VERSION})
set(CPACK_PACKAGE_FILE_NAME ${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION})
set(CPACK_COMPONENT_INCLUDE_TOPLEVEL_DIRECTORY true)

if(CPACK_GENERATOR STREQUAL "DEB")
  set(CPACK_PACKAGING_INSTALL_PREFIX ${CMAKE_INSTALL_PREFIX})
  set(CPACK_PACKAGE_CONTACT Bruno.Levy@inria.fr)
# set(CPACK_PACKAGE_DEPENDS "libglfw3 (>= 3.2-1), libc6 (>= 2.22-11), libstdc++ (>= 6.1.1-4)")
#   TODO: use objdump -p | grep NEEDED to automate...
#   or GET_PROPERTY(result GLOBAL ENABLED_FEATURES)  (successful FIND_PACKAGE())
endif()

if(NOT DEFINED CPACK_GENERATOR) 
  if(WIN32)
    set(CPACK_GENERATOR ZIP)
  else()
    set(CPACK_GENERATOR TGZ)
  endif()
endif()

# Enable component-based packaging for archive generators (TGZ, ZIP)
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)

set(CPACK_COMPONENTS_ALL runtime devkit devkit-full doc-devkit doc-devkit-full)
set(CPACK_COMPONENTS_GROUPING "IGNORE")

set(CPACK_COMPONENT_RUNTIME_DISPLAY_NAME "Vorpaline Application")
set(CPACK_COMPONENT_RUNTIME_GROUP "Runtime")

set(CPACK_COMPONENT_DEVKIT_DISPLAY_NAME "Vorpaline Developer Kit")
set(CPACK_COMPONENT_DEVKIT_GROUP "Development")

set(CPACK_COMPONENT_DEVKIT-FULL_DISPLAY_NAME "Vorpaline Full Developer Kit")
set(CPACK_COMPONENT_DEVKIT-FULL_GROUP "Development")

set(CPACK_COMPONENT_DOC-DEVKIT_DISPLAY_NAME "Vorpaline API Developer Kit Documentation")
set(CPACK_COMPONENT_DOC-DEVKIT_GROUP "Documentation")

set(CPACK_COMPONENT_DOC-DEVKIT-FULL_DISPLAY_NAME "Vorpaline Full Developer Kit Documentation")
set(CPACK_COMPONENT_DOC-DEVKIT-FULL_GROUP "Documentation")

set(CPACK_COMPONENT_DOC-DEVKIT-INTERNAL_DISPLAY_NAME "Vorpaline Internal Developer Kit Documentation")
set(CPACK_COMPONENT_DOC-DEVKIT-INTERNAL_GROUP "Documentation")

# Copy the helper script to build individual packages to the binary directory
configure_file(
    tools/make_package.pl.in
    make_package.pl
    @ONLY
)

# This must always be last!
include(CPack)

