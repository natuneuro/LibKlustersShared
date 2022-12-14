project(libklustersshared)
cmake_minimum_required(VERSION 2.8.6)
set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules ${CMAKE_MODULE_PATH})
MESSAGE(STATUS "${CMAKE_MODULE_PATH}")

set (QT_MIN_VERSION "4.8.0")

option(
  ENFORCE_QT4_BUILD
  "Enable if you want to enfore a build with Qt4"
  OFF
)

# try Qt5 first, and prefer that (if found), but only if not disabled via option
#if(NOT ENFORCE_QT4_BUILD)
  find_package(Qt5Core QUIET)
#endif()

if(Qt5Core_FOUND)
  find_package(Qt5Gui REQUIRED)
  find_package(Qt5Widgets REQUIRED)
  find_package(Qt5PrintSupport REQUIRED)
  find_package(Qt5WebKitWidgets 5.0.2 QUIET)

  include(ECMQt4To5Porting)
  include_directories(${QT_INCLUDES}) # TODO: Port away from this.

  set(QT_QTWEBKIT_FOUND ${Qt5WebKitWidgets_FOUND}) # backward compat with Qt4 webkit code

  if(Qt5_POSITION_INDEPENDENT_CODE)
    if(CMAKE_VERSION VERSION_LESS 2.8.9) # TODO remove once we increase the cmake requirement
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")
    else()
      set(CMAKE_POSITION_INDEPENDENT_CODE ON)
    endif()
  endif()
  MESSAGE(STATUS "use qt5 version")
# Qt4
else()
  MESSAGE(STATUS "use qt4 version")
  find_package(Qt4 ${QT_MIN_VERSION} REQUIRED QtCore QtGui)

  include(${QT_USE_FILE})
  include_directories(${QT_INCLUDES})
endif()


add_definitions(${QT_DEFINITIONS} )

include(MacroEnsureVersion)
include(CheckCXXCompilerFlag)
include(MacroWriteBasicCMakeVersionFile)

set(LIBKLUSTERSSHARED_VERSION_MAJOR 2)
set(LIBKLUSTERSSHARED_VERSION_MINOR 0)
set(LIBKLUSTERSSHARED_VERSION_PATCH 0)
set(LIBKLUSTERSSHARED_VERSION ${LIBKLUSTERSSHARED_VERSION_MAJOR}.${LIBKLUSTERSSHARED_VERSION_MINOR}.${LIBKLUSTERSSHARED_VERSION_PATCH})

# this file will be installed too and will be used by cmake when searching for the Config.cmake file to check the version of kdepimlibs, Alex
macro_write_basic_cmake_version_file(${CMAKE_CURRENT_BINARY_DIR}/LibKlustersSharedConfigVersion.cmake
	${LIBKLUSTERSSHARED_VERSION_MAJOR} ${LIBKLUSTERSSHARED_VERSION_MINOR} ${LIBKLUSTERSSHARED_VERSION_PATCH})


# This macro is for internal use only
# Return the directories present in gcc's include path.
macro(_DETERMINE_GCC_SYSTEM_INCLUDE_DIRS _lang _result)
  set(${_result})
  set(_gccOutput)
  file(WRITE "${CMAKE_BINARY_DIR}/CMakeFiles/dummy" "\n" )
  execute_process(COMMAND ${CMAKE_C_COMPILER} -v -E -x ${_lang} -dD dummy
                  WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/CMakeFiles
                  ERROR_VARIABLE _gccOutput
                  OUTPUT_VARIABLE _gccStdout )
  file(REMOVE "${CMAKE_BINARY_DIR}/CMakeFiles/dummy")

  if( "${_gccOutput}" MATCHES "> search starts here[^\n]+\n *(.+) *\n *End of (search) list" )
    SET(${_result} ${CMAKE_MATCH_1})
    STRING(REPLACE "\n" " " ${_result} "${${_result}}")
    SEPARATE_ARGUMENTS(${_result})
  ENDIF( "${_gccOutput}" MATCHES "> search starts here[^\n]+\n *(.+) *\n *End of (search) list" )
ENDMACRO(_DETERMINE_GCC_SYSTEM_INCLUDE_DIRS _lang)

