project (widelands)

cmake_minimum_required (VERSION 2.8.7)
cmake_policy(VERSION 2.8.7)

include("${CMAKE_SOURCE_DIR}/cmake/UseSystemInfo.cmake")

# This policy is not known to versions prior 3.1 and would result in errors,
# if set on such systems. This can be removed when cmake_minimum_required is set
# to 3.1 or newer by using:
# cmake_policy(VERSION 3.1)
if(POLICY CMP0054)
   cmake_policy(SET CMP0054 NEW)
endif(POLICY CMP0054)

# This policy is not known to versions prior 3.11 and would result in errors,
# if set on such systems. This can be removed when cmake_minimum_required is set
# to 3.11 or newer  by using:
# cmake_policy(VERSION 3.11)
if(POLICY CMP0072)
  cmake_policy(SET CMP0072 NEW)

  # TODO(stonerl): Check on later Ubuntu releases whether this workaround is
  # still needed or not.
  if (CMAKE_SYSTEM MATCHES "Linux")
    distro_name (DISTRO_NAME)
    # DISTRO_NAME is different on the build-servers.
    if ((DISTRO_NAME MATCHES "Ubuntu 18.10" OR DISTRO_NAME MATCHES "Cosmic"
         OR DISTRO_NAME MATCHES "Ubuntu 19.04" OR DISTRO_NAME MATCHES "Disco"
         OR DISTRO_NAME MATCHES "Ubuntu 19.10" OR DISTRO_NAME MATCHES "Eoan")
        AND CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
      cmake_policy(SET CMP0072 OLD)
    endif()
  endif()
endif(POLICY CMP0072)

# This policy is not known to versions prior 3.12 and would result in errors,
# if set on such systems. This can be removed when cmake_minimum_required is set
# to 3.12 or newer  by using:
# cmake_policy(VERSION 3.12)
if(POLICY CMP0074)
  cmake_policy(SET CMP0074 NEW)
endif(POLICY CMP0074)

include("${CMAKE_SOURCE_DIR}/cmake/WlFunctions.cmake")

option(OPTION_USE_GLBINDING "Use glbinding instead of GLEW" OFF)
option(OPTION_GLEW_STATIC "Use static GLEW Library" OFF)
option(OPTION_BUILD_WEBSITE_TOOLS "Build website-related tools" ON)
option(OPTION_BUILD_TRANSLATIONS "Build translations" ON)
option(OPTION_BUILD_TESTS "Build tests" ON)
option(OPTION_BUILD_CODECHECK "Build codecheck" ON)

option(USE_XDG "Follow XDG-Basedir specification" ON) # Enabled by default
IF(USE_XDG AND NOT APPLE AND NOT WIN32)
  ADD_DEFINITIONS(-DUSE_XDG)
  message(STATUS "Building with XDG support")
ENDIF(USE_XDG AND NOT APPLE AND NOT WIN32)

if (CMAKE_SOURCE_DIR STREQUAL CMAKE_BINARY_DIR)
  message(FATAL_ERROR "Build directory and source directory must not be the same.")
endif (CMAKE_SOURCE_DIR STREQUAL CMAKE_BINARY_DIR)

# Define the directory structure for installation - will be hardcoded in WL bin
# (through config.h). If not specified, we are going to use the directory where
# the executable is in. Also on Linux.
# Packagers (or people using make install) have to set this variable to an absolute path.
wl_set_if_unset(WL_INSTALL_BASEDIR ".")

# Define the directory structure for installation of the data files - will be hardcoded
# in WL bin (through config.h). If not specified, we are going to use the "data" directory
# below the directory where the executable is in. Also on Linux.
# Packagers (or people using make install) have to set this variable to an absolute path.
wl_set_if_unset(WL_INSTALL_DATADIR "./data")

if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
  if (CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.8)
    message(FATAL_ERROR "Widelands needs GCC >= 4.8 to compile.")
  endif()
elseif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
  if (CMAKE_CXX_COMPILER_VERSION VERSION_LESS 3.4)
    message(FATAL_ERROR "Clang version must be at least 3.4!")
  endif()
else()
  message(WARNING "You are using an unsupported compiler! Supported are Clang and GCC.")
endif()

SET(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules)

if (WIN32)
  set (Boost_USE_STATIC_LIBS  ON)
  link_libraries(wsock32 ws2_32 bcrypt)
else()
  set (Boost_USE_STATIC_LIBS  OFF)
endif()
set (Boost_USE_MULTITHREADED ON)
set (Boost_DETAILED_FAILURE_MSG ON)
find_package(Boost 1.48
  COMPONENTS
    unit_test_framework
    regex
  REQUIRED
    system)

find_package (PythonInterp REQUIRED)

find_package(Gettext REQUIRED)
find_package(OpenGL REQUIRED)
find_package(PNG REQUIRED)
find_package(SDL2 REQUIRED)
find_package(SDL2_image REQUIRED)
find_package(SDL2_mixer REQUIRED)
find_package(SDL2_ttf REQUIRED)
find_package(ZLIB REQUIRED)
find_package(ICU REQUIRED)
if(OPTION_USE_GLBINDING)
  find_package(glbinding REQUIRED)
else()
  find_package(GLEW REQUIRED)
endif()

if (APPLE OR WIN32 OR
    CMAKE_SYSTEM_NAME MATCHES "FreeBSD" OR
    CMAKE_SYSTEM_NAME MATCHES "OpenBSD")
  if (NOT CMAKE_SYSTEM_NAME MATCHES "kFreeBSD")
    find_package(intl REQUIRED)
  endif()
endif()

# Disable no symbols warning on macOS, but only on versions where this is
# supported. It is not supported on OS X 10.7 (DARWIN_MAJOR_VERSION == 11).
if (APPLE)
    if (${DARWIN_MAJOR_VERSION} GREATER 11)
        SET(CMAKE_C_ARCHIVE_CREATE   "<CMAKE_AR> Scr <TARGET> <LINK_FLAGS> <OBJECTS>")
        SET(CMAKE_CXX_ARCHIVE_CREATE "<CMAKE_AR> Scr <TARGET> <LINK_FLAGS> <OBJECTS>")
        SET(CMAKE_C_ARCHIVE_FINISH   "<CMAKE_RANLIB> -no_warning_for_no_symbols -c <TARGET>")
        SET(CMAKE_CXX_ARCHIVE_FINISH "<CMAKE_RANLIB> -no_warning_for_no_symbols -c <TARGET>")
    endif()
endif()

# TODO(sirver): One day, this should be enabled. Then we have no more cycles in our dependencies....
# set_property(GLOBAL PROPERTY GLOBAL_DEPENDS_NO_CYCLES ON)

if (NOT CMAKE_BUILD_TYPE OR CMAKE_BUILD_TYPE STREQUAL "")
  find_path(FILE_WL_RELEASE "WL_RELEASE" ${CMAKE_CURRENT_SOURCE_DIR})
  if(${FILE_WL_RELEASE} STREQUAL "FILE_WL_RELEASE-NOTFOUND")
    set(CMAKE_BUILD_TYPE Debug)
  else()
    set(CMAKE_BUILD_TYPE Release)
  endif()
endif (NOT CMAKE_BUILD_TYPE OR CMAKE_BUILD_TYPE STREQUAL "")

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
  set(WL_DEBUG_FLAGS "-g -DDEBUG")
  option(OPTION_ASAN "Build with AddressSanitizer" ON)
elseif(CMAKE_BUILD_TYPE STREQUAL "Release")
  if(("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU") AND (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 6.2))
    message(STATUS "GCC >6.2.0 breaks -03, setting -02")
    set(WL_OPTIMIZE_FLAGS "-O2")
  else()
    set(WL_OPTIMIZE_FLAGS "-O3")
  endif ()
  set(WL_DEBUG_FLAGS "-DNDEBUG -DNOPARACHUTE")
  option(OPTION_ASAN "Build with AddressSanitizer" OFF)
elseif(CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
  if(("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU") AND (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 6.2))
    message(STATUS "GCC >6.2.0 breaks -03, setting -02")
    set(WL_OPTIMIZE_FLAGS "-O2")
  else()
    set(WL_OPTIMIZE_FLAGS "-O3")
  endif ()
  set(WL_DEBUG_FLAGS "-DNDEBUG -DNOPARACHUTE")
  option(OPTION_ASAN "Build with AddressSanitizer" ON)
else()
  message(FATAL_ERROR "Unknown CMAKE_BUILD_TYPE: ${CMAKE_BUILD_TYPE}")
endif()

wl_add_flag(WL_GENERIC_CXX_FLAGS "-std=c++11")

if(OPTION_ASAN)
  message(STATUS "Using AddressSanitizer http://clang.llvm.org/docs/AddressSanitizer.html")
  # See http://clang.llvm.org/docs/AddressSanitizer.html
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-fsanitize=address")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-fno-omit-frame-pointer")
  set (CMAKE_EXE_LINKER_FLAGS "-fsanitize=address" CACHE STRING "Set by widelands CMakeLists.txt" FORCE)
else()
  message(STATUS "Not using AddressSanitizer.")
endif(OPTION_ASAN)

# This is set to avoid linker errors when using GLVND-libs on Linux
if("${OpenGL_GL_PREFERENCE}" STREQUAL "GLVND")
   link_libraries("GL")
   add_compile_definitions(WL_USE_GLVND)
   message(STATUS "Adding linker flags for GLVND.")
endif()

if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Weverything")

  # Disabled warnings that are overly verbose right now or just do not make sense.
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wno-c++98-compat")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wno-c++98-compat-pedantic")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wno-conversion")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wno-exit-time-destructors")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wno-global-constructors")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wno-padded")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wno-sign-conversion")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wno-missing-noreturn")

  # It is impossible to write code that both GCC and Clang will like,
  # so we have to switch off the warning for one of them.
  # http://clang-developers.42468.n3.nabble.com/Question-on-Wswitch-enum-td4025927.html
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wno-switch-enum")

  # TODO(sirver): weak-vtables should be enabled, but leads to lot of errors right now.
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wno-weak-vtables")

  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wno-unreachable-code")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wno-documentation")

  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Werror=deprecated")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Werror=non-pod-varargs")

  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Qunused-arguments")

  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wint-to-void-pointer-cast")
  if(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 3.8)
      wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Winconsistent-missing-override")
  endif()
