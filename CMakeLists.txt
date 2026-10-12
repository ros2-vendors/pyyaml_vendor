cmake_minimum_required(VERSION 3.7)

project(pyyaml_vendor)

find_package(ament_cmake REQUIRED)

# PyYAML (YAML Python module) with its libyaml C extension (_yaml), cross-compiled
# for VxWorks. This reproduces the exact recipe that the former pkg/pyyaml make
# package used (verified: builds yaml/_yaml.cpython-39-vxworks.so), driven here so
# it builds under colcon as a *_vendor. libyaml comes from ros2's libyaml_vendor
# (see package.xml depend). Pinned to 6.0.3 to match Ubuntu 26.04 python3-yaml.

set(PYYAML_VERSION 6.0.3)

# Provided by the harness env (sdkenv.sh + top-level Makefile exports).
set(SYSROOT "$ENV{WIND_CC_SYSROOT}")
if(NOT SYSROOT)
  message(FATAL_ERROR "pyyaml_vendor: WIND_CC_SYSROOT not set — source sdkenv.sh")
endif()
if(DEFINED ENV{TGT_PYTHON_MINOR})
  set(PYMIN "$ENV{TGT_PYTHON_MINOR}")
else()
  set(PYMIN "9")
endif()
set(PYTAG "python3.${PYMIN}")
set(PREFIX "${CMAKE_INSTALL_PREFIX}")

# The SDK "develop" tree holds the cross CPython headers + the
# _sysconfigdata__vxworks_vxworks module that makes setup.py cross-target VxWorks.
set(DEV "${SYSROOT}/usr/3pp/develop/usr")

include(ExternalProject)

externalproject_add(pyyaml_external
  GIT_REPOSITORY https://github.com/yaml/pyyaml.git
  GIT_TAG ${PYYAML_VERSION}
  GIT_CONFIG advice.detachedHead=false
  UPDATE_COMMAND ""
  TIMEOUT 600
  CONFIGURE_COMMAND ""
  BUILD_IN_SOURCE TRUE
  # Exact cross-Python env from pkg/pyyaml (CC=wr-cc, VxWorks host platform +
  # sysconfigdata, libyaml from the install prefix), then `setup.py --with-libyaml bdist`.
  BUILD_COMMAND ${CMAKE_COMMAND} -E env
      PYTHON=${PYTAG}
      CC=wr-cc
      "LDSHARED=wr-cc -shared"
      "CFLAGS=-I ${DEV}/include/${PYTAG} -I ${PREFIX}/include/libyaml_vendor"
      "LDFLAGS=-L ${DEV}/lib -lpython3 -L ${PREFIX}/lib -lyaml"
      _PYTHON_HOST_PLATFORM=vxworks-x86_64
      _PYTHON_SYSCONFIGDATA_NAME=_sysconfigdata__vxworks_vxworks
      "PYTHONPATH=${DEV}/lib/${PYTAG}/:${DEV}/lib/${PYTAG}/site-packages"
      ${PYTAG} setup.py --with-libyaml bdist
  # Install the built pure-python + extension tree into the target site-packages.
  INSTALL_COMMAND ${CMAKE_COMMAND} -E make_directory ${PREFIX}/lib/${PYTAG}/site-packages
    COMMAND bash -c "cp -r <SOURCE_DIR>/lib/* ${PREFIX}/lib/${PYTAG}/site-packages/ && \
                     cp -r <SOURCE_DIR>/build/lib.vxworks-x86_64-${PYMIN}/yaml/*.so \
                           ${PREFIX}/lib/${PYTAG}/site-packages/yaml/ 2>/dev/null || true"
)

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()
endif()

ament_package()