#laurent check from kde
if (CMAKE_COMPILER_IS_GNUCXX)
  if (NOT APPLE)
    set ( CMAKE_SHARED_LINKER_FLAGS "-Wl,--fatal-warnings -Wl,--no-undefined  ${CMAKE_SHARED_LINKER_FLAGS}")
    set ( CMAKE_MODULE_LINKER_FLAGS "-Wl,--fatal-warnings -Wl,--no-undefined  ${CMAKE_MODULE_LINKER_FLAGS}")
  endif ()

   _DETERMINE_GCC_SYSTEM_INCLUDE_DIRS(c++ _dirs)
   set(CMAKE_CXX_IMPLICIT_INCLUDE_DIRECTORIES
       ${CMAKE_CXX_IMPLICIT_INCLUDE_DIRECTORIES} ${_dirs})

   set (KDE4_ENABLE_EXCEPTIONS "-fexceptions -UQT_NO_EXCEPTIONS")
   # Select flags.
   set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-O2 -g -DNDEBUG -DQT_NO_DEBUG")
   set(CMAKE_CXX_FLAGS_RELEASE        "-O2 -DNDEBUG -DQT_NO_DEBUG")
   set(CMAKE_CXX_FLAGS_DEBUG          "-g -O2 -fno-reorder-blocks -fno-schedule-insns -fno-inline")
   set(CMAKE_CXX_FLAGS_DEBUGFULL      "-g3 -fno-inline")
   set(CMAKE_CXX_FLAGS_PROFILE        "-g3 -fno-inline -ftest-coverage -fprofile-arcs")
   set(CMAKE_C_FLAGS_RELWITHDEBINFO   "-O2 -g -DNDEBUG -DQT_NO_DEBUG")
   set(CMAKE_C_FLAGS_RELEASE          "-O2 -DNDEBUG -DQT_NO_DEBUG")
   set(CMAKE_C_FLAGS_DEBUG            "-g -O2 -fno-reorder-blocks -fno-schedule-insns -fno-inline")
   set(CMAKE_C_FLAGS_DEBUGFULL        "-g3 -fno-inline")
   set(CMAKE_C_FLAGS_PROFILE          "-g3 -fno-inline -ftest-coverage -fprofile-arcs")

  check_cxx_compiler_flag(-fvisibility=hidden __KDE_HAVE_GCC_VISIBILITY)
  set( __KDE_HAVE_GCC_VISIBILITY ${__KDE_HAVE_GCC_VISIBILITY} CACHE BOOL "GCC support for hidden visibility")

   # get the gcc version
   exec_program(${CMAKE_C_COMPILER} ARGS ${CMAKE_C_COMPILER_ARG1} --version OUTPUT_VARIABLE _gcc_version_info)

   string (REGEX MATCH "[345]\\.[0-9]\\.[0-9]" _gcc_version "${_gcc_version_info}")
   # gcc on mac just reports: "gcc (GCC) 3.3 20030304 ..." without the patch level, handle this here:
   if (NOT _gcc_version)
      string (REGEX MATCH ".*\\(GCC\\).* ([34]\\.[0-9]) .*" "\\1.0" _gcc_version "${gcc_on_macos}")
      if (gcc_on_macos)
        string (REGEX REPLACE ".*\\(GCC\\).* ([34]\\.[0-9]) .*" "\\1.0" _gcc_version "${_gcc_version_info}")
      endif (gcc_on_macos)
   endif (NOT _gcc_version)

   if (_gcc_version)
      macro_ensure_version("4.1.0" "${_gcc_version}" GCC_IS_NEWER_THAN_4_1)
      macro_ensure_version("4.2.0" "${_gcc_version}" GCC_IS_NEWER_THAN_4_2)
      macro_ensure_version("4.3.0" "${_gcc_version}" GCC_IS_NEWER_THAN_4_3)
   endif (_gcc_version)

   set(_GCC_COMPILED_WITH_BAD_ALLOCATOR FALSE)
   if (GCC_IS_NEWER_THAN_4_1)
      exec_program(${CMAKE_C_COMPILER} ARGS ${CMAKE_C_COMPILER_ARG1} -v OUTPUT_VARIABLE _gcc_alloc_info)
      string(REGEX MATCH "(--enable-libstdcxx-allocator=mt)" _GCC_COMPILED_WITH_BAD_ALLOCATOR "${_gcc_alloc_info}")
   endif (GCC_IS_NEWER_THAN_4_1)

   if (__KDE_HAVE_GCC_VISIBILITY AND GCC_IS_NEWER_THAN_4_1 AND NOT _GCC_COMPILED_WITH_BAD_ALLOCATOR AND NOT WIN32)
      set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility=hidden")
      set (KDE4_C_FLAGS "-fvisibility=hidden")
      # check that Qt defines Q_DECL_EXPORT as __attribute__ ((visibility("default")))
      # if it doesn't and KDE compiles with hidden default visibiltiy plugins will break
      set(_source "#include <QtCore/QtGlobal>\n int main()\n {\n #ifndef QT_VISIBILITY_AVAILABLE \n #error QT_VISIBILITY_AVAILABLE is not available\n #endif \n }\n")
      set(_source_file ${CMAKE_BINARY_DIR}/CMakeTmp/check_qt_visibility.cpp)
      file(WRITE "${_source_file}" "${_source}")
      set(_include_dirs "-DINCLUDE_DIRECTORIES:STRING=${QT_INCLUDES}")

      try_compile(_compile_result ${CMAKE_BINARY_DIR} ${_source_file} CMAKE_FLAGS "${_include_dirs}" COMPILE_OUTPUT_VARIABLE _compile_output_var)

      if(NOT _compile_result)
         message(FATAL_ERROR "Qt compiled without support for -fvisibility=hidden. This will break plugins and linking of some applications. Please fix your Qt installation.")
      endif(NOT _compile_result)

      if (GCC_IS_NEWER_THAN_4_2)
          set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility-inlines-hidden")
          MESSAGE(STATUS "compile with visibility hidden")
      endif (GCC_IS_NEWER_THAN_4_2)
   else (__KDE_HAVE_GCC_VISIBILITY AND GCC_IS_NEWER_THAN_4_1 AND NOT _GCC_COMPILED_WITH_BAD_ALLOCATOR AND NOT WIN32)
      set (__KDE_HAVE_GCC_VISIBILITY 0)
   endif (__KDE_HAVE_GCC_VISIBILITY AND GCC_IS_NEWER_THAN_4_1 AND NOT _GCC_COMPILED_WITH_BAD_ALLOCATOR AND NOT WIN32)

