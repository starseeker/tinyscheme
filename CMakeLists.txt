# This file contains the top level CMakeLists.txt logic for the
# TinyScheme software package.

# Minimum required version of CMake
CMAKE_MINIMUM_REQUIRED(VERSION 2.8.3)
if(COMMAND CMAKE_POLICY)
  CMAKE_POLICY(SET CMP0003 NEW)
  CMAKE_POLICY(SET CMP0007 OLD)
  if(${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}.${CMAKE_PATCH_VERSION}
      VERSION_GREATER 2.8.3)
    CMAKE_POLICY(SET CMP0017 OLD)
  endif(${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}.${CMAKE_PATCH_VERSION}
    VERSION_GREATER 2.8.3)
endif(COMMAND CMAKE_POLICY)

# Project name
project(TINYSCHEME)

# Version information
set(TINYSCHEME_VERSION 1.40)

#---------------------------------------------------------------------
# The following logic is what allows binaries to run successfully in
# the build directory AND install directory.  Thanks to plplot for
# identifying the necessity of setting CMAKE_INSTALL_NAME_DIR on OSX.

# use, i.e. don't skip the full RPATH for the build tree
set(CMAKE_SKIP_BUILD_RPATH  FALSE)

# when building, don't use the install RPATH already
# (but later on when installing)
set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)

# the RPATH/INSTALL_NAME_DIR to be used when installing
if (NOT APPLE)
  set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib:\$ORIGIN/../lib")
endif(NOT APPLE)
set(CMAKE_INSTALL_NAME_DIR "${CMAKE_INSTALL_PREFIX}/lib")

# add the automatically determined parts of the RPATH which point to
# directories outside the build tree to the install RPATH
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

#---------------------------------------------------------------------
# Output directories - this is where built library and executable
# files will be placed after building but prior to install.
if(NOT CMAKE_LIBRARY_OUTPUT_DIRECTORY)
  if(WIN32)
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${TINYSCHEME_BINARY_DIR}/bin    CACHE INTERNAL "Single output directory for building all libraries.")
  else(WIN32)
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY
      ${TINYSCHEME_BINARY_DIR}/lib CACHE INTERNAL "Single output directory for building all    libraries.")
  endif(WIN32)
endif(NOT CMAKE_LIBRARY_OUTPUT_DIRECTORY)
if(NOT CMAKE_ARCHIVE_OUTPUT_DIRECTORY)
  set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${TINYSCHEME_BINARY_DIR}/lib CACHE INTERNAL "Single output directory for building all archives.")
endif(NOT CMAKE_ARCHIVE_OUTPUT_DIRECTORY)
if(NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY)
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${TINYSCHEME_BINARY_DIR}/bin CACHE INTERNAL "Single output directory for building all     executables.")
endif(NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY)


#---------------------------------------------------------------------
# Configure install locations.  Don't set these if they have already
# been set by some other means (like a higher level CMakeLists.txt file
# including this one.)

# The location in which to install executables.
if(NOT BIN_DIR)
  set(BIN_DIR bin)
endif(NOT BIN_DIR)

# The location in which to install header files.
if(NOT INCLUDE_DIR)
  set(INCLUDE_DIR include)
endif(NOT INCLUDE_DIR)

# The location in which to install libraries.
if(NOT LIB_DIR)
  if(NOT WIN32)
    set(LIB_DIR lib)
  else(NOT WIN32)
    set(LIB_DIR bin)
  endif(NOT WIN32)
endif(NOT LIB_DIR)

#---------------------------------------------------------------------
# Features

# convenience macro
macro(TS_OPT define default helpstring)
  if(NOT DEFINED ENABLE_${define})
    set(ENABLE_${define} ${default})
  endif(NOT DEFINED ENABLE_${define})
  if(ENABLE_${define})
    add_definitions(-DUSE_${define}=1)
  endif(ENABLE_${define})
  set(ENABLE_${define} ${ENABLE_${define}} CACHE BOOL "${helpstring}")
endmacro(TS_OPT)

# Enables dynamically loaded routines. If you define this symbol,
# need to add dynload.c in your compile.
if(NOT DEFINED BUILD_SHARED_LIBS)
  set(BUILD_SHARED_LIBS ON)