else()
  # Assuming GCC 4.8 or higher.
  if(WIN32)
    # This is needed for getenv().
    wl_add_flag(WL_GENERIC_CXX_FLAGS "-std=gnu++11")
  else()
    # SDL and MinGW both declare 'unsigned int __builtin_ia32_crc32qi(unsigned int, unsigned char)', resulting in lots of warnings. So, we can't have this flag in Windows.
    wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wredundant-decls")
  endif()

  # Because of: https://svn.boost.org/trac/boost/ticket/9240.
  wl_add_flag(WL_GENERIC_CXX_FLAGS "-fext-numeric-literals")

  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wall")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wpedantic")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wdeprecated-declarations")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wextra")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wformat")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wformat-nonliteral")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wformat-security")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wformat-y2k")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Winit-self")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Winvalid-pch")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wlogical-op")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wmissing-include-dirs")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wno-undef")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wold-style-cast")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Woverlength-strings")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wpacked")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wpointer-arith")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wsign-promo")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wsync-nand")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wtrampolines")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wundef")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wunused")
  wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Wunused-macros")
endif()


# Turn some warnings into errors.
wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Werror=format-security")
wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Werror=return-type")
wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Werror=shadow")
wl_add_flag(WL_COMPILE_DIAGNOSTICS "-Werror=uninitialized")