endif()




if (MSVC)
   set (KDE4_ENABLE_EXCEPTIONS -EHsc)
   # Qt disables the native wchar_t type, do it too to avoid linking issues
   set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Zc:wchar_t-" )

endif()

# tell cmake where to search for Qt headers:
include_directories(${QT_INCLUDE_DIR} ${CMAKE_CURRENT_BINARY_DIR} src/gui/ ${CMAKE_CURRENT_BINARY_DIR}/src/gui/ src/libinqt5)

set(LIB_SUFFIX "" CACHE STRING "Define suffix of directory name (32/64)" )
set(EXEC_INSTALL_PREFIX  ${CMAKE_INSTALL_PREFIX}       CACHE PATH  "Base directory for executables and libraries")
#
## the following are directories where stuff will be installed to
set(BIN_INSTALL_DIR          "${EXEC_INSTALL_PREFIX}/bin" CACHE PATH "The binary install dir (default prefix/bin)")
set(LIB_INSTALL_DIR          "${EXEC_INSTALL_PREFIX}/lib${LIB_SUFFIX}"     CACHE PATH "The subdirectory relative to the install prefix where libraries will be installed (default is /lib${LIB_SUFFIX})")
set(INCLUDE_INSTALL_DIR          "${EXEC_INSTALL_PREFIX}/include/klustersshared"     CACHE PATH "The subdirectory relative to the install prefix where includes will be installed ")

set(CMAKE_INCLUDE_CURRENT_DIR TRUE)
set(CMAKE_AUTOMOC TRUE)
set( QT_USE_QTWEBKIT TRUE)


if(QT_QTWEBKIT_FOUND)
   include_directories(${QT_QTWEBKIT_INCLUDE_DIR})
else(QT_QTWEBKIT_FOUND)
   MESSAGE(STATUS "qtwebkit not found. Please install them")
endif()

configure_file(LibKlustersSharedConfig.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/LibKlustersSharedConfig.cmake @ONLY)

add_definitions(-DKLUSTERSSHARED_BUILD_KLUSTERSSHARED_LIB)
add_definitions(-DQT_USE_FAST_CONCATENATION -DQT_USE_FAST_OPERATOR_PLUS)

add_subdirectory(src)

