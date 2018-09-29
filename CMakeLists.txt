cmake_minimum_required(VERSION 3.12.0)

project(CFITSIO
  VERSION 3.41.0
  LANGUAGES C
)

# Allow the developer to select if Dynamic or Static libraries are built
option(BUILD_SHARED_LIBS "Build Shared Libraries" ON)
option(USE_PTHREADS "Thread-safe build (using pthreads)" OFF)
option(BUILD_TOOLS "Build commandline tools" OFF)
option(BUILD_TESTS "Build unit tests of the library" OFF)

list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}")
# set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${PROJECT_SOURCE_DIR}")
# set (LIB_DESTINATION "${CMAKE_INSTALL_PREFIX}/lib${LIB_SUFFIX}")

# Define project version
# set(LIB_NAME cfitsio)

# Microsoft Visual Studio:
if(MSVC OR BORLAND)
  # Define
  add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
  # Need an empty unistd.h in MSVC for flex-generated eval_l.c:
  file(WRITE ${PROJECT_SOURCE_DIR}/unistd.h "")
  include_directories(${PROJECT_SOURCE_DIR})
endif()

if(BORLAND)
  # Suppress spurious Borland compiler warnings about "Suspicious
  # pointer arithmetic", "Possibly incorrect assignment", and
  # "Comparing signed and unsigned values".
  add_definitions(-w-spa)
  add_definitions(-w-pia)
  add_definitions(-w-csu)
endif()

#add_subdirectory (src)

set(LIB_TYPE STATIC)
if(BUILD_SHARED_LIBS)
  set(LIB_TYPE SHARED)
endif()


set(H_FILES
  cfortran.h drvrgsiftp.h drvrsmem.h eval_defs.h eval_tab.h f77_wrap.h fitsio.h
  fitsio2.h fpack.h group.h grparser.h longnam.h region.h simplerng.h)

if(USE_PTHREADS)
  find_package(pthreads REQUIRED)
  include_directories(${PTHREADS_INCLUDE_DIR})
  add_definitions(-D_REENTRANT)
endif()

# Math library (not available in MSVC or MINGW)
if(MSVC OR MINGW)
  set(M_LIB "")
else()
  find_library(M_LIB m)
endif()

set(SRC_FILES
  buffers.c cfileio.c checksum.c drvrfile.c drvrmem.c drvrnet.c drvrsmem.c
  drvrgsiftp.c editcol.c edithdu.c eval_l.c eval_y.c eval_f.c fitscore.c
  getcol.c getcolb.c getcold.c getcole.c getcoli.c getcolj.c getcolk.c getcoll.c
  getcols.c getcolsb.c getcoluk.c getcolui.c getcoluj.c getkey.c group.c
  grparser.c histo.c iraffits.c modkey.c putcol.c putcolb.c putcold.c putcole.c
  putcoli.c putcolj.c putcolk.c putcoluk.c putcoll.c putcols.c putcolsb.c
  putcolu.c putcolui.c putcoluj.c putkey.c region.c scalnull.c swapproc.c
  wcssub.c wcsutil.c imcompress.c quantize.c ricecomp.c pliocomp.c
  fits_hcompress.c fits_hdecompress.c zlib/zuncompress.c zlib/zcompress.c
  zlib/adler32.c zlib/crc32.c zlib/inffast.c zlib/inftrees.c zlib/trees.c
  zlib/zutil.c zlib/deflate.c zlib/infback.c zlib/inflate.c zlib/uncompr.c
  simplerng.c f77_wrap1.c f77_wrap2.c f77_wrap3.c f77_wrap4.c
)

find_package(CURL REQUIRED)

add_library(cfitsio ${LIB_TYPE} ${H_FILES} ${SRC_FILES})
add_library(cfitsio::cfitsio ALIAS cfitsio)
target_include_directories(cfitsio
  PUBLIC
    $<INSTALL_INTERFACE:include/>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
)

target_link_libraries(cfitsio
  PUBLIC
    ${M_LIB}
  PRIVATE
    ${PTHREADS_LIBRARY}
    CURL::libcurl
)

if(CURL_FOUND)
  target_compile_definitions(cfitsio
    PRIVATE
      unix CFITSIO_HAVE_CURL HAVE_NET_SERVICES
  )
endif()

set_target_properties(cfitsio PROPERTIES
  VERSION ${PROJECT_VERSION}
  SOVERSION ${CMAKE_MAJOR_VERSION}
)

include(GNUInstallDirs)

# set(LIB_DESTINATION "${CMAKE_INSTALL_PREFIX}/lib${LIB_SUFFIX}")
# set(INCLUDE_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/include/")
set(INSTALL_CONFIGDIR ${CMAKE_INSTALL_LIBDIR}/cmake/cfitsio)
install(TARGETS cfitsio
  EXPORT cfitsioTargets
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

install(EXPORT cfitsioTargets
  NAMESPACE    cfitsio::
  DESTINATION  ${INSTALL_CONFIGDIR}
)

install(FILES ${H_FILES} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

include(CMakePackageConfigHelpers)
configure_package_config_file(
  ${CMAKE_CURRENT_LIST_DIR}/cmake/cfitsioConfig.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/cfitsioConfig.cmake
  INSTALL_DESTINATION ${INSTALL_CONFIGDIR}
)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/cfitsioConfig.cmake
  DESTINATION ${INSTALL_CONFIGDIR}
)

##############################################
## Exporting from the build tree
export(EXPORT cfitsioTargets
  FILE ${CMAKE_CURRENT_BINARY_DIR}/cfitsioTargets.cmake
  NAMESPACE cfitsio::
)

#Register package in user's package registry
export(PACKAGE cfitsio)


if(BUILD_TESTS)
  enable_testing()
  add_executable(TestProg testprog.c)
  target_link_libraries(TestProg cfitsio)
  add_test(TestProg TestProg)
  # Copy testprog.tpt to build directory to allow quick test
  # of ./TestProg (or .\Release\TestProg.exe in MSVC):
  file(COPY ${PROJECT_SOURCE_DIR}/testprog.tpt
    DESTINATION ${CMAKE_CURRENT_BINARY_DIR})
endif()

if(BUILD_TOOLS)
  add_executable(cookbook cookbook.c)
  target_link_libraries(cookbook cfitsio)
  add_test(cookbook cookbook)

  add_executable(FPack fpack.c fpackutil.c)
  target_link_libraries(FPack cfitsio)

  add_executable(Funpack funpack.c fpackutil.c)
  target_link_libraries(Funpack cfitsio)

  add_executable(Fitscopy fitscopy.c)
  target_link_libraries(Fitscopy cfitsio)

  # To expands the command line arguments in Windows, see:
  # http://msdn.microsoft.com/en-us/library/8bch7bkk.aspx
  if(MSVC)
    set_target_properties(FPack Funpack PROPERTIES LINK_FLAGS "setargv.obj")
  endif()
endif()
