cmake_minimum_required(VERSION 3.7)

project(pyyaml_vendor)

find_package(ament_cmake REQUIRED)

# PyYAML (YAML Python module) with its libyaml C extension (_yaml), cross-compiled
# for the target. Builds under colcon as a *_vendor; libyaml comes from ros2's
# libyaml_vendor (see package.xml depend). Pinned to 6.0.3 to match Ubuntu 26.04.
#
# The CPython cross-build needs an OS-specific _PYTHON_HOST_PLATFORM +
# _PYTHON_SYSCONFIGDATA_NAME. Those are selected per target OS via explicit
# branches on CMAKE_SYSTEM_NAME (set by the colcon toolchain file) — add an
# elseif() for a new OS (QNX, ...). The arch and compiler are taken from the
# toolchain, never hardcoded.

set(PYYAML_VERSION 6.0.3)

# --- cross compiler from the toolchain (not hardcoded wr-cc) ---
if(CMAKE_C_COMPILER)
  set(CROSS_CC "${CMAKE_C_COMPILER}")
elseif(DEFINED ENV{CC})
  set(CROSS_CC "$ENV{CC}")
else()
  message(FATAL_ERROR "pyyaml_vendor: no cross compiler (CMAKE_C_COMPILER / CC)")
endif()

# --- target arch from the compiler triple (arm64 -> aarch64) ---
set(TGT_ARCH "$ENV{TGT_ARCH}")
if(NOT TGT_ARCH)
  execute_process(
    COMMAND ${CROSS_CC} -print-target-triple -c ${CMAKE_CURRENT_SOURCE_DIR}/package.xml
    OUTPUT_VARIABLE _triple OUTPUT_STRIP_TRAILING_WHITESPACE ERROR_QUIET)
  string(REGEX REPLACE "-.*$" "" TGT_ARCH "${_triple}")
  string(REPLACE "arm64" "aarch64" TGT_ARCH "${TGT_ARCH}")
endif()

# --- per-OS CPython cross settings (branch on the toolchain's system name) ---
if(CMAKE_SYSTEM_NAME STREQUAL "VxWorks")
  set(PY_HOST_PLATFORM "vxworks-${TGT_ARCH}")
  set(PY_SYSCONFIG     "_sysconfigdata__vxworks_vxworks")
elseif(CMAKE_SYSTEM_NAME STREQUAL "QNX")
  set(PY_HOST_PLATFORM "qnx-${TGT_ARCH}")
  set(PY_SYSCONFIG     "_sysconfigdata__qnx_qnx")
else()
  message(FATAL_ERROR
    "pyyaml_vendor: unsupported target OS '${CMAKE_SYSTEM_NAME}'. "
    "Add an elseif() branch with its _PYTHON_HOST_PLATFORM + sysconfigdata name.")
endif()

# --- sysroot / python minor from the harness env ---
set(SYSROOT "$ENV{WIND_CC_SYSROOT}")
if(NOT SYSROOT)
  message(FATAL_ERROR "pyyaml_vendor: WIND_CC_SYSROOT not set — build via the SDK env")
endif()
if(DEFINED ENV{TGT_PYTHON_MINOR})
  set(PYMIN "$ENV{TGT_PYTHON_MINOR}")
else()
  set(PYMIN "9")
endif()
set(PYTAG "python3.${PYMIN}")
set(PREFIX "${CMAKE_INSTALL_PREFIX}")
set(DEV "${SYSROOT}/usr/3pp/develop/usr")

message(STATUS "pyyaml_vendor: os=${CMAKE_SYSTEM_NAME} arch=${TGT_ARCH} "
               "host_platform=${PY_HOST_PLATFORM} sysconfig=${PY_SYSCONFIG}")

include(ExternalProject)

externalproject_add(pyyaml_external
  GIT_REPOSITORY https://github.com/yaml/pyyaml.git
  GIT_TAG ${PYYAML_VERSION}
  GIT_CONFIG advice.detachedHead=false
  UPDATE_COMMAND ""
  TIMEOUT 600
  CONFIGURE_COMMAND ""
  BUILD_IN_SOURCE TRUE
  BUILD_COMMAND ${CMAKE_COMMAND} -E env
      PYTHON=${PYTAG}
      "CC=${CROSS_CC}"
      "LDSHARED=${CROSS_CC} -shared"
      "CFLAGS=-I ${DEV}/include/${PYTAG} -I ${PREFIX}/include/libyaml_vendor"
      "LDFLAGS=-L ${DEV}/lib -lpython3 -L ${PREFIX}/lib -lyaml"
      _PYTHON_HOST_PLATFORM=${PY_HOST_PLATFORM}
      _PYTHON_SYSCONFIGDATA_NAME=${PY_SYSCONFIG}
      "PYTHONPATH=${DEV}/lib/${PYTAG}/:${DEV}/lib/${PYTAG}/site-packages"
      ${PYTAG} setup.py --with-libyaml bdist
  INSTALL_COMMAND ${CMAKE_COMMAND} -E make_directory ${PREFIX}/lib/${PYTAG}/site-packages
    COMMAND bash -c "cp -r <SOURCE_DIR>/lib/* ${PREFIX}/lib/${PYTAG}/site-packages/ && \
                     cp -r <SOURCE_DIR>/build/lib.${PY_HOST_PLATFORM}-${PYMIN}/yaml/*.so \
                           ${PREFIX}/lib/${PYTAG}/site-packages/yaml/ 2>/dev/null || true"
)

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()
endif()

ament_package()