IF (WIN32)
  add_definitions(-DMINGW_HAS_SECURE_API -DWIN32_LEAN_AND_MEAN -D__STDC_FORMAT_MACROS -D__USE_MINGW_ANSI_STDIO)
  if (CMAKE_SIZEOF_VOID_P EQUAL 4)
    set (CMAKE_EXE_LINKER_FLAGS "-Wl,--large-address-aware" CACHE STRING "Set by widelands CMakeLists.txt" FORCE)
    message (STATUS "Enabled large address awareness on mingw32")
  else (CMAKE_SIZEOF_VOID_P EQUAL 4)
    message (STATUS "Detected mingw32-w64")
  endif (CMAKE_SIZEOF_VOID_P EQUAL 4)
endif (WIN32)

# on BSD this must be explicitly linked
if (CMAKE_SYSTEM_NAME MATCHES "FreeBSD" OR CMAKE_SYSTEM_NAME MATCHES "OpenBSD")
  # Not needed on Debian GNU/kFreeBSD..
  if (NOT CMAKE_SYSTEM_NAME MATCHES "kFreeBSD")
    find_library(EXECINFO_LIBRARY NAMES execinfo)
  endif (NOT CMAKE_SYSTEM_NAME MATCHES "kFreeBSD")

  # OpenBSD needs the X11 include directory in order to find GL/glu.h
  if (CMAKE_SYSTEM_NAME MATCHES "OpenBSD")
    find_package(X11 REQUIRED)
    include_directories(${X11_INCLUDE_DIR})
  endif ()