# install the file with the exported targets
install(EXPORT LibKlustersSharedLibraryTargets
        DESTINATION ${LIB_INSTALL_DIR}/cmake/LibKlustersShared
        FILE LibKlustersSharedTargetsWithPrefix.cmake
        )

install(FILES
	${CMAKE_CURRENT_BINARY_DIR}/LibKlustersSharedConfig.cmake ${CMAKE_CURRENT_BINARY_DIR}/LibKlustersSharedConfigVersion.cmake
  	DESTINATION ${LIB_INSTALL_DIR}/cmake/LibKlustersShared  COMPONENT Devel)

INCLUDE(InstallRequiredSystemLibraries)

set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Library for Klusters, Neuroscope and NDManager")
SET(CPACK_PACKAGE_NAME "libklustersshared")
SET(CPACK_PACKAGE_NAME_SIMPLIFIED "libklustersshared")
SET(CPACK_PACKAGE_VERSION_MAJOR "${LIBKLUSTERSSHARED_VERSION_MAJOR}")
SET(CPACK_PACKAGE_VERSION_MINOR "${LIBKLUSTERSSHARED_VERSION_MINOR}")
SET(CPACK_PACKAGE_VERSION_PATCH "${LIBKLUSTERSSHARED_VERSION_PATCH}")
SET(CPACK_INSTALL_DIRECTORY "${CMAKE_INSTALL_PREFIX}")

SET(CPACK_PACKAGE_VERSION "${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${LIBKLUSTERSSHARED_VERSION_PATCH}")
# GET_FILENAME_COMPONENT(CPACK_INSTALLED_DIRECTORIES "${CPACK_INSTALL_DIRECTORY}" REALPATH)
# LIST(APPEND CPACK_INSTALLED_DIRECTORIES ".")

SET(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME_SIMPLIFIED}-${CPACK_PACKAGE_VERSION}-source")
SET(CPACK_PACKAGE_NAME_AND_VERSION "${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}-Source")
SET(CPACK_PACKAGE_INSTALL_DIRECTORY "${CPACK_PACKAGE_FILE_NAME}")

IF(WIN32)
        SET(CPACK_GENERATOR "NSIS")
        SET(CPACK_NSIS_DISPLAY_NAME "${CPACK_PACKAGE_NAME_AND_VERSION}")
        SET(CPACK_NSIS_PACKAGE_NAME "${CPACK_PACKAGE_NAME_AND_VERSION}")
        SET(CPACK_PACKAGE_INSTALL_REGISTRY_KEY "${CPACK_PACKAGE_NAME_AND_VERSION}")
        SET(CPACK_NSIS_INSTALL_ROOT "libklustersshared") # use "$PROGRAMFILES" to get "C:\Program Files" or similar
        SET(CPACK_PACKAGE_INSTALL_DIRECTORY "${CPACK_PACKAGE_NAME_AND_VERSION}")
ELSEIF(APPLE)
        SET(CPACK_GENERATOR "ZIP")
        SET(CPACK_SYSTEM_NAME "OSX")
