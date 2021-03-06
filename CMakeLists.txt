cmake_minimum_required(VERSION 2.8.10.1)
project(QtBuilder)
  #
  # Enable C and CXX languages so CMAKE_SIZEOF_VOID_P indicates whether
  # this is a 64-bit build or not. (Otherwise, we could use 'NONE' for
  # languages in the project command because this project is built entirely
  # with ExternalProject custom build steps...)
  #

include(ExternalProject)
include(CTest)


function(qb_get_qt_version_info version var_tarball_name var_tarball_md5)
  string(REGEX REPLACE "[^A-Za-z0-9_]" "" var_suffix "${version}")

  if(var_suffix LESS 460)
    set(tarball_basename "qt-all-opensource-src-")
  else()
    set(tarball_basename "qt-everywhere-opensource-src-")
  endif()

  set(qt_tgz_md5_464 "8ac880cc07a130c39607b65efd5e1421")
  set(qt_tgz_md5_481 "7960ba8e18ca31f0c6e4895a312f92ff")
  set(qt_tgz_md5_482 "3c1146ddf56247e16782f96910a8423b")
  set(qt_tgz_md5_483 "a663b6c875f8d7caa8ac9c30e4a4ec3b")
  set(qt_tgz_md5_484 "89c5ecba180cae74c66260ac732dc5cb")
  set(qt_tgz_md5_500 "1a1b73f913a965f6b0cf02bb820c8a03")

  set(tarball_name "${tarball_basename}${version}.tar.gz")
  set(tarball_md5 "${qt_tgz_md5_${var_suffix}}")

  message("version='${version}'")
  message("var_tarball_name='${var_tarball_name}'")
  message("var_tarball_md5='${var_tarball_md5}'")
  message("var_suffix='${var_suffix}'")
  message("tarball_name='${tarball_name}'")
  message("tarball_md5='${tarball_md5}'")

  set(${var_tarball_name} "${tarball_name}" PARENT_SCOPE)
  set(${var_tarball_md5} "${tarball_md5}" PARENT_SCOPE)
endfunction()


set(QB_VERSION "5.0.0" CACHE STRING
  "Version of Qt to download, build and install")

set(VersionValues
  4.6.4
  4.8.1
  4.8.2
  4.8.3
  4.8.4
  5.0.0
)

set_property(CACHE QB_VERSION PROPERTY STRINGS ${VersionValues})

if(WIN32)
  set(QB_BASEDIR_DEFAULT "C:/Qt")
else()
  set(QB_BASEDIR_DEFAULT "$ENV{HOME}/Qt")
endif()

set(QB_BASEDIR "${QB_BASEDIR_DEFAULT}" CACHE PATH
  "Local base directory for downloading and building")

set(qt_server "http://releases.qt-project.org")
set(qt_version "${QB_VERSION}")
qb_get_qt_version_info("${qt_version}" qt_tgz_name qt_tgz_md5)

if(qt_version VERSION_LESS 5.0.0)
  set(qt_baseurl "${qt_server}/qt4/source")
else()
  set(qt_baseurl "${qt_server}/qt5/${qt_version}/single")
endif()

set(qt_md5 "${qt_tgz_md5}")
set(qt_url "${qt_baseurl}/${qt_tgz_name}")

if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/${qt_tgz_name}")
  set(qt_url "${CMAKE_CURRENT_SOURCE_DIR}/${qt_tgz_name}")
endif()

set(suffix "unsupported")

if(MSVC11)
  set(suffix "vs11")
elseif(MSVC10)
  set(suffix "vs10")
elseif(MSVC90)
  set(suffix "vs2008")
elseif(MSVC80)
  set(suffix "vs2005")
elseif(MSVC71)
  set(suffix "vs2003")
elseif(CMAKE_COMPILER_IS_GNUCC)
  set(suffix "gcc")
endif()

if(CMAKE_SIZEOF_VOID_P EQUAL 8)
  set(suffix "${suffix}-x64")
endif()


if(WIN32)
#  if(NOT CMAKE_GENERATOR MATCHES "Makefile")
#    message(FATAL_ERROR "
# error: building Qt on Windows requires using a Makefile generator
#
#  The Visual Studio generators are not supported.
#  Run cmake with -G \"NMake Makefiles\" or -G \"NMake Makefiles JOM\"
#  from a Visual Studio command prompt...
# ")
#  endif()

  #
  # Windows builds with nmake/jom are "in-source/in-place" builds.
  #   i.e. source == build == install dir
  #   and prefix is irrelevant...
  #
  set(qt_dir "${QB_BASEDIR}/${qt_version}-${suffix}")
  set(prefix_arg "")
  set(other_args -opensource -confirm-license
    -nomake demos -nomake examples
    -shared -release
  )
else()

  #
  # Linux/Mac builds with make honor --prefix. They do not necessarily have to
  # be "in-source/in-place" builds, although they can be... So we do in-source
  # here for consistency with the Windows builds.
  #
  set(qt_dir "${QB_BASEDIR}/${qt_version}-${suffix}")
  set(prefix_arg --prefix=${qt_dir})
  set(other_args --opensource --confirm-license
    -nomake demos -nomake examples
    -shared -release
  )

  if(APPLE)
    set(other_args ${other_args} -cocoa)
    if("$ENV{QMAKE_MACOSX_DEPLOYMENT_TARGET}" STREQUAL "")
      message(AUTHOR_WARNING "Environment variable QMAKE_MACOSX_DEPLOYMENT_TARGET is not set. Current OS version will be the default deployment target.")
    endif()
  endif()
endif()

# CACHE so users can add-to/modify the Qt configure args if necessary:
#
set(QB_CONFIGURE_ARGS ${prefix_arg} ${other_args} CACHE STRING
  "Args for Qt configure step")

set(md5_args "")
if(qt_md5)
  set(md5_args URL_MD5 ${qt_md5})
endif()


if(WIN32)
  set(build_cmd "jom")
else()
  set(build_cmd "$(MAKE)")
endif()


ExternalProject_Add(qt
  URL ${qt_url}
  ${md5_args}
  DOWNLOAD_DIR ${QB_BASEDIR}/Downloads
  SOURCE_DIR ${qt_dir}
  BUILD_IN_SOURCE 1
  CONFIGURE_COMMAND <SOURCE_DIR>/configure ${QB_CONFIGURE_ARGS}
  BUILD_COMMAND ${build_cmd}
  INSTALL_COMMAND ""
  STEP_TARGETS download
)


message(STATUS "suffix='${suffix}'")
message(STATUS "qt_dir='${qt_dir}'")
message(STATUS "qt_url='${qt_url}'")

add_test(SmokeTest-Qt-${qt_version}-moc "${qt_dir}/bin/moc" "-v")
add_test(SmokeTest-Qt-${qt_version}-qmake "${qt_dir}/bin/qmake" "-v")
add_test(SmokeTest-Qt-${qt_version}-uic "${qt_dir}/bin/uic" "-v")

set_property(TEST SmokeTest-Qt-${qt_version}-moc PROPERTY
  PASS_REGULAR_EXPRESSION "Meta Object Compiler")