endif (CMAKE_SYSTEM_NAME MATCHES "FreeBSD" OR CMAKE_SYSTEM_NAME MATCHES "OpenBSD")

if (NOT DEFINED WL_VERSION)
  include (${CMAKE_SOURCE_DIR}/cmake/BzrRevision.cmake)
else (NOT DEFINED WL_VERSION)
  add_custom_target (
    InputRevision ALL
    COMMAND ${CMAKE_COMMAND} -DWL_INSTALL_BASEDIR=${WL_INSTALL_BASEDIR} -DWL_VERSION=${WL_VERSION} -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} -DSOURCE_DIR=${CMAKE_CURRENT_SOURCE_DIR} -DBINARY_DIR=${CMAKE_CURRENT_BINARY_DIR} -P ${CMAKE_CURRENT_SOURCE_DIR}/cmake/InputRevision.cmake
  )
endif (NOT DEFINED WL_VERSION)

# Enable testing.
if (OPTION_BUILD_TESTS)
  include(CTest)
  enable_testing()

  # Run a test after a normal compile. This magic is needed as 'make test' will
  # not rebuild tests:
  # http://stackoverflow.com/questions/733475/cmake-ctest-make-test-doesnt-build-tests
  add_custom_target(_run_all_tests ALL
    COMMAND ${CMAKE_CTEST_COMMAND} --output-on-failure
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    DEPENDS wl_tests
  )
endif (OPTION_BUILD_TESTS)

install (
  FILES ${CMAKE_CURRENT_BINARY_DIR}/VERSION
  DESTINATION ${WL_INSTALL_BASEDIR}
  CONFIGURATIONS Debug;Release
  COMPONENT CoreVersionFile
)

# Installation.
add_subdirectory(doc)
if(OPTION_BUILD_TRANSLATIONS)
  add_subdirectory(po)
endif(OPTION_BUILD_TRANSLATIONS)
add_subdirectory(src)

# install files to the correct locations here
install(
  DIRECTORY
    data/ai
    data/campaigns
    data/i18n
    data/images
    data/scripting
    data/shaders
    data/templates
    data/tribes
    data/txts
    data/world
  DESTINATION ${WL_INSTALL_DATADIR}
  CONFIGURATIONS Debug;Release
  COMPONENT CoreDataFiles
)

install(
  DIRECTORY
    data/maps
  DESTINATION ${WL_INSTALL_DATADIR}
  CONFIGURATIONS Debug;Release
  COMPONENT MapFiles
)

install(
  DIRECTORY
    data/music
    data/sound
  DESTINATION ${WL_INSTALL_DATADIR}
  CONFIGURATIONS Debug;Release
  COMPONENT MusicFiles
)

install(
  FILES
    COPYING
    CREDITS
    ChangeLog
  DESTINATION ${WL_INSTALL_BASEDIR}
  CONFIGURATIONS Debug;Release
  COMPONENT CoreLicenseFiles
)

install(
  DIRECTORY
    doc
  DESTINATION ${WL_INSTALL_BASEDIR}
  CONFIGURATIONS Debug
  COMPONENT DocFiles
  PATTERN "CMakeLists.txt" EXCLUDE
)

set_directory_properties(PROPERTIES ADDITIONAL_MAKE_CLEAN_FILES ${WL_INSTALL_DATADIR}/locale)

install(
  DIRECTORY
    ${CMAKE_CURRENT_BINARY_DIR}/locale/
  DESTINATION ${WL_INSTALL_DATADIR}/locale
  CONFIGURATIONS Debug;Release
  COMPONENT CoreLanguageFiles
)

if (UNIX AND NOT APPLE)
  add_subdirectory(xdg)
endif (UNIX AND NOT APPLE)