ELSE()
        SET(CPACK_GENERATOR "TBZ2")

        # Determine distribution and release
        execute_process(COMMAND lsb_release -si OUTPUT_VARIABLE DISTRIBUTION OUTPUT_STRIP_TRAILING_WHITESPACE)
        execute_process(COMMAND lsb_release -sc OUTPUT_VARIABLE RELEASE OUTPUT_STRIP_TRAILING_WHITESPACE)
        execute_process(COMMAND uname -i OUTPUT_VARIABLE ARCHITECTURE)

        # Set package name and type (deb vs rpm)
        if(DISTRIBUTION STREQUAL "Ubuntu")

          # Set Ubuntu-specific information (see http://www.cmake.org/Wiki/CMake:CPackPackageGenerators)
          if(ARCHITECTURE MATCHES ".*x86_64.*")
            set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE "amd64")
          else()
            set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE "i386")
          endif()
          set(CPACK_DEBIAN_PACKAGE_MAINTAINER "Micha??l Zugaro")
          set(CPACK_DEBIAN_PACKAGE_SECTION "Science")
          set(CPACK_DEBIAN_PACKAGE_RECOMMENDS "nphys-data, klusters, neuroscope, ndmanager")
          set(CPACK_DEBIAN_PACKAGE_DEPENDS "libqt4-network, libqtcore4, libqtgui4, libqtwebkit4")
          set(CPACK_DEBIAN_PACKAGE_HOMEPAGE "http://klusters.sourceforge.net")
          set(CPACK_GENERATOR "DEB")
          set(CPACK_PACKAGE_FILE_NAME ${CPACK_PACKAGE_NAME}_${CPACK_PACKAGE_VERSION}-${RELEASE}_${CPACK_DEBIAN_PACKAGE_ARCHITECTURE})

        # >>> TEMPLATE CODE FOR ADDITIONAL DEBIAN-LIKE DISTRIBUTIONS <<<

        #elseif(DISTRIBUTION STREQUAL "Ubuntu")
        #
        #  # Set Ubuntu-specific information (see http://www.cmake.org/Wiki/CMake:CPackPackageGenerators)
        #  if(ARCHITECTURE MATCHES ".*x86_64.*")
        #    set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE "amd64")
        #  else()
        #    set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE "i386")
        #  endif()
        #  set(CPACK_DEBIAN_PACKAGE_MAINTAINER "Micha??l Zugaro")
        #  set(CPACK_DEBIAN_PACKAGE_SECTION "Science")
        #  set(CPACK_DEBIAN_PACKAGE_RECOMMENDS "nphys-data, klusters, neuroscope, ndmanager")
        #  set(CPACK_DEBIAN_PACKAGE_DEPENDS "libqt4-network, libqtcore4, libqtgui4, libqtwebkit4")
        #  set(CPACK_DEBIAN_PACKAGE_HOMEPAGE "http://klusters.sourceforge.net")
        #  set(CPACK_GENERATOR "DEB")
        #  set(CPACK_PACKAGE_FILE_NAME ${CPACK_PACKAGE_NAME}_${CPACK_PACKAGE_VERSION}-${RELEASE}_${CPACK_DEBIAN_PACKAGE_ARCHITECTURE})

        elseif(DISTRIBUTION MATCHES "openSUSE.*")

          # Set openSUSE-specific information (see http://www.cmake.org/Wiki/CMake:CPackPackageGenerators)
          if(ARCHITECTURE MATCHES ".*x86_64.*")
            set(CPACK_RPM_PACKAGE_ARCHITECTURE "x86_64")
          else()
            set(CPACK_RPM_PACKAGE_ARCHITECTURE "i586")
          endif()
          set(CPACK_PACKAGE_CONTACT "Micha??l Zugaro")
          set(CPACK_RPM_PACKAGE_GROUP "Science")
          set(CPACK_RPM_PACKAGE_LICENSE "GPL")
          set(CPACK_RPM_PACKAGE_SUGGESTS "nphys-data, neuroscope, ndmanager")
          set(CPACK_RPM_PACKAGE_REQUIRES "libqt4, libqt4-x11, libQtWebKit4")
          set(CPACK_RPM_PACKAGE_URL "http://klusters.sourceforge.net")
          set(CPACK_GENERATOR "RPM")
          set(CPACK_PACKAGE_FILE_NAME ${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}-${RELEASE}.${CPACK_RPM_PACKAGE_ARCHITECTURE})

        elseif(DISTRIBUTION STREQUAL "Fedora")

          # Set Fedora-specific information (see http://www.cmake.org/Wiki/CMake:CPackPackageGenerators)
          execute_process(COMMAND lsb_release -sr OUTPUT_VARIABLE RELEASE OUTPUT_STRIP_TRAILING_WHITESPACE)
          if(ARCHITECTURE MATCHES ".*x86_64.*")
            set(CPACK_RPM_PACKAGE_ARCHITECTURE "x86_64")
          else()
            set(CPACK_RPM_PACKAGE_ARCHITECTURE "i686")
          endif()
          set(CPACK_PACKAGE_CONTACT "Micha??l Zugaro")
          set(CPACK_RPM_PACKAGE_GROUP "Science")
          set(CPACK_RPM_PACKAGE_LICENSE "GPL")
          set(CPACK_RPM_PACKAGE_SUGGESTS "nphys-data, neuroscope, ndmanager")
          set(CPACK_RPM_PACKAGE_REQUIRES "qt, qt-mobility, qtwebkit, qt-x11")
          set(CPACK_RPM_PACKAGE_URL "http://klusters.sourceforge.net")
          set(CPACK_GENERATOR "RPM")
          set(CPACK_PACKAGE_FILE_NAME ${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}.fc${RELEASE}.${CPACK_RPM_PACKAGE_ARCHITECTURE})

        elseif(DISTRIBUTION MATCHES "Scientific")

          # Set Scientific-Linux-specific information (see http://www.cmake.org/Wiki/CMake:CPackPackageGenerators)
          if(ARCHITECTURE MATCHES ".*x86_64.*")
            set(CPACK_RPM_PACKAGE_ARCHITECTURE "x86_64")
          else()
            set(CPACK_RPM_PACKAGE_ARCHITECTURE "i386")
          endif()
          set(CPACK_PACKAGE_CONTACT "Rajat Mani Thomas")
          set(CPACK_RPM_PACKAGE_GROUP "Science")
          set(CPACK_RPM_PACKAGE_LICENSE "GPL")
          set(CPACK_RPM_PACKAGE_SUGGESTS "nphys-data, neuroscope, ndmanager")
          set(CPACK_RPM_PACKAGE_REQUIRES "libQtCore, libQtGui, libQtNetwork, libQtWebKit")
          set(CPACK_RPM_PACKAGE_URL "http://klusters.sourceforge.net")
          set(CPACK_GENERATOR "RPM")
          set(CPACK_PACKAGE_FILE_NAME ${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}-${RELEASE}.${CPACK_RPM_PACKAGE_ARCHITECTURE})

        # >>> TEMPLATE CODE FOR ADDITIONAL REDHAT-LIKE DISTRIBUTIONS <<<

        #elseif(DISTRIBUTION MATCHES "openSUSE.*")
        #
        #  # Set openSUSE-specific information (see http://www.cmake.org/Wiki/CMake:CPackPackageGenerators)
        #  if(ARCHITECTURE MATCHES ".*x86_64.*")
        #    set(CPACK_RPM_PACKAGE_ARCHITECTURE "x86_64")
        #  else()
        #    set(CPACK_RPM_PACKAGE_ARCHITECTURE "i586")
        #  endif()
        #  set(CPACK_PACKAGE_CONTACT "Micha??l Zugaro")
        #  set(CPACK_RPM_PACKAGE_GROUP "Science")
        #  set(CPACK_RPM_PACKAGE_LICENSE "GPL")
        #  set(CPACK_RPM_PACKAGE_SUGGESTS "nphys-data, neuroscope, ndmanager")
        #  set(CPACK_RPM_PACKAGE_REQUIRES "libqt4, libqt4-x11, libQtWebKit4")
        #  set(CPACK_RPM_PACKAGE_URL "http://klusters.sourceforge.net")
        #  set(CPACK_GENERATOR "RPM")
        #  set(CPACK_PACKAGE_FILE_NAME ${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}-${RELEASE}.${CPACK_RPM_PACKAGE_ARCHITECTURE})

        endif()
