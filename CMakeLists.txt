cmake_minimum_required(VERSION 3.7)

project(pyyaml_vendor)

find_package(ament_cmake REQUIRED)

# PyYAML is the YAML parser/emitter Python module. Its optional C extension
# (_yaml) links libyaml — provided by ros2's libyaml_vendor (see package.xml
# depend). lyrical targets Ubuntu 26.04 (resolute), whose python3-yaml is
# 6.0.3 — pin the upstream tag to match.
#
# PyYAML uses setuptools (not CMake), so we drive setup.py with the VxWorks
# cross-Python env (ported from the former pkg/pyyaml 3PP) and install the
# built module tree into the target site-packages.

set(PYYAML_VERSION 6.0.3)

# These must be provided by the build harness (the vxworks mixin / sdkenv):
#   WIND_CC_SYSROOT, ROOT_DIR (install prefix), TGT_PYTHON_MINOR
if(NOT DEFINED ENV{WIND_CC_SYSROOT})
  message(FATAL_ERROR "pyyaml_vendor: WIND_CC_SYSROOT not set (source sdkenv.sh)")
endif()
set(SYSROOT "$ENV{WIND_CC_SYSROOT}")
if(DEFINED ENV{TGT_PYTHON_MINOR})
  set(PYMINOR "$ENV{TGT_PYTHON_MINOR}")
else()
  set(PYMINOR "9")
endif()
set(PYTAG "python3.${PYMINOR}")

# libyaml headers/lib come from the install prefix (libyaml_vendor installed them).
set(PREFIX "${CMAKE_INSTALL_PREFIX}")

include(ExternalProject)

externalproject_add(pyyaml_external
  GIT_REPOSITORY https://github.com/yaml/pyyaml.git
  GIT_TAG ${PYYAML_VERSION}
  GIT_CONFIG advice.detachedHead=false
  UPDATE_COMMAND ""
  TIMEOUT 600
  CONFIGURE_COMMAND ""
  BUILD_IN_SOURCE TRUE
  # Cross-build the C extension against libyaml (ported from pkg/pyyaml).
  BUILD_COMMAND ${CMAKE_COMMAND} -E env
      PYTHON=${PYTAG}
      CC=wr-cc
      "CFLAGS=-I ${SYSROOT}/usr/3pp/develop/usr/include/${PYTAG} -I ${PREFIX}/include/libyaml_vendor"
      "LDSHARED=wr-cc -shared"
      "LDFLAGS=-L ${SYSROOT}/usr/3pp/develop/usr/lib -lpython3 -L ${PREFIX}/lib -lyaml"
      _PYTHON_HOST_PLATFORM=vxworks-x86_64
      _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__vxworks_vxworks
      "PYTHONPATH=${SYSROOT}/usr/3pp/develop/usr/lib/${PYTAG}/:${SYSROOT}/usr/3pp/develop/usr/lib/${PYTAG}/site-packages"
      ${PYTAG} setup.py --with-libyaml build
  # Install the built lib/ tree into the target site-packages.
  INSTALL_COMMAND ${CMAKE_COMMAND} -E make_directory ${PREFIX}/lib/${PYTAG}/site-packages
    COMMAND ${CMAKE_COMMAND} -E copy_directory
      <SOURCE_DIR>/lib ${PREFIX}/lib/${PYTAG}/site-packages
)

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()
endif()

ament_package()