endif(NOT DEFINED BUILD_SHARED_LIBS)

if(BUILD_SHARED_LIBS)
  add_definitions(-DUSE_DL=1)
endif(BUILD_SHARED_LIBS)

TS_OPT(STANDALONE OFF "Define this to produce a standalone interpreter.")
TS_OPT(MATH ON "Includes math routines.")
TS_OPT(CHAR_CLASSIFIERS ON "Includes character classifier procedures.")
TS_OPT(ASCII_NAMES ON "Enable extended character notation based on ASCII names.")
TS_OPT(STRING_PORTS ON "Enables string ports.")
TS_OPT(ERROR_HOOK OFF "Force system errors through user-defined error handling.")
TS_OPT(TRACING OFF "Enable the use of TRACING.")
TS_OPT(COLON_HOOK OFF "Enable use of qualified identifiers.")
TS_OPT(PLIST OFF "Enables property lists.")
TS_OPT(STRCASECMP OFF "Defines stricmp as strcasecmp, for Unix.")
TS_OPT(SCHEME_STACK OFF "Enables 'cons' stack.")

if(NOT DEFINED ENABLE_TINYSCHEME_REPL)
  set(ENABLE_TINYSCHEME_REPL ON CACHE BOOL "Build scheme REPL application." )
else(NOT DEFINED ENABLE_TINYSCHEME_REPL)
  set(ENABLE_TINYSCHEME_REPL ${ENABLE_TINYSCHEME_REPL} CACHE BOOL "Build scheme REPL application." )
endif(NOT DEFINED ENABLE_TINYSCHEME_REPL)

if(WIN32)
  # Informs TinyScheme that stdio translates "\n" to "\r\n".
  add_definitions(-DSTDIO_ADDS_CR=1)
endif(WIN32)

if(CYGWIN)
  add_definitions(-DUSE_STRLWR=0)
endif(CYGWIN)

include(CheckLibraryExists)
CHECK_LIBRARY_EXISTS(m cos "" HAVE_M_LIBRARY)
if(HAVE_M_LIBRARY)
  set(SYS_LIBS ${SYS_LIBS} m)
endif(HAVE_M_LIBRARY)
CHECK_LIBRARY_EXISTS(dl dlopen "" HAVE_STD_DLOPEN)
IF(HAVE_STD_DLOPEN)
  set(SYS_LIBS ${SYS_LIBS} dl)
ELSE(HAVE_STD_DLOPEN)
  CHECK_LIBRARY_EXISTS(c dlopen "" HAVE_C_DLOPEN)
  IF(HAVE_C_DLOPEN)
    set(SYS_LIBS ${SYS_LIBS} c)
  ENDIF(HAVE_C_DLOPEN)
ENDIF(HAVE_STD_DLOPEN)

include_directories(
  ${CMAKE_CURRENT_BINARY_DIR}
  ${CMAKE_CURRENT_SOURCE_DIR}
  )

if(BUILD_SHARED_LIBS)
  set(tinyscheme_srcs scheme.c dynload.c)
  add_library(tinyscheme SHARED ${tinyscheme_srcs})
  if(SYS_LIBS)
    target_link_libraries(tinyscheme ${SYS_LIBS})
  endif(SYS_LIBS)
endif(BUILD_SHARED_LIBS)
if(BUILD_STATIC_LIBS)
  add_library(tinyscheme_static STATIC scheme.c)
  if(NOT MSVC)
    set_target_properties(tinyscheme-static PROPERTIES OUTPUT_NAME "tinyscheme")
  endif(NOT MSVC)
endif(BUILD_STATIC_LIBS)

if(ENABLE_TINYSCHEME_REPL)
  add_executable(scheme ${tinyscheme_srcs})
  if(SYS_LIBS)
    target_link_libraries(scheme ${SYS_LIBS})
  endif(SYS_LIBS)
endif(ENABLE_TINYSCHEME_REPL)

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/init.scm ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/init.scm COPYONLY)

mark_as_advanced(EXECUTABLE_OUTPUT_PATH)
mark_as_advanced(LIBRARY_OUTPUT_PATH)

# Local Variables:
# tab-width: 8
# mode: cmake
# indent-tabs-mode: t
# End:
# ex: shiftwidth=2 tabstop=8