ENDIF()


SET(CPACK_TOPLEVEL_TAG "${CPACK_SYSTEM_NAME}")
FILE(TO_CMAKE_PATH "${CPACK_INSTALL_DIRECTORY}" XPLATFORM_INSTALL_DIR) # to get forward slashes on Windows

SET(CPACK_IGNORE_FILES '.git'  "/[._]obj/" "/moc_" "/[._]o/" "/CMakeFiles/" "/[._]cpp" "/[._]depends" "CMakeCache.txt" "Makefile"
                       "^${XPLATFORM_INSTALL_DIR}/install_manifest.txt"
                       "^${XPLATFORM_INSTALL_DIR}/src/qrc_libklustersshared-icons.cxx"
                       "^${XPLATFORM_INSTALL_DIR}/src/klustersshared_automoc.cpp"
                       "^${XPLATFORM_INSTALL_DIR}/src/cmake_install.cmake"
                       "^${XPLATFORM_INSTALL_DIR}/src/gui/libklustersshared-icons.qrc.depends"
                       "^${XPLATFORM_INSTALL_DIR}/src/qpageview_p.moc"
                       "^${XPLATFORM_INSTALL_DIR}/src/CPackSourceConfig.cmake"
                       "^${XPLATFORM_INSTALL_DIR}/CPackSourceConfig.cmake"
                       "^${XPLATFORM_INSTALL_DIR}/CMakeTmp/check_qt_visibility.cpp"
                       "^${XPLATFORM_INSTALL_DIR}/CPackConfig.cmake"
                       "^${XPLATFORM_INSTALL_DIR}/cmake_install.cmake")


include(CPack)
add_subdirectory(test)
