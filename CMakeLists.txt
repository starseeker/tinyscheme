# This file contains the top level CMakeLists.txt logic for the
# TinyScheme software package.

# Minimum required version of CMake
cmake_minimum_required(VERSION 2.8.3)

# Project name
project(TINYSCHEME)

# Version information
set(TINYSCHEME_VERSION 1.41)

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

TS_OPT(MATH ON "Includes math routines.")
TS_OPT(CHAR_CLASSIFIERS ON "Includes character classifier procedures.")
TS_OPT(ASCII_NAMES ON "Enable extended character notation based on ASCII names.")
TS_OPT(STRING_PORTS ON "Enables string ports.")
TS_OPT(ERROR_HOOK OFF "Force system errors through user-defined error handling.")
TS_OPT(TRACING OFF "Enable the use of TRACING.")
TS_OPT(COLON_HOOK OFF "Enable use of qualified identifiers.  (see \"Colon Qualifiers - Packages\") It should only be used if you *absolutely* need to use '::' in identifiers.")
TS_OPT(PLIST OFF "Enables property lists (not Standard Scheme stuff).")
TS_OPT(STRCASECMP OFF "Defines stricmp as strcasecmp, for Unix.")
#TS_OPT(SCHEME_STACK OFF "Enables 'cons' stack. (the alternative is a faster calling scheme, which breaks continuations). Undefine it if you don't care about strict compatibility but you do care about faster execution.")

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
check_library_exists(m cos "" HAVE_M_LIBRARY)
if(HAVE_M_LIBRARY)
  set(SYS_LIBS ${SYS_LIBS} m)
endif(HAVE_M_LIBRARY)
check_library_exists(dl dlopen "" HAVE_STD_DLOPEN)
if(HAVE_STD_DLOPEN)
  set(SYS_LIBS ${SYS_LIBS} dl)
else(HAVE_STD_DLOPEN)
  check_library_exists(c dlopen "" HAVE_C_DLOPEN)
  if(HAVE_C_DLOPEN)
    set(SYS_LIBS ${SYS_LIBS} c)
  endif(HAVE_C_DLOPEN)
endif(HAVE_STD_DLOPEN)

include_directories(
  ${CMAKE_CURRENT_BINARY_DIR}
  ${CMAKE_CURRENT_SOURCE_DIR}
  )

# Rather than load the init.scm file at run time,
# with the uncertainties as to where exactly the file
# resides, use William Ahern's hexdump to generate
# an embeddable version. Build our own copy of hexdump
# to ensure consistent behavior and portability.
# See http://25thandclement.com/~william/projects/hexdump.c.html
add_executable(initfiletohex hexdump.c)
set_property(TARGET initfiletohex APPEND PROPERTY COMPILE_DEFINITIONS "HEXDUMP_MAIN")
add_custom_command(
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/init_scm.h
  COMMAND initfiletohex -i ${CMAKE_CURRENT_SOURCE_DIR}/init.scm > ${CMAKE_CURRENT_BINARY_DIR}/init_scm.h
  DEPENDS initfiletohex ${CMAKE_CURRENT_SOURCE_DIR}/init.scm
  )
add_custom_target(init_hdr ALL DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/init_scm.h)


if(BUILD_SHARED_LIBS)
  set(tinyscheme_srcs scheme.c dynload.c)
  add_library(tinyscheme SHARED ${tinyscheme_srcs})
  if(SYS_LIBS)
    target_link_libraries(tinyscheme ${SYS_LIBS})
  endif(SYS_LIBS)
  add_dependencies(tinyscheme init_hdr)
endif(BUILD_SHARED_LIBS)
if(BUILD_STATIC_LIBS)
  add_library(tinyscheme-static STATIC scheme.c)
  if(NOT MSVC)
    set_target_properties(tinyscheme-static PROPERTIES OUTPUT_NAME "tinyscheme")
  endif(NOT MSVC)
  add_dependencies(tinyscheme-static init_hdr)
endif(BUILD_STATIC_LIBS)

if(ENABLE_TINYSCHEME_REPL)
  add_executable(scheme ${tinyscheme_srcs})
  set_property(TARGET ${libname} APPEND PROPERTY COMPILE_DEFINITIONS "STANDALONE=1")
  if(SYS_LIBS)
    target_link_libraries(scheme ${SYS_LIBS})
  endif(SYS_LIBS)
  add_dependencies(scheme init_hdr)
endif(ENABLE_TINYSCHEME_REPL)

mark_as_advanced(EXECUTABLE_OUTPUT_PATH)
mark_as_advanced(LIBRARY_OUTPUT_PATH)

# Local Variables:
# tab-width: 8
# mode: cmake
# indent-tabs-mode: t
# End:
# ex: shiftwidth=2 tabstop=8
